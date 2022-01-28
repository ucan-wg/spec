# User Controlled Authorization Network (UCAN) Specification v0.8.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)
* [Philipp Krüger](https://github.com/matheus23), [Fission](https://fission.codes)
* [Daniel Holmgren](https://github.com/dholms), `<REDACTED>`

# 0. Abstract

User Controlled Authorization Network (UCAN) is a trustless, secure, local-first, user-originated authorization and revocation scheme. It provides public-key verifable, delegatable, expressive, openly extensible [capabilities](https://en.wikipedia.org/wiki/Object-capability_model) by extending the familiar [JWT](https://datatracker.ietf.org/doc/html/rfc7519) structure. UCANs achieve public verifiability via chained certificates, and [decentralized identifiers (DIDs)](https://www.w3.org/TR/did-core/). Verifyable chain compression is enabled via [content addressing](https://en.wikipedia.org/wiki/Content-addressable_storage). UCAN improves on the familiarity and adoptability of schemes like [SPKI/SDSI](https://theworld.com/~cme/html/spki.html) for a web context and native application context, allowing creation and discharge by any agent with a DID, including peer-to-peer beyond traditional cloud computing.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1. Introduction

## 1.1 Motivation

Since at least the release of Unix, the most popular form of digital authorization has been access control lists (ACLs), where a list of what each user is allowed to do is maintained on the resource. This has been a successful model, and is well suited to architectures where persistent access to a single list is viable, and rules are sufficently well specified, such as in a centralized database.

With increasing interconnectivity between machines becomes commonplace, authorization needs to scale to meet the demands of the high load, partition faulty reality of distributed systems. It is not always practical to maintain a single central source of authorizations. Even if distirbuting the copies of a list to many authorization servers, latency and patitions introduce particularly troblesome challenges with conflicting updates, to say nothing of storage requirements.

An large portion of personal information now also moves through connected systems. Data privacy is a prominent theme when considering the design of modern applications, to the point of being legislated in parts of the world. 

Ahead-of-time coordination is often a barrier to development in many projects. Flexbilty to define specialized authorization sementics for a resources, and the ability to trustlessly integrate with external systems are important as the number of autonomous, specilialized, and coordinating applications increases.

Many high-value applications run in hostile environments. In recognition of this, many vendors have started including public key functionality, such as [nonextractable keys in browsers](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey), [cerificate systems for external keys](https://fidoalliance.org/fido-authentication/), and [secure hardware enclaves](https://support.apple.com/en-ca/guide/security/sec59b0b31ff) in widespread consumer devices.

Two related models that work extremeley well in the above context are Simple Public Key Infrastructure ([SPKI](https://www.rfc-editor.org/rfc/rfc2693.html)) and object capabilities ([OCAP](http://erights.org/elib/capability/index.html)). Since offline operation and self-verifiability are two requirements, UCAN adopts an approach closely related to SPKI. UCANs follow the "capabilties as certificates" model, with extensions for revocation and stateful capabilities.

## 1.2 Intuition

By analogy, ACLs are like a bouncer at an exclusive event. This bouncer has a list names of who is allowed in, and which of those are VIPs that get extra access. The attendees show their government-issued ID, and are accepted or rejected. They may get a lanyard to identify that they have previously been allowed in. If someone is disruptive, they can simply be crossed off the list and denied further entry.

If there are many such events at many venues, then the organizers need to coordinate ahead of time, denials need to be synchronized, and attendees need to show their ID cards to many bouncers. The likelihood of the bouncer letting in the wrong person due to synchronization lag or confusion by someone sharing a name is nonzero.

UCANs work more like [movie tickets](http://www.erights.org/elib/capability/duals/myths.html#caps-as-keys) or a festival pass between multiple venues. At the door, no one needs to check your ID; who you are is irrelevant. You have a ticket to see Citizen Kane, and are admitted to Theatre 3. If you are unable to attend an event, you can hand this ticket to a friend who wants to see the film instead, and there is no coordination required with the theatre ahead of time. If the theatre needs to cancel tickets for some reason, they need a way of uniquely identifying them and sharing this information between them.

The above analogies illustrate several important tradeoffs between these systems, but is only accurate enough to build an intuition. For a more thorough presentation of these tradeoffs, a good resource is [Capability Myths Demolished](https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf). In this framework, UCAN approximates SPKI with some dynamic features.

## 1.3 Security Considerations

* attenuation & contextual confinment (macaroon paper)

* Each UCAN includes a constructive assrtion of what it is allowed to do (note: not a predicate)


* ambient authority, confused deputy

User-controlled data is a 90° rotation of the above picture. Users are in control of their data: it's stored single tenant, with apps asking for stripes of data. Here, the user is in control, and apps ask for permission to see some portion of the user's data.


What if you're writing a collaborative application, and it needs access across user stores? Not a problem: users can grant access to their stores, and the caller can glue these rights together.

![](./overlapping_rights.jpg)

# 2. Terminology

## 2.1 Resource

A resource is some data or process that has an address. It can be anything from a row in a database, a user account, storage quota, email address, and so on.

## 2.2 Action

An action possible on some resource. Each action MAY have its own semantics. They MAY be unary, support a heirarchy, be monotone, patrial orders, and so on. These MAY be general and applicable to many kinds of resource, or tied to a specific one.

For instance, `wnfs/APPEND` is an action for Webnative filesystem paths. The action `wnfs/OVERWRITE` also implies the ability to append. Email has no such tiered relationship. One can `email/SEND`, but there is no concept of a ”super send”.

## 2.3 Capability

A capability is the association of an action to a resource: `resource x action`

## 2.4 Scope

An authorization scope is a set of capabilities. Scopes MUST compose with set semantics, so multiple scopes in an array MAY be considered the (deduplicated) union of all of the inner scopes.

![](./assets/scope_union.jpg)

The "scope" is the total rights of the authorization space down to the relevant volume of authorizations. With the exception of rights amplification, every individual delegation MUST have equal or less scope from their delegator.

Inside this content space, can draw a boundary around some resource(s) (their type, identifiers, and paths or children), and their capabilities.

As a practical matter, since scopes form a group, you can be fairly loose: order doesn’t matter, and merging resources can be quite broad since the more powerful of any overlap will take precedence (i.e. you don’t need a clean separation).

## 2.5 Delegation

Delegation is the act of granting another principal (the delegatee) the capability to use a resource that another has (the delegator). This MUST be proven by a "witness", which is either the signature of the owning principal, or a UCAN that has access to that capability in its scope.

Each direct delegation leaves the potency of the action at the same level, or diminishes it. The only exception is in "rights amplification", where a delegation MAY be proven by one-or-more witnesses of a different types, if part of the resource's semantics. 

## 2.6 Attenuation

The process of diminishing rights in a delegation chain.

## 2.7 Revocation

* Subchain revocation
* DID revocation

# 3. Invocation Security



# 4. JWT Structure

UCANs MUST be formatted as JWTs, with additional required and optional keys. The overall container of a header, claims, and siganture remain. Please refer to RFC 7519 for more omn this format

## 4.1 Header

The header MUST include all of the following fields:
| Field | Type     | Description                    | Required |
|-------|----------|--------------------------------|----------|
| `alg` | `String` | Signature algorithm            | Yes      |
| `typ` | `String` | Type (MUST be `"JWT"`)         | Yes      |
| `ucv` | `String` | UCAN Semantic Version (v2.0.0) | Yes      |

This is a standard JWT header, with an additional REQUIRED field `ucv`. This field sets the the version of the UCAN specification used in the payload.

EdDSA as applied to JOSE (including JWT) is described in [RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037).

### Examples

```json
{
  "alg": "EdDSA",
  "typ": "JWT"
  "ucv": "0.8.0"
}
```

## 4.2 Payload

The payload MUST describe the authorization claims being made, who is involved, and its validity period.

| Field | Type       | Description                                      | Required |
|-------|------------|--------------------------------------------------|----------|
| `iss` | `String`   | Issuer DID (sender)                              | Yes      |
| `aud` | `String`   | Audience DID (receiver)                          | Yes      |
| `nbf` | `Number`   | Not Before UTC Unix Timestamp (valid from)       | No       |
| `exp` | `Number`   | Expiration UTC Unix Timestamp (valid until)      | Yes      |
| `nnc` | `String`   | Nonce                                            | No       |
| `fct` | `Json[]`   | Facts (asserted, signed data)                    | No       |
| `prf` | `String[]` | Proof of delegation (witnesses are nested UCANs) | Yes      |
| `att` | `Json[]`   | Attenuations                                     | Yes      |

### 4.2.1 Principals

The `iss` and `aud` fields describe the token's principals. This can be conceptualized as a "to" and "from" of a postal letter. The token MUST be signed with the private key associated with the DID in the `iss` field. Implementations MUST include the [`did:key` method](https://w3c-ccg.github.io/did-method-key/), and MAY be augmented with [additional DID methods](https://www.w3.org/TR/did-core/).

#### Examples

```json
"aud": "did:key:zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV",
"iss": "did:key:zAKJP3f7BD6W4iWEQ9jwndVTCBq8ua2Utt8EEjJ6Vxsf",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:pkh:eth:0xb9c5714089478a327f09197987f16f9e5d936e8a",
```

### 4.2.2 Time Bounds

`nbf` and `exp` stand for "not before" and "expires at" respectively. These are standard fields from RFC 7519 (JWT). Taken together they represent the time bounds for a token.

The `nbf` field is OPTIONAL. When omitted, the token MUST be treated as valid beginning from the Unix epoch. Setting the `nbf` field to a time in the future MUST delay use of a UCAN. For example, preprovisioning access to conference materials ahead of time, but not allowing access until the day of is achievable with judicious use of `nbf`.

The `exp` field MUST be set. If the time is in the past, the token MUST fail validation.

It is RECOMMENDED to keep the window of validity be as short as possible. By limiting the time range, the risk of a malicious user abusing a UCAN. This is situationally dependent, as trusted devices may not want to  reauthorize it very often. Due to clock drift, time bounds SHOULD NOT be considered exact. A buffer of ±60 seconds is RECOMMENDED.

#### Examples

```json
"nbf": 1529496683,
"exp": 1575606941,
```

### 4.2.3 Nonce

The OPTIONAL nonce parameter `nnc` MAY be any value. A randomly generated string is RECOMMENDED to provide a unique UCAN, though it MAY also be a monotonically increasing count of hash link. This field helps prevent replay attacks, and ensures a unique CID per delegation. The `iss`, `aud`, and `exp` fields together will often ensure that UCANs are unique, but adding the nonce ensures this uniqueness.

This field SHOULD NOT be used to sign arbitrary data, such as signature challenges. See the `fct` field for more.

#### Examples

``` json
"nnc": "1701-D"
```

### 4.2.4 Facts

The OPTIONAL `fct` field contains arbitrary facts and proofs of knowledge. The enclosed data MUST be self-evident and externally verifiable. It MAY include hash preimages, server challenges, a Merkle proof, dictionary data, and so on.

#### Examples

``` json
"fct": [
  {
    "challenge": "abcdef",
    "from": "example.com
  },
  {
    "sha3_256": "B94D27B9934D3E08A52E52D7DA7DABFAC484EFE37A5380EE9088F7ACE2EFCDE9",
    "msg": "hello world"
  }
]
```

### 4.2.5 Attenuation Scope

The attenuation scope (i.e. UCAN output, or "caveats") MUST be an array of heterogeneous access scopes (defined below). This array MAY be empty.

This array MUST contain some or none of the following: a strict subset (attenuation) of the proofs, resources originated by the `iss` DID (i.e. by parenthood), and resources that compositions of others (see rights amplification). This scoping also includes time ranges, making the proofs that starts latest and end soonest the lower and upper time bounds.

Each capability has its own semantics, which need to be interpretable by the target resource handler. A particular validator SHOULD NOT reject UCANs with resources that it does not know how to interpret.

The attenuation field MUST contain either a wildcard (`*`), or an array of JSON objects. A JSON capability MUST contain the `with` and `can` field, and MAY contain additional field needed to describe the capability.

``` json
{
  "with": $RESOURCE,
  "can": $ABILITY
}
```

#### 4.2.5.1 Resource Pointer

A resource describes the noun of a capability. The resource pointer MUST be provided in [URI](https://datatracker.ietf.org/doc/html/rfc3986) format. Arbitrary and custom URIs MAY be used, provided that the intended recipient is able to decode the URI. The URI merely a unique identifier to describe the pointer to -- and within -- a resource.

The same resource MAY be validly addressed several forms. For instance a database may be addressed at the level of direct memory with `file`, via `sqldb` to gain access to SQL semantics, `http` to use web addressing, and `dnslink` to use Merkle DAGs inside DNS `TXT` records. 

Resource pointers MAY also include wildcards (`*`) to indicate "any resource of this type" -- even if not yet created -- bounded by attenuation witnesses. These are generally used for account linking. Wildcards are not required to delegate longer paths, as paths are generally taken as OR filters.

| Value | Meaning |
|-------|---------|
| `{"with": "wnfs://user.example.com/path", ...}`      | File paths in our file system        |
| `{"with": "my:*", ...}` | All apps that the iss has access to, including future ones |
| `{"with": "dnslink:*", ...}` | All apps that the iss has access to, including future ones |
| `{"with": "dnslink://myapp.fission.app", ...}` | A URL for the app (ideally the auto-assigned one) |

#### 4.2.5.2 Ability

The `can` field describes the verb portion of the capability: an action, potency, or ability. For instance, the standard HTTP methods such as `GET`, `PUT`, and `POST` would be possible `can` values for an `http` resource. Arbitrary semantics can be described, but must be a valid way to describe actions on the resource.

Abilities MAY be organized in a heirarchy with enums. A common example is super user access ("anything") on a file system. Another would be read vs write access, such that in an HTTP context `READ` implies `PUT`, `PATCH`, `DELETE`, and so on. Organizing potencies this way allows for adding more options over time in a backwards-compatible manner, avoiing the need to reissue UCANs with new resource semantics.

Abilities MUST NOT be case sensitive, and MAY be namespaced by at least one path segment. For instance, `http/PUT` and `foo/PUT` MUST be trated as unique from each other. If not explicitely namespaced, the namespace MUST be inferred to be the same as the scheme. For example, `PUT` for `http://exmaple.com/` MUST be inferred to be `http/PUT`.

The only reserved ability MUST be `"*"`. This MAY be used as part of the action for resources (such as `my`), but MAY NOT be available as part of others. 

#### Examples

``` json
"att": [
  {
    "with": "wnfs://boris.fission.name/public/photos/",
    "can": "crud/DELETE"
  },
  {
    "with": "wnfs://boris.fission.name/private/84MZ7aqwKn7sNiMGsSbaxsEa6EPnQLoKYbXByxNBrCEr",
    "can": "APPEND"
  },
  {
    "with": "mailto:boris@fission.codes",
    "can": "SEND"
  }
]
```

### 4.2.6 Proof of Delegation

The `prf` field MUST contain UCAN witnesses (the ”inputs” of a UCAN). As they need to be independently verifiable, proofs MUST either be the fully encoded version of a UCAN (including the signature), or the content address of the relevant proof. Attenuations not covered by a proof in the `prf` array MUST be treated as owned by the issuer DID.

Proofs referenced by content address MUST be resolvable by the recipient, for instance over a DHT or database. Since they have no availability concern, inlined proofs SHOULD be preferred, unless the proof chain has become too large for a request.

#### Examples

``` json
"prf": ["eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsInVhdiI6IjAuMS4wIn0.eyJhdWQiOiJkaWQ6a2V5OnpTdEVacHpTTXRUdDlrMnZzemd2Q3dGNGZMUVFTeUExNVc1QVE0ejNBUjZCeDRlRko1Y3JKRmJ1R3hLbWJtYTQiLCJpc3MiOiJkaWQ6a2V5Ono1QzRmdVAyRERKQ2hoTUJDd0FrcFlVTXVKWmROV1dINU5lWWpVeVk4YnRZZnpEaDNhSHdUNXBpY0hyOVR0anEiLCJuYmYiOjE1ODg3MTM2MjIsImV4cCI6MTU4OTAwMDAwMCwic2NwIjoiLyIsInB0YyI6IkFQUEVORCIsInByZiI6bnVsbH0.Ay8C5ajYWHxtD8y0msla5IJ8VFffTHgVq448Hlr818JtNaTUzNIwFiuutEMECGTy69hV9Xu9bxGxTe0TpC7AzV34p0wSFax075mC3w9JYB8yqck_MEBg_dZ1xlJCfDve60AHseKPtbr2emp6hZVfTpQGZzusstimAxyYPrQUWv9wqTFmin0Ls-loAWamleUZoE1Tarlp_0h9SeV614RfRTC0e3x_VP9Ra_84JhJHZ7kiLf44TnyPl_9AbzuMdDwCvu-zXjd_jMlDyYcuwamJ15XqrgykLOm0WTREgr_sNLVciXBXd6EQ-Zh2L7hd38noJm1P_MIr9_EDRWAhoRLXPQ"]
```

# 5. Reserved Capabilities

The following capabilities are REQUIRED to be implemented.

## 5.1 All Owned Resources

### 5.1.1. `my` Scheme

The `my` URI scheme represents ownership over a resource -- typically by parenthood -- at decision-time (i.e. the validator's "now"). Resources that are created after the UCAN was created MUST be included. This higher-order scheme descibes delegating some or all ambient authorty to another DID.

The use case of "pairing" two DIDs by delegating all current and future resources is not uncommon when a user would like to use multiple devices as "root", but does not have access to all of them directly at all times.

The format for this scheme is as follows:

```anbf
ownershipscheme = "my:" kind ["@" did]
kind = "*" / <scheme> 
```

The wildcard `*` resource MUST be taken to mean "everything" (all resources of all types).

A "subscheme" MAY be used to delegate some of that scheme controlled by parenthood. For example `my:dns` delegates access to all DNS records (`my:dns:*` works equally well since wildcards are also part of the DNS URI). `my:mailto` selects all owned email addresses controlled by this user.

Redelegating these to further DIDs in a chain MUST address the specific parent DID that owns that resource separated by an `@`. For instance: `my:*@did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` selects all resources originating from the specified DID, and `my:mailto@did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` selects email addresses from the DID. 

### 5.1.2 Action

The action for `my:*` MUST be `*`.

``` json
{"with": "my:*", "can": "*"}
```

For `my` capabilities scoped to some scheme, the action MUST be one normally associated with that resource. 

``` json
{"with": "my:dns", "can": "crud/UPDATE"}
```

## 5.2 UCAN Addressing

### 5.2.1 `ucan` Scheme

The `ucan` URI scheme defines addressing for UCANs and their fields

``` abnf
ucan_uri = "ucan:" selector
selector = "*" / cid / field_path / cid field_path
field_path = "/" <path>
```

For example, `ucan://bafkreihb5iw53yervbng7mncvk36exflyrbbdaioevcz5emlqho4tqju3a` selects the UCAN represented by the CID. `ucan:bafkreihb5iw53yervbng7mncvk36exflyrbbdaioevcz5emlqho4tqju3a/prf/*` selects all witnesses in the `prf` field of the present UCAN. `ucan:*` represents all of the UCANs in the current proof scope.






# 6. Validation

Each capability has its own semantics, which need to be interpretable by the target resource handler. A particular validator SHOULD NOT reject UCANs with resources that it does not know how to interpret.

If any of the following criterea are not met, the UCAN MUST be considered invalid.

## 6.1 Time

A UCAN's time bounds MUST NOT be considered valid if the current system time is prior to the `nbf` field, or after the `exp` field. This is called "ambient time validity".

All witnesses MUST contain time bounds equal to or wider than the UCAN being delegated to. If the witness expires before the outer UCAN -- or starts after it -- the reader MUST treat the UCAN as invalid. This is called "timely delegation".

A UCAN is valid inclusive from the `nbf` time, and until the `exp` field. If the current time is outside of these bounds, the UCAN MUST be considered invalid. A delegator or invoker SHOULD account for expected clock drift when setting these bounds. This is called "timely invocation".

## 6.2 Pricipal Alignment

In delegation, the `aud` field of every witness MUST match the `iss` field of the outer UCAN (the one being delegated to). This alignment MUST form a chain all the way back to the originating principal for each resource.

An agent discharging a capability MUST verify that the outermost `aud` field matches its own DID. If they do not match, the associated action MUST NOT be performed.

## 6.3 Witness Chaining

Each capability MUST either be originated by the issuer (root capability, or "parenthood"), or have one-or-more witnesses in the `prf` field to attest that this issuer is authorized to use that capability ("introduction"). In the introduction case, this check must be recursively applied to its witnesses, until a root witness is found (i.e. issued by the resource owner).

With the exception of rights amplification (below), each delegation of a capability MUST have equal or lesser power from its witness. The time bounds MUST also be equal to or contained inside the time bounds of the witnesses time bounds. This shrinkilowering of rights at each delegation is called "attenuation".

## 6.4 Rights Amplification

Some capabilities are more than the sum of their parts. The canonical example is a can of soup and a can opener. You need both to access the soup inside the can, but the can opener may come from a completely separate source than the can of soup. Such semantics MAY be implemented in UCAN capabilities. This means that validating a particular capabilties MAY require more than one direct witness. The relevant witnesses MAY be of a different resource and action from the amplified capabaility. The delegated capability MUST have this behaviour in its semantics, even if the witnesses do not.

## 6.5 Content Identifiers

UCANs MAY be referenced by content ID (CID), per the [multiformats/cid](https://github.com/multiformats/cid) specification. The resolution of these addresses is left to the implementation and end user, and MAY (non-exclusively) include the following: local store, distributed hash table (DHT), gossip network, or RESTful service.

CIDs MAY be used to refer to any UCAN: a witness in a delegation chain, or an entire UCAN. This has many benefits, some of which are outlined in the implementation recommendations of this document.

Due to the potential for unresolvable CIDs, this SHOULD NOT be the preferred method of transmission. "Inline witnesses" SHOULD be used whenever possible, and complete UCANs SHOULD be expanded. When a CID is used, it is RECOMMENDED that it be substituted as close to the top UCAN as possible (i.e. the invocation), and as few witnesses be referenced by CID, to keep the number of required CID resolutions to a minimum. As UCANs are signed, all further delegations would require CID resolution, and so SHOULD NOT be used when the intention is delegation rather than invocation. 

## 7. Implementation Recommendations

## 7.1 UCAN Store

A validator MAY keep a local store of UCANs that it has received. UCANs are immutable, but also time bound, so this store MAY evict expired or revoked UCANs.

This store MAY be indexed by CID (content addressing). Multiple indices built on top of this store MAY be used to improve capability search or selection performance.

## 7.2 Memoized Validation

Aside from revocation, UCAN validation is an idempotent action. Marking a CID as valid acts as memoization, obviating the need to check the entire structure on every validation. This extends to distinct UCANs that share a witness: if the witness was previously checked and is not revoked, it is RECOMMENDED to immedietly consider it valid.

Revocation is irreversible. If the validator learns of a revocation by UCAN CID or issuer DID, the UCAN and all of its derivatives in such a cache MUST be marked as invalid, and all validations immedietly fail without needing to walk the entire structure.

## 7.3 Session Content ID

If many invocations will be discharged during a session, the sender and receiver MAY agree to use the CID rather than creating new UCANs for every message. This saves bandwidth, and avoids needing to use another session token exchange mechanism, or bearer token with lower security, such as a shared secret.

# 8. Related Work and Prior Art

[SPKI/SDSI](https://datatracker.ietf.org/wg/spki/about/) is very closely related to UCAN. A different format is used, and some details vary (such as a deleation-locking bit), but the core idea and general usage pattern is very close. UCAN can be seen as a way of making these ideas more palletable to a modern audience, and adding a few features such as content IDs that were less widespread at the time SPKI/SDSI were written.

[ZCAP-LD](https://w3c-ccg.github.io/zcap-ld/) is closely related to UCAN, and the projects were started around the same time. The primary differences are in formatting, addressing by URL, and an emphasis on linked data rather than inlining for availabilty.

[CACAO](https://blog.ceramic.network/capability-based-data-security-on-ceramic/) is a translation of many of these ideas to a cross-blockchain invocation model. It contains the same basic ideas, but is aimed at small messages and identities that are rooted in mutable documents rooted on a blockchain.

[Local-First Auth](https://github.com/local-first-web/auth) uses a CRDT-based ACLs and key lockboxes for role-based signatures. This is a non-certificate based approach, instead relying on the CRDT and signed data to build up a list of roles and members. It does have a very friendly invitiation certificate mechanism in [Seitan token exchange](https://book.keybase.io/docs/teams/seitan). It is also very easy to see which users have access to what, avoiding the confinement problem seen in so many decentralized auth systems.

[Macaroon](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf) is a MAC-based capability and cookie system aimed at distirbuting authority across services in a trusted network (typically in the context of a Cloud). By not relying on asymmetric signatures, Macaroons achieve very good space savings and performance, given that the MAC can be checked against the relevant services during discharge. The authority is rooted in an originating server, rather than with an end user.

[Biscuit](https://github.com/biscuit-auth/biscuit/) uses Datalog to describe capabilities. It has a specialized format, but is otherwise largely in line with UCAN.

[Verifiable credentials](https://www.w3.org/2017/vc/WG/) are a solution for this on data about people or organziations.

# 9. Acknowledgements

Thank you to [Brendan O'Brien](https://github.com/b5) for real-world feedback, technical collaboration, and implementing the first Golang library for UCAN.

Many thanks to [Irakli Gozalishvili](https://github.com/Gozala) for feedback and recommendations, including suggesting renaming a field to `can`.

Thank you [Dan Finlay](https://github.com/danfinlay) for being sufficiently passionate about OCAP that we realized that such systems had an actual chance of adoption in an ACL-dominated world.

Thanks to the entire [SPKI WG](https://datatracker.ietf.org/wg/spki/about/) for their closely related prioneering work.

We want to especially recognize [Mark Miller](https://github.com/erights) for his numerous contributions to the field of distributed auth, programming langauges, and computer security.
