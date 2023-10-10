# User Controlled Authorization Network (UCAN) Specification v1.0.0-rc.1

## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Brooklyn Zelenka], [Fission]
* [Daniel Holmgren], [Bluesky]
* [Irakli Gozalishvili], [Protocol Labs]
* [Philipp Krüger], [Fission]

## Sub-Specifications

* [UCAN Invocation]
* [UCAN Revocation]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 0. Abstract

User-Controlled Authorization Network (UCAN) is a [trustless], secure, [local-first], user-originated, distributed authorization scheme. This document provides a high level overview of the components of the system, concepts, and motivation. Exact formats are given in [sub-specifications].

# 1. Introduction

User-Controlled Authorization Network (UCAN) is a [trustless], secure, [local-first], user-originated , distributed authorization scheme. It provides public-key verifiable, delegable, expressive, openly extensible [capabilities] by extending the familiar [JWT] structure. UCANs achieve public verifiability with late-bound certificate chains and principals represented by [decentralized identifiers (DIDs)][DID].

Being encoded with the familiar JWT, UCAN improves the familiarity and adoptability of schemes like [SPKI/SDSI][SPKI] for web and native application contexts. UCAN allows for the creation, delegation, and invocation of authority by any agent with a DID, including traditional systems and peer-to-peer architectures beyond traditional cloud computing.

## 1.1 Motivation

Since at least [Multics], access control lists ([ACL]s) have been the most popular form of digital authorization, where a list of what each user is allowed to do is maintained on the resource. ACLs have been a successful model suited to architectures where persistent access to a single list is viable. ACLs require that rules are sufficiently well specified, such as in a centralized database with rules covering all possible permutations of scenario. This both imposes a very high maintenance burden on programmers as a systems grows in complexity, and is a key vector for [confused deputies][confused deputy problem]. 

With increasing interconnectivity between machines becoming commonplace, authorization needs to scale to meet the load demands of distributed systems while providing partition tolerance. However, it is not always practical to maintain a single central authorization source. Even when copies of the authorization list are distributed to the relevant servers, latency and partitions introduce troublesome challenges with conflicting updates, to say nothing of storage requirements.

A large portion of personal information now also moves through connected systems. As a result, data privacy is a prominent theme when considering the design of modern applications, to the point of being legislated in parts of the world. 

Ahead-of-time coordination is often a barrier to development in many projects. Flexibility to define specialized authorization semantics for resources and the ability to integrate with external systems trustlessly are essential as the number of autonomous, specialized, and coordinating applications increases.

Many high-value applications run in hostile environments. In recognition of this, many vendors now include public key functionality, such as [non-extractable keys in browsers][browser api crypto key], [certificate systems for external keys][fido], [platform keys][passkey], and [secure hardware enclaves] in widespread consumer devices.

Two related models that work exceptionally well in the above context are Simple Public Key Infrastructure ([SPKI][spki rfc]) and object capabilities ([OCAP]). Since offline operation and self-verifiability are two requirements, UCAN adopts a [certificate capability model] related to [SPKI].

## 1.2 Intuition for Auth System Differences 

The below analogies illustrate several significant tradeoffs between these systems but are only accurate enough to build intuition. A good resource for a more thorough presentation of these tradeoffs is [Capability Myths Demolished]. In this framework, UCAN approximates SPKI with some dynamic features.

### 1.2.1 Access Control Lists

By analogy, ACLs are like a bouncer at an exclusive event. This bouncer has a list attendees allowed in and which of those are VIPs that get extra access. People trying to get in show their government-issued ID and are accepted or rejected. In addition, they may get a lanyard to identify that they have previously been allowed in. If someone is disruptive, they can simply be crossed off the list and denied further entry.

If there are many such events at many venues, the organizers need to coordinate ahead of time, denials need to be synchronized, and attendees need to show their ID cards to many bouncers. The likelihood of the bouncer letting in the wrong person due to synchronization lag or confusion by someone sharing a name is nonzero.

### 1.2.2 Certificate Capabilities

UCANs work more like [movie tickets][caps as keys] or a festival pass. No one needs to check your ID; who you are is irrelevant. For example, if you have a ticket issued by the theatre to see Citizen Kane, you are admitted to Theater 3. If you cannot attend an event, you can hand this ticket to a friend who wants to see the film instead, and there is no coordination required with the theater ahead of time. However, if the theater needs to cancel tickets for some reason, they need a way of uniquely identifying them and sharing this information between them.

### 1.2.3 Object Capabilities

Object capability ("ocap") systems use a combination of references, encapsulated state, and proxy forwarding. As the name implies, this is fairly close to object-oriented or actor-based systems. Object capabilities are [robust][Robust Composition], flexible, and expressive.

To achieve these properties, object capabilities have two requirements: [fail-stop], and locality preservation. The emphasis on consistency rules out partition tolerance[^pcec].

[^pcec]: To be precise, this is a [PC/EC][PACELC] system, which is a critical tradeoff for many systems. UCAN can be used to model both PC/EC and PA/EL, but is most typically PC/EL.

## 1.3 Security Considerations

Each UCAN includes a constructive set of assertions of what it is allowed to do. Note that this is not a predicate: it is a positive assertion of authority. "Proofs" are positive evidence (elsewhere called "witnesses") of the possession of rights. They are cryptographically verifiable chains showing that the UCAN issuer either claims to directly own a resource, or that it was delegated to them by some claimed owner. In the most common case, the root owner's ID is the only globally unique identity for the resource.

Root capability issuers function as verifiable, distributed roots of trust. The delegation chain is by definition a provenance log. Private keys themselves SHOULD NOT move from one context to another. Keeping keys unique to each physical device and unique per use case is RECOMMENDED to reduce opportunity for keys to leak, and limit blast radius in the case of compromises. "Sharing authority without sharing keys" is provided by capabilities, so there is no reason to share keys directly.

Note that a structurally and cryptographicaly valid UCAN chain can be semantically invalid. The executor MUST verify the ownership of any external resources at execution time. While not possible for all use cases (e.g. replicated state machines and eventually consistent data), having the executor be the resource itself is RECOMMENDED.

While certificate chains go a long way toward improving security, they do not provide [confinement] on their own. The principle of least authority SHOULD be used when delegating a UCAN: minimizing the amount of time that a UCAN is valid for and reducing authority to the bare minimum required for the delegate to complete their task. This delegate should be trusted as little as is practical since they can further sub-delegate their authority to others without alerting their delegator. UCANs do not offer confinement (as that would require all processes to be online), so it is impossible to guarantee knowledge of all of the sub-delegations that exist. The ability to revoke some or all downstream UCANs exists as a last resort.

## 1.4 Inversion of Control

This is achieved due to two properties: self-certifying delegation and reference passing. There is no Authorization Server (AS) that sits between requestors and resources. In traditional terms, the owner of a UCAN resource is the resource server (RS) directly.

This inverts the usual relationship between resources and users: the resource grants some (or all) authority over itself to agents, as opposed an Authorization Server managing the relationship between them. This has several major advantages:

- Fully distributed and scalable
- Self-contained request without intermediary 
- Partition tolerance, [support for replicated data and machines][overcoming SSI]
- Flexible granularity
- Compositionality: no distinction between resources residing together or apart

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│             │   │             │   │             │
│             │   │ ┌─────────┐ │   │             │
│             │   │ │  Bob's  │ │   │             │
│             │   │ │  Photo  │ │   │             │
│             │   │ │ Gallery │ │   │             │
│             │   │ └─────────┘ │   │             │
│             │   │             │   │             │
│   Alice's   │   │    Bob's    │   │   Carol's   │
│    Stuff    │   │    Stuff    │   │    Stuff    │
│             │   │             │   │             │
│     ┌───────┼───┼─────────────┼───┼──┐          │
│     │       │   │             │   │  │          │
│     │       │   │         ┌───┼───┼──┼────────┐ │
│     │       │   │ Alice's │   │   │  │        │ │
│     │       │   │  Music  │   │   │  │Carol's │ │
│     │       │   │ Player  │   │   │  │  Game  │ │
│     │       │   │         │   │   │  │        │ │
│     │       │   │         └───┼───┼──┼────────┘ │
│     │       │   │             │   │  │          │
│     └───────┼───┼─────────────┼───┼──┘          │
│             │   │             │   │             │
└─────────────┘   └─────────────┘   └─────────────┘
```

This additionally allows UCAN to model auth for [eventually consistent and replicated state][Post-SSI].

# 2 Lifecycle

The UCAN lifecycle has three parts:

| Spec         | Description                                                               | Requirement Level |
|--------------|---------------------------------------------------------------------------|-------------------|
| [Delegation] | Pass, attenuate, and secure authority in a partition-tolerant way         | REQUIRED          |
| [Invocation] | Exercise authority that has been delegated through one or more delegatees | REQUIRED          |
| [Revocation] | Undo a delegation, breaking a delegation chain for mallicious users       | RECOMMENDED       |

``` mermaid
flowchart TD
    rev[Revocation]
    inv[Invocation]
    del[Delegation]

    rev -->|is a kind of| inv -->|is proven by| del
    rev -.->|invalidates| del

    click del href "https://github.com/ucan-wg/delegation" "UCAN Delegation Spec"
    click inv href "https://github.com/ucan-wg/invocation" "UCAN Invocation Spec"
    click rev href "https://github.com/ucan-wg/revocation" "UCAN Revocation Spec"
```

## 2.1 Example

Here is a concrete example of all stages of the UCAN lifecycle for database write access.

``` mermaid
sequenceDiagram
    participant Database

    actor DBAgent
    actor Alice
    actor Bob

    Note over Database, DBAgent: Set Up Agent-Owned Resource
    DBAgent ->> Database: createDB()

    autonumber 1

    Note over DBAgent, Bob: Delegation
    DBAgent -->> Alice: delegate(DBAgent, write)
    Alice -->> Bob: delegate(DBAgent, write)

    Note over Database, Bob: Invocation
    Bob ->> DBAgent: invoke(DBAgent, [write, [key, value]], proof: [➊,➋])
    DBAgent ->> Database: write(key, value)
    DBAgent ->> Bob: ACK

    Note over DBAgent, Bob: Revocation
    Alice ->> DBAgent: revoke(➋, proof: [➊,➋])
    Bob ->> DBAgent: invoke(DBAgent, [write, [key, newValue]], proof: [➊,➋])
    DBAgent -X Bob: NAK(➏) [rejected]
```

# 2. Terminology

## 2.1 Roles

There are several roles that an agent MAY assume:

| Name      | Description                                                                                      | 
|-----------|--------------------------------------------------------------------------------------------------|
| Agent     | The general class of entities and principals that interact with a UCAN                           |
| Audience  | The Principal delegated to in the current UCAN. Listed in the `aud` field                        |
| Executor  | The Agent that actually performs the action described in an invocation                           |
| Invoker   | A Principal that requests that an Executor perform some action that uses the Invoker's authority |
| Issuer    | The Principal of the current UCAN. Listed in the `iss` field                                     |
| Owner     | A Subect that controls some external resource                                                    |
| Principal | An agent identified by DID (listed in a UCAN's `iss` or `aud` field)                             |
| Revoker   | The Issuer listed in a proof chain that revokes a UCAN                                           |
| Subject   | The Principal who's authority is delegated or invoked                                            |
| Validator | Any Agent that interprets a UCAN to determine that it is valid, and which capabilities it grants |

``` mermaid
flowchart TD
    subgraph Agent
        subgraph Principal
            direction TB

            subgraph Issuer
                direction TB
                
                subgraph Subject
                    direction TB
                    
                    Executor
                    Owner
                end

                Revoker
            end

            subgraph Audience
                Invoker
            end
        end

        Validator
    end
```

## 2.3 Subject

A Subject MUST be referenced by [DID]. This behaves much like a [GUID], with the addition of public key verifiability. This unforgeability prevents mallicious namespace collisions which can lead to [confused deputies].

## 2.3 Resource

A resource is some data or process that can be uniquely identified by a [URI]. It can be anything from a row in a database, a user account, storage quota, email address, etc. Resource MAY be as coarse or fine grained as desired. Finer-grained is RECOMMENDED where possible, as it is easier to model the principle of least authority ([POLA]).

A resource describes the noun of a capability. It is either The resource pointer MUST be provided in [URI] format. Arbitrary and custom URIs MAY be used, provided that the intended recipient can decode the URI. The URI is merely a unique identifier to describe the pointer to — and within — a resource.

Having a unique agent represent a resource (and act as its manager) is RECOMMENDED. However, to help traditional ACL-based systems transition to certificate capabilities, an agent MAY manage multiple resources, and act as the registrant in the ACL system. These situatons can be represneted as follows:

Unless explicitly stated, the Resource of a UCAN MUST be the Subject.

## 2.3 Operation

Operations are concrete messages ("verbs") that MUST be unambiguously interpretable by the Subject of a UCAN. Operations are REQUIRED in invocations. Some examples include `msg/send`, `crud/read`, and `ucan/revoke`.

Much like other message-passing systems, the specific resource MUST define the behaviour for a particular message. For instance, `crud/update` MAY be used to destructively update a database row, or append to a append-only log. Specific messages MAY be created at will; the only restriction is that the Executor understand how to interpret that message in the context of a specific resource.

While arbitrary semantics MAY be described, they MUST apply to the target resource. For instance, it does not make sense to apply `msg/send` to a typical file system. 

Operations MUST NOT be case-sensitive. There MUST be at least one path segment as a namespace. For example, `http/put` and `db/put` MUST be treated as unique from each other.

## 2.4 Ability

Abilities abstract over Operations to allow for extension of UCAN delegations.

Abilities MAY be organized in a hierarchy. A typical example is a superuser capability ("anything") on a file system. Another is read vs write access, such that in an HTTP context, `WRITE` implies `PUT`, `PATCH`, `DELETE`, and so on. Organizing abilities this way allows for adding more options over time in a backward-compatible manner, avoiding the need to reissue UCANs with new resource semantics.

## 2.5 Caveats

Capabilities MAY define additional optional or required fields specific to their use case in the caveat fields. This field is OPTIONAL in the general case, but MAY be REQUIRED by particular capability types that require this information to validate. Caveats MAY function as an "escape hatch" for when a use case is not fully captured by the resource and ability fields. Caveats can be read as "on the condition that `<some caveat>` holds".

A common use for the caveats field is to define the resource that sits behind the Subject.

## 2.6 Capability

A capability is the association of an ability to a subject: `subject x ability x caveats`.

The resource and ability fields are REQUIRED. Any non-normative extensions are OPTIONAL.

For example, a capability may used to represent the ability to send email from a certain address on Fridays:

| Field   | Example                                                    |
|---------|------------------------------------------------------------|
| Subject | `did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK` |
| Ability | `msg/send`                                                 |
| Caveat  | `{sender: "mailto:alice@example.com", "day": "friday"}`    |

## 2.7 Authority

The set of capabilities delegated by a UCAN is called its "authority." This functions as a declarative description of delegated abilities.

Merging capability authorities MUST follow set semantics, where the result includes all capabilities from the input authorities. Since broader capabilities automatically include narrower ones, this process is always additive. Capability authorities can be combined in any order, with the result always being at least as broad as each of the original authorities.

``` plaintext
                 ┌───────────────────┐  ─┐
                 │                   │   │
                 │                   │   │
┌────────────────┼───┐               │   │
│                │   │ Subject B     │   │
│                │   │               │   │       BxZ
│                │   │     X         │   ├─── Capability
│     Subject A  │   │               │   │
│                │   │ Ability Z     │   │
│         X      │   │               │   │
│                │   │               │   │
│     Ability Y  │   │               │   │
│                └───┼───────────────┘  ─┘
│                    │
│                    │
└────────────────────┘

└──────────────────┬─────────────────┘
                   │

               AxY U BxZ
               authority
```

The capability authority is the total rights of the authorization space down to the relevant volume of authorizations. Individual capabilities MAY overlap; the authority is the union. Except for [rights amplification], every unique delegation MUST have equal or narrower capabilities from their delegator. Inside this content space, you can draw a boundary around some resource(s) (their type, identifiers, and paths or children) and their capabilities.

## 2.8 Attenuation

Attenuation is the process of constraining the capabilities in a delegation chain. Each direct delegation MUST either directly restate or attenuate (diminish) its capabilities.

# 3. Token Resolution

Token resolution is transport specific. The exact format is left to the relevant UCAN transport specification. At minimum, such a specification MUST define at least the following:

1. Request protocol
2. Response protocol
3. Collections format

Note that if an instance cannot dereference a CID at runtime, the UCAN MUST fail validation. This is consistent with the [constructive semantics] of UCAN.

# 4. Implementation Recommendations

## 4.1 Delegation Store

A validator MAY keep a local store of UCANs that it has received. UCANs are immutable but also time-bound so that this store MAY evict expired or revoked UCANs.

This store SHOULD be indexed by CID (content addressing). Multiple indices built on top of this store MAY be used to improve capability search or selection performance.

## 4.2 Memoized Validation

Aside from revocation, capability validation is idempotent. Marking a CID (or capability index inside that CID) as valid acts as memoization, obviating the need to check the entire structure on every validation. This extends to distinct UCANs that share a proof: if the proof was previously reviewed and is not revoked, it is RECOMMENDED to consider it valid immediately.

Revocation is irreversible. Suppose the validator learns of revocation by UCAN CID or issuer DID. In that case, the UCAN and all of its derivatives in such a cache MUST be marked as invalid, and all validations immediately fail without needing to walk the entire structure.

## 4.3 Replay Attack Prevention

Replay attack prevention is REQUIRED ([Token Uniqueness]). The exact strategy is left to the implementer. Some simple strategies include maintaining a set of previously seen CIDs, or requiring that nonces be monotonically increasing per principal. This MAY be the same structure as a validated UCAN memoization table (if one is implementated).

Maintaining a secondary token expiry index is RECOMMENDED. This enables garbage collection and more efficient search. In cases of very large stores, normal cache performance techniques MAY be used, such as Bloom filters, multi-level caches, and so on.

## 4.4 Beyond Single System Image

> As we continue to increase the number of globally connected devices, we must embrace a design that considers every single member in the system as the primary site for the data that it is generates. It is completely impractical that we can look at a single, or a small number, of globally distributed data centers as the primary site for all global information that we desire to perform computations with.
>
> —[Meiklejohn], [A Certain Tendency Of The Database Community]

FIXME Expand

Unlike many authorization systems where a service controls access to resources in their care, location-independent, offline, and leaderless resources require control to live with the user. Therefore, the same data MAY be used across many applications, data stores, and users.




``` mermaid
sequenceDiagram
    participant CRDT as Inital Grow-Only Set (CRDT)

    actor Alice
    actor Bob
    actor Carol
    
    autonumber

    Note over CRDT, Bob: Setup
    CRDT -->> Alice: delegate(CRDT_ID, append)
    CRDT -->> Bob: delegate(CRDT_ID, append)
    
    Note over Bob, Carol: Bob Invites Carol
    Bob -->> Carol: delegate(CRDT_ID, append)

    Note over Alice, Carol: Direct P2P Gossip
    Carol ->> Bob: invoke(CRDT_ID, append, {C1}, proof: [➋,➍])
    Alice ->> Carol: invoke(CRDT_ID, append, {A1}}, proof: [➊])
    Bob ->> Alice: invoke(CRDT_ID, append, {B1, C1}, proof: [➋])
```

## 4.5 Wrapping Existing Systems

In the RECOMMENDED scenario, the agent controlling a resource has a unique reference to it. This is always possible in a system that has adopted capabilities end-to-end.

Interacting with existing systems MAY require relying on ambient authority contained in an ACL, nonunique reference, or other authorization logic. These cases are still compatible with UCAN, but the security guarantees are weaker since 1. the surface area is larger, and 2. part of the auth system lives outside UCAN.

``` mermaid
sequenceDiagram
    participant Database
    participant ACL as External Auth System

    actor DBAgent
    actor Alice
    actor Bob

    Note over ACL, DBAgent: Setup
    DBAgent ->> ACL: signup(DBAgent)
    ACL ->> ACL: register(DBAgent)

    autonumber 1

    Note over DBAgent, Bob: Delegation
    DBAgent -->> Alice: delegate(DBAgent, write)
    Alice -->> Bob: delegate(DBAgent, write)

    Note over Database, Bob: Invocation
    Bob ->>+ DBAgent: invoke(DBAgent, [write, key, value], proof: [➊,➋])

    critical External System
        DBAgent ->> ACL: getToken(write, key, AuthGrant)
        ACL ->> DBAgent: AccessToken
        
        DBAgent ->> Database: request(write, value, AccessToken)
        Database ->> DBAgent: ACK
    end

    DBAgent ->>- Bob: ACK
```

# 5. Related Work and Prior Art

[SPKI/SDSI] is closely related to UCAN. A different format is used, and some details vary (such as a delegation-locking bit), but the core idea and general usage pattern are very close. UCAN can be seen as making these ideas more palatable to a modern audience and adding a few features such as content IDs that were less widespread at the time SPKI/SDSI were written.

[ZCAP-LD] is closely related to UCAN. The primary differences are in formatting, addressing by URL instead of CID, the mechanism of separating invocation from authorization, and single versus multiple proofs.

[CACAO] is a translation of many of these ideas to a cross-blockchain invocation model. It contains the same basic concepts but is aimed at small messages and identities that are rooted in mutable documents rooted on a blockchain and lacks the ability to subdelegate capabilities.

[Local-First Auth] is a non-certificate-based approach, instead relying on a CRDT to build up a list of group members, devices, and roles. It has a friendly invitation mechanism based on a [Seitan token exchange]. It is also straightforward to see which users have access to what, avoiding the confinement problem seen in many decentralized auth systems.

[Macaroon] is a MAC-based capability and cookie system aimed at distributing authority across services in a trusted network (typically in the context of a Cloud). By not relying on asymmetric signatures, Macaroons achieve excellent space savings and performance, given that the MAC can be checked against the relevant services during discharge. The authority is rooted in an originating server rather than with an end-user.

[Biscuit] uses Datalog to describe capabilities. It has a specialized format but is otherwise in line with UCAN.

[Verifiable credentials] are a solution for data about people or organizations. However, they are aimed at a slightly different problem: asserting attributes about the holder of a DID, including things like work history, age, and membership.

# 6. Acknowledgments

Thank you to [Brendan O'Brien] for real-world feedback, technical collaboration, and implementing the first Golang UCAN library.

Many thanks to [Hugo Dias], [Mikael Rogers], and the entire DAG House team for the real world feedback, and finding inventive new use cases.

Thank you [Blaine Cook] for the real-world feedback, ideas on future features, and lessons from other auth standards.

Many thanks to [Brian Ginsburg] and [Steven Vandevelde] for their many copy edits, feedback from real world usage, maintenance of the TypeScript implementation, and tools such as [ucan.xyz].

Many thanks to [Christopher Joel] for his real-world feedback, raising many pragmatic considerations, and the Rust implementation and related crates.

Many thanks to [Christine Lemmer-Webber] for her handwritten(!) feedback on the design of UCAN, spearheading the [OCapN] initiative, and her related work on [ZCAP-LD].

Thanks to [Benjamin Goering] for the many community threads and connections to [W3C] standards.

Thanks to [Juan Caballero] for the numerous questions, clarifications, and general advice on putting together a comprehensible spec.

Thank you [Dan Finlay] for being sufficiently passionate about [OCAP] that we realized that capability systems had a real chance of adoption in an ACL-dominated world.

Thanks to [Martin Kleppmann] conversations exporating options for access control on CRDTs and [local-first] applications.

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

# 7. FAQ

## 7.1 What prevents an unauthorized party from using an intercepted UCAN?

UCANs always contain information about the sender and receiver. A UCAN is signed by the sender (the `iss` field DID) and can only be created by an agent in possession of the relevant private key. The recipient (the `aud` field DID) is required to check that the field matches their DID. These two checks together secure the certificate against use by an unauthorized party.

## 7.2 What prevents replay attacks on the invocation use case?

A UCAN delegated for purposes of immediate invocation MUST be unique. If many requests are to be made in quick succession, a nonce can be used. The receiving agent (the one to perform the invocation) checks the hash of the UCAN against a local store of unexpired UCAN hashes.

This is not a concern when simply delegating since presumably the recipient agent already has that UCAN.

## 7.3 Is UCAN secure against person-in-the-middle attacks?

_UCAN does not have any special protection against person-in-the-middle (PITM) attacks._

Were a PITM attack successfully performed on a UCAN delegation, the proof chain would contain the attacker's DID(s). It is possible to detect this scenario and revoke the relevant UCAN but does require special inspection of the topmost `iss` field to check if it is the expected DID. Therefore, it is strongly RECOMMENDED to only delegate UCANs to agents that are both trusted and authenticated and over secure channels.

<!-- Internal Links -->

<!-- External Links -->

[A Certain Tendency Of The Database Community]: https://arxiv.org/pdf/1510.08473.pdf
[Alan Karp]: https://github.com/alanhkarp
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Benjamin Goering]: https://github.com/gobengo
[Biscuit]: https://github.com/biscuit-auth/biscuit/
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede 
[CACAO]: https://blog.ceramic.network/capability-based-data-security-on-ceramic/
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats
[Capability Myths Demolished]: https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[DID fragment]: https://www.w3.org/TR/did-core/#fragment
[DID path]: https://www.w3.org/TR/did-core/#path
[DID subject]: https://www.w3.org/TR/did-core/#dfn-did-subjects
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[ECDSA security]: https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security
[FIDO]: https://fidoalliance.org/fido-authentication/
[Fission]: https://fission.codes
[GUID]: https://en.wikipedia.org/wiki/Universally_unique_identifier
[Hugo Dias]: https://github.com/hugomrdias
[Irakli Gozalishvili]: https://github.com/Gozala
[JWT]: https://datatracker.ietf.org/doc/html/rfc7519
[Juan Caballero]: https://github.com/bumblefudge
[Local-First Auth]: https://github.com/local-first-web/auth
[Macaroon]: https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf
[Mark Miller]: https://github.com/erights
[Martin Kleppmann]: https://martin.kleppmann.com/
[Meiklejohn]: https://christophermeiklejohn.com/
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: http://erights.org/elib/capability/index.html
[OCapN]: https://github.com/ocapn/ocapn
[PACELC]: https://en.wikipedia.org/wiki/PACELC_theorem
[POLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Philipp Krüger]: https://github.com/matheus23
[Protocol Labs]: https://protocol.ai/
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://datatracker.ietf.org/doc/html/rfc8037
[SHA2-256]: https://en.wikipedia.org/wiki/SHA-2
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Seitan token exchange]: https://book.keybase.io/docs/teams/seitan
[Steven Vandevelde]: https://github.com/icidasset
[URI]: https://www.rfc-editor.org/rfc/rfc3986
[Verifiable credentials]: https://www.w3.org/2017/vc/WG/
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[browser api crypto key]: https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey
[capabilities]: https://en.wikipedia.org/wiki/Object-capability_model
[caps as keys]: http://www.erights.org/elib/capability/duals/myths.html#caps-as-keys
[confinement]: http://www.erights.org/elib/capability/dist-confine.html
[constructive semantics]: https://en.wikipedia.org/wiki/Intuitionistic_logic
[content addressable storage]: https://en.wikipedia.org/wiki/Content-addressable_storage
[content addressing]: https://en.wikipedia.org/wiki/Content-addressable_storage
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L104
[delegation]: https://github.com/ucan-wg/delegation
[invocation]: https://github.com/ucan-wg/invocation
[local-first]: https://www.inkandswitch.com/local-first/
[raw data multicodec]: https://github.com/multiformats/multicodec/blob/a03169371c0a4aec0083febc996c38c3846a0914/table.csv?plain=1#L41
[revocation]: https://github.com/ucan-wg/revocation
[secure hardware enclave]: https://support.apple.com/en-ca/guide/security/sec59b0b31ff
[spki rfc]: https://www.rfc-editor.org/rfc/rfc2693.html
[time definition]: https://en.wikipedia.org/wiki/Temporal_database
[ucan.xyz]: https://ucan.xyz
