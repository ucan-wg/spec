# CAR Pool

CAR Pool is a collection of specs, which taken together aim to provide a network efficient method for transmitting data from one or more sources to a single sink.

## Specs

### CAR Mirror

* [High Level Spec](./car-mirror/SPEC.md)
* [CAR Mirror over HTTP](./car-mirror/http.md)
  
### CAR Pool

* [Spec](./car-pool/README.md) (future)

## Appendices

* [Research](./RESEARCH.md)

## Abstract

We propose developing a transport agnostic CAR-based synchronization mechanism, with the primary purpose being efficient & reliable block delivery over HTTP.

## Motivation & High-Level Description

This proposal resolves multiple major, persistent challenges that Fission has experienced as a production IPFS operator for the past several years. From discussion with others in the IPFS Operators Group, we believe the scope of this work to be an important stepping stone in the maturation of IPFS and Filecoin in production settings.

We propose extending the existing efforts to move specifically CAR files over HTTP (e.g. Filecoin point-to-point CAR, Qri’s DSync), and enable other transports such as WSS, HLS, and WebRTC. We know of several other projects in the broader ecosystem that would like to make use of this protocol, from storage providers and CDNs to decentralized social media protocols.

IPLD (and content addressing broadly) has an incredible advantage over conventional RESTful transfers: IPLD can easily deduplicate data. We strongly desire to retain this property. At Fission, our in-house file system (WNFS) is immutable-by-default. Uploading many redundant gigabytes for a small change is not practical.

### Round Trip Reduction

Bitswap is very efficient for deduplication, but suffers when synchronizing large IPLD graphs due to the large number of round trips involved. This scales specifically with the depth of the diff in the requested graph. Discovering that a node is not available only occurs after many round trips.

Further, due to the Want List behavior being session agnostic, we have observed uploads and downloads stall mid-transfer when a peer is considered idle (or when many new peers connect), evicting the node from the peer list. With CAR Mirror, in both streaming and at-once delivery cases, it is well understood if the session is still open.

### HTTPS & WSS Are Mature Transports

Streaming CAR files know that there is an open connection over a reliable, mature transport: HTTP, Streaming HTTP, and persistent WSS. Being able to efficiently push and fetch IPLD in CAR files over HTTP in particular means gaining the ability to move data over a mature, absolutely ubiquitous transport on privileged ports.

Relying on HTTP also makes TLS available everywhere, improving the security and privacy of messages. Since the proposed protocol does not depend on the public DHT (see Scope section), this provides some improvement even with public providers. The lack of dependence on the public DHT is well suited to short-lived nodes, such as browsers, GitHub Actions, and battery-sensitive devices such as mobile phones.

It is worth highlighting that HTTP is unable to support many P2P use cases, and many client devices are not directly dialable as data providers. Those use cases will need continued dependence on WebRTC and similar. These strategies can run entirely in parallel, and are in no way mutually exclusive. A node being undialable leads to some difficulty in the design of this system, but even under these conditions, CAR protocols can be constructed that perform better than the naive case.

### Deduplication

The primary challenges are in efficient remote graph selection. Avoiding sending redundant bytes over the wire is a tradeoff which Bitswap is on one end of. There is no perfect information-theoretic solution, so designing the alternate protocol is largely an exercise in optimization.

Fission has a strong real-world use case for deduplication: the [WebNative File System](https://github.com/WebNativeFileSystem/) is both eventually consistent and persistent-by-default (history is retained on update, like in Apple’s Time Machine). Thanks to the large amount of structural sharing, most of this structure is unchanged when performing a synchronization.

Deduplication in this scenario is now nontrivial. When pushing or fetching data, a node needs to know what is available locally and remotely, create a diff, package that up and send it. We propose using a Bloom filter to capture a snapshot of the local state relative to some CID, and send it to the remote. We then have a picture of what is different in its copy below that root. As we can only squeeze so much information into a message, this is then done in rounds/epochs if the request is not fulfilled in the first one.

### Parallel Streams

Streaming data in true parallel from multiple sources is a very attractive feature of Bitswap. We aim to retain this property in CAR Pool with (likely) a combination of rendezvous hashing and intra-cluster communication of inverted (subtractive) counting bloom filters forming essentially a “session CRDT'' of provided CIDs. This is done both with single blooms, and across multiple epochs (which may happen eagerly in parallel).

## Scope

Our strategy is to build as much on top of existing libp2p technologies as possible, extended with three things:

* A CAR-based HTTP API for arbitrary DAGs in Kubo (go-ipfs)
* Store reconciliation (remote differential graph selection)
* Parallel streaming from multiple cooperative peers

Note that this does not include general DHT discovery or connection. Further work may involve peer discovery with ad hoc inclusion to the CAR Pool cluster, but would require additional protocols. Our strategy is to ship an immediately impactful protocol into Kubo, which the scope here manages the 90th percentile use case that we have seen in practice. We are happy to scope out the broader DHT provider upon request.
