# CAR Mirror

> Objects in the CAR Mirror are closer than they appear
>
> -- Quinn Wilton

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# 0. Abstract

CAR Mirror describes a method for efficiently diffing, deduplicating, packaging, and transmitting IPLD data from a source node. The two primary advantages of CAR Mirror are the reduction in network round trips versus [Bitswap](https://docs.ipfs.io/concepts/bitswap/), and probabilistically excluding redundant blocks from being transferred. The protocol aims to be easy to implement for clients, whether or not they have a complete [libp2p](https://libp2p.io/) network stack.

## 0.1 Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1. Introduction

## 1.1 Motivation 

IPLD data is structured as a Merkle tree, which closes authentication over some rooted structure, but gives no help in determining what might be contained in it without actually walking this tree. Performing a pull request against a remote with a deep tree requires a large number of round trips as children are discovered at each level. Not packaging up the entire structure into one request is motivated by deduplication: the recipient may already have many of the blocks, and transferring them would be redundant. This situation creates an apparent zero sum optimization problem between deduplication and round trips.

## 1.2 Relevant Insights

The motivating insights are:

1. The Requestor often has information about the structure and content of the data it's requesting (e.g. an update to a directory)
2. Rounds and deduplication are not mutually exclusive; they're not even directly correlated! To put this another way: it's possible to sacrifice some deduplication accuracy to get a large reduction in the number of rounds.
3. In a multiple round protocol, approximate set reconciliation can be iteratively improved as more information is shared.

It would be impractical (and indeed privacy violating) for the provider to maintain a list of all CIDs held by the requestor. Sending a well tuned [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) is size efficient, has a very fast membership check, and doesn't require announcing every block that it has.

IPLD provides opportunities for optimizing both latency and bandwidth. The goal of CAR Mirror is to merely do better than either pure Bitswap or pure uploads. There are steps that work with imperfect knowledge, but make a bet that the data will be useful to avoid a negotiation round ahead of sending blocks. CAR Mirror aims to keep the number of rounds low, while reducing the number of duplicated blocks.

The worst case scenario for CAR Mirror efficiency is exactly [DSync](https://pkg.go.dev/github.com/qri-io/dag/dsync). This is already an improvement over Bitswap, but narrowing the sync space has the potential to continually improve efficiency at the limit.

## 1.3 Constraints

> The limitation of local knowledge is the fundamental fact about the setting in which we work, and is a very powerful limitation
>
> -- Nancy Lynch, [A Hundred Impossibility Proofs for Distributed Computing](https://groups.csail.mit.edu/tds/papers/Lynch/podc89.pdf)

CAR Mirror makes few assumptions about network topology, prior history, availability, or transport. This protocol makes a cold start assumption. While CAR Mirror can do better with more state, it is stateless by default: peers assume that any information that they have about each other from previous interactions is out of date and heuristic at best. CAR Mirror is unicast and unidirectional (source-to-sink).

The Bloom filter introduces new overhead not present in Bitswap. This cost is amortized over the typical reduction in round trips, though for very simple cases (such as sending a single block), it is possible that this overhead is not paid back. The size of the Bloom filter is kept to a minimum, and tuned on heuristic factors. CAR Mirror has the greatest gains on deep trees, and very shallow trees generally work well in Bitswap.

CAR Mirror is not a discovery protocol; it assumes that the provider has (at least some) of the required data. The efficiency gains are purely in reducing the number of round trips; it is completely agnostic about the existence of a DHT.

Transmitting complete information about very large block stores is not practical. Further, a Provider will typically have a much larger number of blocks than a Requestor. Narrowing the problem space down is required in the majority of cases, which can take advantage of any number of heuristics. As such, finding ways to narrow the problem space is critical.

# 2. Concepts

## 2.1 Roles

|           | Sender            | Receiver          | Dialable |
|-----------|-------------------|-------------------|----------|
| Requestor | Push Upload       | Pull Download     | Maybe    |
| Provider  | Fulfill Retrieval | Fulfill Storage   | Yes      |

### 2.1.1 Requestor

The Requestor is the node that starts the session. This will be either the sender or receiver, depending on the direction of data flow.

### 2.1.2 Provider

The Provider is the node that will fulfill the data request, either by providing storage or supplying the requested blocks.

### 2.1.3 Sender

The Sender is the node that has the data, and is pushing it to the other node.

### 2.1.4 Receiver

The Receiver is the node that is pulling data from the other node.

### 2.1.5 Dialable

Whether a peer can initiate contact with a particular node. For example, a web browser is not typically directly dialable.

## 2.2 CAR

[Content Addressable aRchive (CAR)](https://ipld.io/specs/transport/car/) is a format similar to the [TAR file](https://en.wikipedia.org/wiki/Tar_(computing)) modified to work efficiently on IPLD data. This specification uses [CARv1](https://ipld.io/specs/transport/car/carv1/).

## 2.2.1 Streaming CAR

A CAR file can be expressed as a streaming data structure, and transmitted over [HTTP Live Streaming](https://datatracker.ietf.org/doc/html/rfc8216), [WebSockets](https://www.rfc-editor.org/rfc/rfc6455.html) or similar.

## 2.3 Session

A session is made up of several rounds. Each round involves the Requestor contacting the Provider. Both parties MAY maintain stateful information between rounds as part of the session. The purpose of this session information is to aid in deduplication, completion, and cleanup.

### 2.3.1 Deduplication

A benefit of content addressing is deduplication. CAR Mirror deduplicates blocks across both parties to avoid putting redundant data over the wire. To achieve deduplication, the Requestor shares a Bloom filter with the Provider containing an estimate of blocks that are possibly shared. This is an imprecise measure, though it does significantly better than sending duplicate blocks on the average case.

### 2.3.2 Stragglers

Stragglers are the opposite of deduplicated blocks: they are missing. This is typically caused by false positives on the Bloom filters. As such, this protocol is biased towards stragglers.

Each round of communication will have fewer stragglers. There is a degenerate case where the Provider only matches on the roots, but omits everything from the Bloom filter. This situation is roughly equivalent to Bitswap, but with the overhead of Bloom filters. In such cases, the Requestor MAY omit the Bloom filter and request specific CIDs only. This is called the "cleanup" phase. It is possible that after a few rounds, this process becomes "unstuck" and the Bloom becomes useful again.

# 3. Transport

The protocol here is described in discrete rounds. When run over fully bidirectional transports such as WSS, portions of rounds may be updated live rather than waiting for an arbitrary "round" to complete. Unidirectional transports like HTTP proceed in clear, distinct rounds. Discrete rounds are given below both because many transports require this mode, and because it is easier to explain in structured steps, even if not a hard requirement.

## 3.1 Phases

```plaintext
  ┌─────────────┐
  │             │
  │  Selection  │
  │             │
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │             │
  │  Narrowing  │◄───────┐
  │             │        │
  └──────┬──────┘        │
         │               │
         ▼               │
┌────────────────┐       │
│                │       │
│  Transmission  │       │ Next Round
│                │       │
└────────┬───────┘       │
         │               │
         ▼               │
┌────────────────┐       │
│                │       │
│  Graph Status  ├───────┘
│                │
└────────┬───────┘
         │
         ▼
   ┌───────────┐
   │           │
   │  Cleanup  │
   │           │
   └───────────┘
```

### 3.1.1 Selection

The first step is always deciding which graph to mirror, referenced by its root or roots.

### 3.1.2 Narrowing

Narrowing involves various heuristics to narrow the sync space to minimize the number of duplicate blocks sent. The goal is to reduce the amount of redundant bandwidth and number of communication rounds.

It is not possible to do a perfect job of narrowing asynchronously. Even if both parties operate with each other in synchronous rounds, they make no claims about what is being sent from other peers, garbage collection, corruptions, or cold starts.

The worst-case for narrowing is the inability to narrow the problem at all. In this case, the only information shared between the peers is the CID to be synced. There are 3 possible strategies for this scenario:

1. Maximal rounds, low duplication risk. Essentially Bitswap.
2. One round, high duplication risk. Essentially DSync.
3. Starting with a "cold call" (transfer initial round) and see if this transfer reveals more information.

By analogy, the third option above is a little like playing Sudoku or Minesweeper. Each round has the potential to reveal more information about the other peer's state, improving the ability to make smarter choices about what to put on the wire.

### 3.1.3 Transmission

The actual transfer of blocks via a CARv1. This may be streaming, or a discrete file.

#### 3.1.3.1 Cold Calls

This protocol opts for kick starting worst-case narrowing with a "cold call" round. This keeps the transferred blocks as shallow in the graph as possible to avoid sending a large number of duplicate blocks. Once this round is complete, the other party has an opportunity to respond with information relevant to the set reconciliation.

### 3.1.4 Graph Status

Periodically -- at the end of the file or every $n$ streamed blocks -- the Receiver sends a list of unsynchronized subgraph roots: the bottom of the currently synced graph with unsynchronized children. The Provider also sends a Bloom of blocks that it thinks may be discovered as duplicates.

### 3.1.5 Cleanup

The cleanup phase falls back to a slower but more accurate strategy. In this phase, any blocks that were missed due to Bloom false positives are found, explicitly enumerated, and transmitted up. The idea is that there will only be a handful (if any) stragglers to clean up, and a low chance that they will be deeply nested. At this stage, the peers MAY switch to Bitswap to fulfill these requirements.

## 3.2 Pull Protocol 📥

### 3.2.1 High Level

For each round, the Requestor OPTIONALLY creates a Bloom filter with all leaf and interior CIDs for the nodes suspected to be shared with the structure that it is pulling. It sends this Bloom and the top CIDs to the Provider.

In turn, the Provider initializes a fresh CAR file, which MAY be streaming. The Provider maintains a session CID memoization table (such as a hash set). It performs graph traversal on its local store, starting from the first requested CID. Absent any other information, preorder traversal is RECOMMENDED. However, an implementation MAY choose another strategy, for instance if it knows that certain blocks will be preferred first.

Any available graph roots MUST be sent as an array (`[CID]`) by the Requestor, regardless of the Requestor's Bloom filter. These form the basis of subquery anchors in the session.

Every CID visited is added to the Provider's internal memoization table. Each Provider block is matched against both the Requestor Bloom filter and the Provider's session cache. If a block is found in either structure, it is RECOMMENDED that the Provider exclude it from the payload, and stop walking that path. Once the relevant graph has been exhausted, the Provider closes the channel.

On receipt of each block, the Requestor adds the block to its local store, and to its copy of the Bloom. If the Bloom crosses a saturation point, the Requestor MUST resize it before the next round.

At the end of each round, the Requestor MUST inspect its graph for any missing subgraphs. If there are incomplete subgraphs, the Requestor begins again with the roots of the missing subgraphs. If the Provider fails to send a root, the Requester marks it as unavailable from that Provider. The Requestor excludes unavailable roots from future rounds in the session.

If the Provider begins returning only the requested roots but no other blocks, the Requestor MAY initiate a cleanup round. A cleanup phase merely omits the Bloom filter from the request. It is akin to a point-to-point Bitswap Want List, and performs best when there is a large number of subgraph roots with the expectation of no child nodes.

The Requestor MAY garbage collect its session state as soon as it has all of the blocks for the structure, minus the subgraphs marked as unavailable. It is RECOMMENDED that it keep this data indexed by the `peerId` of the Provider and root CID from the request, so that it can use this information in future requests.

The Provider MAY garbage collect its session state when it has exhausted its graph, since false positives in the Bloom filter MAY lead to the Provider having an incorrect picture of the Requestor's store, and further requests MAY come in for that session. Session state is an optimization, so treating this as a totally new session is acceptable. However, due to this fact, it is RECOMMENDED that the Provider maintain a session state TTL of at least 30 seconds since the last block is sent. Maintaining this cache for long periods can speed up future requests, so the Provider MAY keep this information around to aid future requests.

## 3.2.2 Individual Round Sequence Diagram

```plaintext
┌─────────────────────┐                                                ┌────────────────────┐
│                     │                                                │                    │
│   Previous Round    │                                                │   Previous Round   │
│  Bloom & CID Roots  │                                                │      Hash Set      │
│                     │                                                │                    │
└───┬────┬────────────┘                                                └────┬──────────┬────┘
    │    │                                                                  │          │
    │    │                                                                  │          │
    │    │                                                                  │          │
    │    │            Requestor                          Provider           │          │
    │    │                │                                  │              │          │
    │    │                │                                  │              │          │
    │   Find New          │                                  │              │          │
    │   Subgraph Roots    │                                  │              │          │
    │   & Update Bloom    │                                  │              │          │
    │    │                │                                  │              │          │
    │    │                │                                  │              │          │
    │    │                │        (Bloom, CID Roots)        │              │          │
    │    └───────────────►├─────────────────────────────────►├────────┐     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │   Walk Local │          │
    │                     │                                  │      Graph   │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        ▼     ▼          │
    │                     │                                  │     ┌────────────────┐  │
    │                     │                                  │     │                │  │
    │                     │                                  │     │  Response CAR  │  │
    │                     │                                  │     │                │  │
    │                     │                                  │     └──┬─────┬───────┘  │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │                                  │        │     │          │
    │                     │       CAR { CID => Block }       │        │     │          │
    │         ┌───────────┤◄─────────────────────────────────┤ ◄──────┘     │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    │      Update         │                                  │              │          │
    │    Local Store      │                                  │              │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    │         │           │                                  │              │          │
    ▼         ▼           │                                  │              ▼          ▼
┌─────────────────────┐   │                                  │         ┌────────────────────┐
│                     │   │                                  │         │                    │
│      Updated        │   │                                  │         │       Updated      │
│  Bloom & CID Roots  │   │                                  │         │      Hash Set      │
│                     │   │                                  │         │                    │
└─────────────────────┘   │                                  │         └────────────────────┘
```

## 3.3 Push Protocol 📤

### 3.3.1 High Level

The push protocol is very similar to pull (§3.2), roughly inverted. The major difference aside from direction of data flow is that the first round MAY be a "cold call". This prioritizes starting the first round with zero information about the Provider, under the assumption that the first branch of the graph will either not contain many duplicate blocks, or if it does that it is doing as well as the prior art CAR transfers. This strategy prioritizes limiting round trips over reducing bandwidth.

The sending Requestor begins with a local phase estimating what the Provider has in its store. This may be from stateful information (e.g. Bloom filters and CIDs) learned in a previous round, or by using a heuristic such as knowing that the provider has a previous copy associated with an IPNS record or DNSLink. If no information is available, the estimate is the empty set.

The Requestor performs graph traversal of the data under the CID to push, appending blocks to a CAR file. This CAR MAY be discrete or streaming, depending on the transport. All other factors being equal, breadth-first traversal is RECOMMENDED. Since the Provider is not expected to use the data immediately, it exposes the largest number of roots, and thus grants the highest chance of discovering shared subgraphs. An implementation MAY choose a different traversal strategy, for example if it knows more about the use.

At the end of each round, the receiving Provider MUST respond with a Bloom filter of likely future duplicates, and an array of CID roots for pending subgraphs. Either MAY be empty, in which case it is treated merely as an `ACK`.

On the next round, the Requestor checks each block against the filter, and begins appending them to the CAR and continuing on as normal.

## 3.3.2 Individual Round Sequence Diagram

```plaintext
┌─────────────────────┐
│                     │                                     ┌─────────────────────┐
│   Previous Round    │                                     │                     │
│  Bloom, Hash Set,   │                                     │   Previous Round    │
│     & CID Roots     │                                     │  Bloom & CID Roots  │
│                     │                                     │                     │
└───┬─────────┬───────┘                                     └──────────┬──────────┘
    │         │                                                        │
    │         │                                                        │
    │         │                                                        │
    │         │       Requestor                   Provider             │
    │         │           │                           │                │
    │         │           │                           │                │
    │     Walk local      │                           │                │
    │       graph         │                           │                │
    │         │           │                           │                │
    │         │           │                           │                │
    │         │           │                           │                │
    │ ┌───────┴───────┐   │                           │                │
    │ │               │   │                           │                │
    │ │  Request CAR  │   │                           │                │
    │ │   DAG Bloom   │   │                           │                │
    │ │               │   │                           │                │
    │ └──┬─────┬──────┘   │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │     │          │   CAR { CID => Block }    │                │
    │    │     │          │            and            │                │
    │    │     │          │   remaining graph Bloom   │                │
    │    │     └─────────►├──────────────────────────►├─────────┐      │
    │    │                │                           │         │      │
    │    │                │                           │         │      │
    │    │                │                           │         │      │
    │    │                │                           │    Add Blocks  │
    │    │                │                           │    and compare │
    │    │                │                           │   Bloom filter │
    │    │                │                           │         │      │
    │    │                │                           │         │      │
    │    │                │                           │         ▼      ▼
    │    │                │                           │      ┌─────────────┐
    │    │                │                           │      │             │
    │    │                │                           │      │   Bloom &   │
    │    │                │                           │      │  CID Roots  │
    │    │                │                           │      │             │
    │    │                │                           │      └──┬──────┬───┘
    │    │                │                           │         │      │
    │    │                │                           │         │      │
    │    │                │                           │         │      │
    │    │                │    (Bloom, CID Roots)     │         │      │
    │    │     ┌──────────┤◄──────────────────────────┼─◄───────┘      │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │   Update       │                           │                │
    │    │ Local Cache    │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    │    │     │          │                           │                │
    ▼    ▼     ▼          │                           │                ▼
┌─────────────────────┐   │                           │        ┌───────────────┐
│                     │   │                           │        │               │
│      Updated        │   │                           │        │    Updated    │
│  Bloom & CID Roots  │   │                           │        │   Hash Set    │
│                     │   │                           │        │               │
└─────────────────────┘   │                           │        └───────────────┘
```

## 3.4 Bloom Optimization

The parameters for the Bloom are set by the Requestor. Many of the parameters are self-evident from the filter itself, but the number of hashes must be passed along in the initial request.

Optimizing Bloom filters depends on balancing false positive probability (FPP or $\epsilon$), filter size, number of hashes, hash function, and expected number of elements. This Bloom MUST use the deterministic hash function [XXH3](https://cyan4973.github.io/xxHash/). 

It is RECOMMENDED to make the FPP one order of magnitude (OOM) under the inverse of the order of magnitude of the number of inserted elements. For instance, if there are some 100ks of elements in the filter, then the FPP should be $1/1M$. This can grow quickly, so an implementation MAY use another order of magnitude, such as the inverse of the OOM of the number of inserted elements.

The core idea of using a Bloom filter is that it is very fast and space efficient. For example, a Bloom filter with 100k elements and a FPP of $10^{-6}$ can be expressed in a little over 350KB.

### 3.4.1 Equations

Legend
* $n$: number of elements in the filter
* $m$: size of filter in bits
* $\epsilon$: false positive probability
* $k$: number of hash functions

Some [optimality equations](https://en.wikipedia.org/wiki/Bloom_filter#Optimal_number_of_hash_functions):
* $k = {m \over n} \ln{2}$
* $m = {n \ln{\epsilon} \over {(\ln{2})^{2}}}$

A typical example looks like this:

* $n = 500\text{k}$
* $m \approx 1.8\text{MB}$
* $\epsilon \approx 1 \times 10^{-6}$
* $k = 20$

## 3.5 Similarity

As the entire data store of an arbitrary Requestor may be unbounded in size, the Bloom filter is populated with blocks that are likely to match based on semantic criteria or other heuristics.

The primary challenge in determining what information to send is that this protocol deals directly with the fact that knowledge in a distributed system is local.

### 3.5.1 Small Store

If a local store is on the order of 100,000s of blocks, then the entire store can reasonably be sent in the Bloom filter.

### 3.5.2 Mutable Pointer

A simple heuristic is on data updated from a mutable pointer, such as on [IPNS](https://docs.ipfs.io/concepts/ipns/) or [DNSLink](https://dnslink.io). This often shares some structure, and thus the stale graph MAY be placed into the Bloom filter.

# 4. Caches

It is RECOMMENDED that CAR Mirror-enabled nodes maintain local caches of peer states, metadata about their own block stores, and memoization tables.

# 5. FAQ

## 5.1 Why use a classic Bloom filter?

There are many variants on Bloom filters that are used for similar problems in the literature, including the Invertible Bloom Filter, Distributed Bloom Filter, and HEX-BLOOM, among others. These all trade off size for the ability to synchronize multiple clients. CAR Mirror is a unicast source-to-sink protocol, thus the additional features of these structures are made redundant. 

## 5.2 Privacy

There exist protocols for zero-knowledge set intersection (PSI) and union (PSU). In fact, some of these even use Bloom filters! Typically these protocols have a tradeoff on number of rounds, or at very least amount of system complexity. Since CAR Mirror is focused on transfer of data and not in the intersection of two sets, this is of less value than in use cases such as contacts matching, privacy-preserving routing, and so on. In fact, we expect to explore this as part of CAR Pool to extend the federated model to an open protocol.

# 6. Acknowledgments

Many thanks to [Philipp Krüger](https://github.com/matheus23) for the research, investigation, and experimental verification of Bloom filter parameters and variants.

A big thank you to [Justin Johnson](https://github.com/justincjohnson) for feedback on this spec, many suggestions on improving the readability, and writing the Golang implementation.

Thank you to [Quinn Wilton](https://github.com/QuinnWilton) for late night discussions on graph synchronization, error correction, Tornado codes, and alternate designs.

Many thanks to [James Walker](https://github.com/walkah/) for feedback and editing on the initial project proposal.

Thanks to [Brendan O'Brien](https://github.com/b5) for the general encouragement for this scope of work, and for the prior art on [Qri's DSync](https://pkg.go.dev/github.com/qri-io/dag/dsync).
