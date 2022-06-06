# User Controlled Authorization Network (UCAN) Specification v0.9.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)
* [Philipp Krüger](https://github.com/matheus23), [Fission](https://fission.codes)
* [Daniel Holmgren](https://github.com/dholms), [Bluesky](https://blueskyweb.xyz/)
* [Irakli Gozalishvili](https://github.com/Gozala), [Protocol Labs](https://protocol.ai/)

# 0. Abstract

User-Controlled Authorization Network (UCAN) is a trustless, secure, local-first, user-originated authorization and revocation scheme. It provides public-key verifiable, delegable, expressive, openly extensible [capabilities](https://en.wikipedia.org/wiki/Object-capability_model) by extending the familiar [JWT](https://datatracker.ietf.org/doc/html/rfc7519) structure. UCANs achieve public verifiability with chained certificates and [decentralized identifiers (DIDs)](https://www.w3.org/TR/did-core/). Verifiable chain compression is enabled via [content addressing](https://en.wikipedia.org/wiki/Content-addressable_storage). Being encoded with the familiar JWT, UCAN improves the familiarity and adoptability of schemes like [SPKI/SDSI](https://theworld.com/~cme/html/spki.html) for web and native application contexts. UCAN allows for the creation and discharge of authority by any agent with a DID, including traditional systems and peer-to-peer architectures beyond traditional cloud computing.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1. Introduction

## 1.1 Motivation

Since at least the release of Unix, access control lists (ACLs) have been the most popular form of digital authorization, where a list of what each user is allowed to do is maintained on the resource. ACLs have been a successful model suited to architectures where persistent access to a single list is viable. ACLs require that rules are sufficiently well specified, such as in a centralized database with rules covering all possible permutations of rights.

With increasing interconnectivity between machines becoming commonplace, authorization needs to scale to meet the load demands of distributed systems while providing partition tolerance. However, it is not always practical to maintain a single central authorization source. Even when copies of the authorization list are distributed to the relevant servers, latency and partitions introduce troublesome challenges with conflicting updates, to say nothing of storage requirements.

A large portion of personal information now also moves through connected systems. As a result, data privacy is a prominent theme when considering the design of modern applications, to the point of being legislated in parts of the world. 

Ahead-of-time coordination is often a barrier to development in many projects. Flexibility to define specialized authorization semantics for resources and the ability to integrate with external systems trustlessly are essential as the number of autonomous, specialized, and coordinating applications increases.

Many high-value applications run in hostile environments. In recognition of this, many vendors now include public key functionality, such as [non-extractable keys in browsers](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey), [certificate systems for external keys](https://fidoalliance.org/fido-authentication/), and [secure hardware enclaves](https://support.apple.com/en-ca/guide/security/sec59b0b31ff) in widespread consumer devices.

Two related models that work exceptionally well in the above context are Simple Public Key Infrastructure ([SPKI](https://www.rfc-editor.org/rfc/rfc2693.html)) and object capabilities ([OCAP](http://erights.org/elib/capability/index.html)). Since offline operation and self-verifiability are two requirements, UCAN adopts an approach closely related to SPKI. UCANs follow the "capabilities as certificates" model, with extensions for revocation and stateful capabilities.

## 1.2 Intuition

By analogy, ACLs are like a bouncer at an exclusive event. This bouncer has a list attendees allowed in and which of those are VIPs that get extra access. The attendees show their government-issued ID and are accepted or rejected. In addition, they may get a lanyard to identify that they have previously been allowed in. If someone is disruptive, they can simply be crossed off the list and denied further entry.

If there are many such events at many venues, the organizers need to coordinate ahead of time, denials need to be synchronized, and attendees need to show their ID cards to many bouncers. The likelihood of the bouncer letting in the wrong person due to synchronization lag or confusion by someone sharing a name is nonzero.

UCANs work more like [movie tickets](http://www.erights.org/elib/capability/duals/myths.html#caps-as-keys) or a festival pass between multiple venues. No one needs to check your ID; who you are is irrelevant. For example, if you have a ticket to see Citizen Kane, you are admitted to Theater 3. If you cannot attend an event, you can hand this ticket to a friend who wants to see the film instead, and there is no coordination required with the theater ahead of time. However, if the theater needs to cancel tickets for some reason, they need a way of uniquely identifying them and sharing this information between them.

The above analogies illustrate several significant tradeoffs between these systems but are only accurate enough to build intuition. A good resource for a more thorough presentation of these tradeoffs is [Capability Myths Demolished](https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf). In this framework, UCAN approximates SPKI with some dynamic features.

## 1.3 Security Considerations

Each UCAN includes a constructive set of assertions of what it is allowed to do. Note that this is not a predicate: it is a positive assertion of rights. "Proofs" are positive evidence (elsewhere called "witnesses") of the possession of rights. They are cryptographically signed data showing that the UCAN issuer either owns this or that it was delegated to them by the root owner.

This signature chain is the root of trust. Private keys themselves SHOULD NOT move from one context to another: this is what the delegation mechanism provides: "sharing authority without sharing keys."

UCANs (and other forms of PKI) depend on the ambient authority of the owner of each resource. This means that the discharging agent must be able to verify the root ownership at decision time. The rest of the chain in-between is self-certifying.

While certificate chains go a long way toward improving security, they do not provide [confinement](http://www.erights.org/elib/capability/dist-confine.html) on their own. The principle of least authority SHOULD be used when delegating a UCAN: minimizing the amount of time that a UCAN is valid for and reducing authority to the bare minimum required for the delegate to complete their task. This delegate should be trusted as little as is practical since they can further sub-delegate their authority to others without alerting their delegator. UCANs do not offer confinement (as that would require all processes to be online), so it is impossible to guarantee knowledge of all of the sub-delegations that exist. The ability to revoke some or all downstream UCANs exists as a last resort.

## 1.4 Inversion of Control

Unlike many authorization systems where a service controls access to resources in their care, location-independent, offline, and leaderless resources require control to live with the user. Therefore, the same data MAY be used across many applications, data stores, and users.

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

# 2. Terminology

## 2.1 Resource

A resource is some data or process that has an address. It can be anything from a row in a database, a user account, storage quota, email address, etc.

## 2.2 Action

An action MUST be performed against some resource. Each action MAY have its own semantics. For example, actions MAY be unary, support a hierarchy, be monotone, form partial orders, etc. In addition, actions MAY be general and applicable to many kinds of resources or tied to a specific one.

For instance, `wnfs/APPEND` is an action for WebNative filesystem paths. The action `wnfs/OVERWRITE` also implies the capacity to append. An email has no such tiered relationship. One can `email/SEND`, but there is no concept of a "super send."

## 2.3 Capability

A capability is the association of an "action" to a "resource": `resource x action`

## 2.4 Capability Scope

The set of capabilities delegated by a UCAN is called its "capability scope," often shortened to "scope." This functions as a declarative description of delegated abilities.

Merging capability scopes MUST follow set semantics, where the result includes all capabilities from the input scopes. Since broader capabilities automatically include narrower ones, this process is always additive. Capability scopes can be combined in any order, with the result always being at least as broad as each of the original scopes.

``` plaintext
                 ┌───────────────────┐  ─┐
                 │                   │   │
                 │                   │   │
┌────────────────┼───┐               │   │
│                │   │ Resource B    │   │     
│                │   │               │   │       BxZ
│                │   │     X         │   ├─── Capability
│     Resource A │   │               │   │
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
                 scope
```

The capability scope is the total rights of the authorization space down to the relevant volume of authorizations. Individual capabilities MAY overlap; the scope is the union. Except for [rights amplification](#54-rights-amplification), every unique delegation MUST have equal or narrower capabilities from their delegator. Inside this content space, you can draw a boundary around some resource(s) (their type, identifiers, and paths or children) and their capabilities.

For example, given the following scopes against a WebNative filesystem, they can be merged as follows:

```js
// "wnfs" abilities:
// FETCH < APPEND < OVERWRITE < SUPERUSER

ScopeA = [
  { "with": "wnfs://alice.example.com/pictures/", "can": "wnfs/APPEND" }
];

ScopeB = [
  { "with": "wnfs://alice.example.com/pictures/vacation/", "can": "wnfs/APPEND" };
  { "with": "wnfs://alice.example.com/pictures/vacation/hawaii/", "can": "wnfs/OVERWRITE"}
];

merge(ScopeA, ScopeB) == [
   {"with": "wnfs://alice.example.com/pictures/", "can": "wnfs/APPEND"},
   {"with": "wnfs://alice.example.com/pictures/vacation/hawaii", "can": "wnfs/OVERWRITE"}
   // Note that ("/pictures/vacation/" x APPEND) has become redundant, being contained in ("/pictures/" x APPEND)
];
```

## 2.5 Delegation

Delegation is the act of granting another principal (the delegate) the capability to use a resource that another has (the delegator). A constructive "proof" acts as the authorization proof for a delegation. Proofs are either the owning principal's signature or a UCAN with access to that capability in its scope.

Each direct delegation leaves the action at the same level or diminishes it. The only exception is in "rights amplification," where a delegation MAY be proven by one-or-more proofs of different types if part of the resource's semantics. 

Note that delegation is a separate concept from invocation [§2.8](#28-invocation). Delegation is the act of granting a capability to another, not the act of using it (invocation), which has additional requirements.

## 2.6 Attenuation

Attenuation is the process of constraining the capabilities in a delegation chain.

## 2.7 Revocation

Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

In the case of UCAN, this MUST be done by a proof's issuer DID. For more on the exact mechanism, see the revocation validation section.

## 2.8 Invocation

UCANs are used to delegate capabilities between DID-holding agents, eventually terminating in an "invocation" of those capabilities. Invocation is when the capability is exercised to perform some task on a resource. Note that **the only agent allowed to perform an action with a UCAN MUST be the one holding the DID private key associated with the `aud` field**. For more on the specifics of this validation, see [§5.2.1](#521-recipient-validation).

## 2.9 Time

Time takes on [multiple meanings](https://en.wikipedia.org/wiki/Temporal_database) in systems representing facts or knowledge. The senses of the word "time" are given below.

## 2.9.1 Valid Time Range

The period of time that a capability is valid from and until.

## 2.9.2 Assertion Time

The moment at which a delegation was asserted. This MAY be captured via an `iat` field, but is generally superfluous to capture in the token. "Assertion time" is useful when discussing the lifecycle of a token.

## 2.9.3 Decision (or Validation) Time

Decision time is the part of the lifecycle when "a decision" about the token is made. This is typically during validation, but also includes resolving external state (e.g. storage quotas).

# 3. JWT Structure

UCANs MUST be formatted as JWTs, with additional required and optional keys. The overall container of a header, claims, and signature remains. Please refer to [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) for more on this format.

## 3.1 Header

The header MUST include all of the following fields:
| Field | Type     | Description                    | Required |
| ----- | -------- | ------------------------------ | -------- |
| `alg` | `String` | Signature algorithm            | Yes      |
| `typ` | `String` | Type (MUST be `"JWT"`)         | Yes      |
| `ucv` | `String` | UCAN Semantic Version (v2.0.0) | Yes      |

The header is a standard JWT header, with an additional REQUIRED field `ucv`. This field sets the version of the UCAN specification used in the payload.

EdDSA, as applied to JOSE (including JWT), is described in [RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037).

### Examples

```json
{
  "alg": "EdDSA",
  "typ": "JWT",
  "ucv": "0.9.0"
}
```

## 3.2 Payload

The payload MUST describe the authorization claims, who is involved, and its validity period.

| Field | Type       | Description                                      | Required |
| ----- | ---------- | ------------------------------------------------ | -------- |
| `iss` | `String`   | Issuer DID (sender)                              | Yes      |
| `aud` | `String`   | Audience DID (receiver)                          | Yes      |
| `nbf` | `Number`   | Not Before UTC Unix Timestamp (valid from)       | No       |
| `exp` | `Number`   | Expiration UTC Unix Timestamp (valid until)      | Yes      |
| `nnc` | `String`   | Nonce                                            | No       |
| `fct` | `Json[]`   | Facts (asserted, signed data)                    | No       |
| `prf` | `String[]` | Proof of delegation (hash-linked UCANs)               | Yes      |
| `att` | `Json[]`   | Attenuations                                     | Yes      |

### 3.2.1 Principals

The `iss` and `aud` fields describe the token's principals. These can be conceptualized as the sender and receiver of a postal letter. The token MUST be signed with the private key associated with the DID in the `iss` field. Implementations MUST include the [`did:key` method](https://w3c-ccg.github.io/did-method-key/), and MAY be augmented with [additional DID methods](https://www.w3.org/TR/did-core/).

If an issuer's DID has more than one key (e.g. [`did:ion`](https://github.com/decentralized-identity/ion), [`did:3`](https://github.com/ceramicnetwork/CIP/blob/main/CIPs/CIP-79/CIP-79.md)), the key used to sign the UCAN MUST be explicitly, using the [DID fragment](https://www.w3.org/TR/did-core/#fragment) (the hash index).

The underlying key types RSA, ECDSA, and EdDSA MUST be supported. Use of ECDSA is supported but [RECOMMENDED that another key type be preferred](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security).

#### Examples

```json
"aud": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
"iss": "did:key:zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:pkh:eth:0xb9c5714089478a327f09197987f16f9e5d936e8a",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:ion:test:EiANCLg1uCmxUR4IUkpW8Y5_nuuXLbAEwonQd4q8pflTnw#key-1",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:3:bafyreiffkeeq4wq2htejqla2is5ognligi4lvjhwrpqpl2kazjdoecmugi#yh27jTt7Ny2Pwdy",
```

### 3.2.2 Time Bounds

`nbf` and `exp` stand for "not before" and "expires at," respectively. These are standard fields from [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) (JWT). Taken together, they represent the time bounds for a token.

The `nbf` field is OPTIONAL. When omitted, the token MUST be treated as valid beginning from the Unix epoch. Setting the `nbf` field to a time in the future MUST delay using a UCAN. For example, pre-provisioning access to conference materials ahead of time but not allowing access until the day it starts is achievable with judicious use of `nbf`.

The `exp` field MUST be set. If the time is in the past, the token MUST fail validation.

Keeping the window of validity as short as possible is RECOMMENDED. Limiting the time range can mitigate the risk of a malicious user abusing a UCAN. However, this is situationally dependent. It may be desirable to limit the frequency of forced reauthorizations for trusted devices. Due to clock drift, time bounds SHOULD NOT be considered exact. A buffer of ±60 seconds is RECOMMENDED.

#### Examples

```json
"nbf": 1529496683,
"exp": 1575606941,
```

### 3.2.3 Nonce

The OPTIONAL nonce parameter `nnc` MAY be any value. A randomly generated string is RECOMMENDED to provide a unique UCAN, though it MAY also be a monotonically increasing count of the number of links in the hash chain. This field helps prevent replay attacks and ensures a unique CID per delegation. The `iss`, `aud`, and `exp` fields together will often ensure that UCANs are unique, but adding the nonce ensures this uniqueness.

This field SHOULD NOT be used to sign arbitrary data, such as signature challenges. See the `fct` field for more.

#### Examples

``` json
"nnc": "1701-D"
```

### 3.2.4 Facts

The OPTIONAL `fct` field contains arbitrary facts and proofs of knowledge. The enclosed data MUST be self-evident and externally verifiable. It MAY include information such as hash preimages, server challenges, a Merkle proof, dictionary data, etc.

#### Examples

``` json
"fct": [
  {
    "challenge": "abcdef",
    "from": "example.com"
  },
  {
    "sha3_256": "B94D27B9934D3E08A52E52D7DA7DABFAC484EFE37A5380EE9088F7ACE2EFCDE9",
    "msg": "hello world"
  }
]
```

### 3.2.5 Attenuations

The attenuations (i.e. UCAN output, or "caveats") MUST be an array of heterogeneous capabilities (defined below). This array MAY be empty.

This array MUST contain some or none of the following:
1. A strict subset (attenuation) of the capability scope from the `prf` field
2. Capabilities composed from multiple proofs (see rights amplification)
3. Capabilities originated by the `iss` DID (i.e. by parenthood)

This array also includes time ranges and the proofs that start latest and end soonest from the lower and upper time bounds of the range.

The attenuation field MUST contain an array of JSON objects, which MAY be empty. A JSON capability MUST include the `with` and `can` fields, and MAY have additional fields needed to describe the capability.

``` json
{
  "with": $RESOURCE,
  "can": $ABILITY,
  "nb": $EXTENSION
}
```

#### 3.2.5.1 Resource Pointer

A resource describes the noun of a capability. The resource pointer MUST be provided in [URI](https://datatracker.ietf.org/doc/html/rfc3986) format. Arbitrary and custom URIs MAY be used, provided that the intended recipient can decode the URI. The URI is merely a unique identifier to describe the pointer to — and within — a resource.

The same resource MAY be addressed with several URI formats. For instance, a database may have an address at the level of direct memory with `file`, via `sqldb` to gain access to SQL semantics, `http` to use web addressing, and `dnslink` to use Merkle DAGs inside DNS `TXT` records. 

| URI                                            | Meaning                                                         |
| ---------------------------------------------- | --------------------------------------------------------------- |
| `{"with": "mailto:username@example.com", ...}` | A single email address                                          |
| `{"with": "my:*", ...}`                        | All resources that the iss has access to, including future ones |
| `{"with": "my:dnslink", ...}`                  | All DNSLinks that the iss has access to, including future ones  |
| `{"with": "dnslink://myapp.example.com", ...}` | A mutable pointer to some data                                  |

#### 3.2.5.2 Ability

The `can` field is REQUIRED. It describes the verb portion of the capability: an action that can be performed on a resource. For instance, the standard HTTP methods such as `GET`, `PUT`, and `POST` would be possible `can` values for an `http` resource. While arbitrary semantics MAY be described, they MUST apply to the target resource. For instance, it does not make sense to apply `msg/SEND` to a typical file system. 

Abilities MAY be organized in a hierarchy with enums. A typical example is a superuser capability ("anything") on a file system. Another is read vs write access, such that in an HTTP context, `WRITE` implies `PUT`, `PATCH`, `DELETE`, and so on. Organizing potencies this way allows for adding more options over time in a backward-compatible manner, avoiding the need to reissue UCANs with new resource semantics.

Abilities MUST NOT be case sensitive. For example, `http/post`, `http/POST`, `HTTP/post`, `HTTP/POST`, and `hTtP/pOsT` MUST all mean the same ability.

There MUST be at least one path segment as a namespace. For example, `http/PUT` and `db/PUT` MUST be treated as unique from each other.

The only reserved ability MUST be the un-namespaced [`"*"` or "superuser"](#41-superuser), which MUST be allowed on any resource.

### 3.2.5.3 `nb` Non-Normative Fields

Capabilities MAY define additional optional or required fields specific to their use case in the `nb` ([nota bene](https://en.wikipedia.org/wiki/Nota_bene)) field. This field is OPTIONAL in the general case, but MAY be REQUIRED by particular capability types that require this information to validate. The `nb` field MAY contain additional caveats or other important information related to specifying the capability, and MAY function as an "escape hatch" for when a use case is not fully captured by the `with` and `can` fields.

Further delegation of a capability with `nb` fields set MUST respect the `nb` fields. On validation, if a `nb` field is present, it MUST be checked. If a validator is not able to interpret the `nb` field, it MUST reject the capability. As such, any `nb` caveats from a proof MUST further attenuate the delegated capability.

#### 3.2.5.4 Examples

``` json
"att": [
  {
    "with": "exampleuri://example.com/public/photos/",
    "can": "crud/DELETE"
  },
  {
    "with": "exampleuri://example.com/private/84MZ7aqwKn7sNiMGsSbaxsEa6EPnQLoKYbXByxNBrCEr",
    "can": "wnfs/APPEND"
  },
  {
    "with": "exampleuri://example.com/public/photos/",
    "can": "crud/DELETE",
    "nb": {
      "matching": "/(?i)(\W|^)(baloney|darn|drat|fooey|gosh\sdarnit|heck)(\W|$)/"
    }
  },
  {
    "with": "mailto:username@example.com",
    "can": "msg/SEND"
  },
  {
    "with": "mailto:username@example.com",
    "can": "msg/READ",
    "nb": {
      "max_count": 5,
      "templates": ["newsletter", "marketing"]
    }
  }
]
```

### 3.2.6 Proof of Delegation

Attenuations MUST satisfy any of the following conditions to be considered valid:

1. Match to a proof in the `prf` array ([§3.2.6.1](#3261-prf-field))
2. Use the `my:` prefix to indicate parenthood ([§4.2.1](#421-my-delegation-by-parenthood))

Checked proofs MUST be resolvable by the recipient. A proof MAY be left unresolvable if it is not used as support for the top-level UCAN's capability chain. The exact format MUST be defined in the relevant transport specification. Some examples of possible formats include: a JSON object payload delivered with the UCAN, a federated HTTP endpoint, a DHT, or shared database.

#### 3.2.6.1 `prf` Field

The `prf` field MUST contain the content address ([§5.6](#56-content-identifiers)) of UCAN proofs (the "inputs" of a UCAN). 

#### 3.2.6.2 Examples

``` json
"prf": [
  "bafkreihogico5an3e2xy3fykalfwxxry7itbhfcgq6f47sif6d7w6uk2ze",
  "bafkreiemaanh3kxqchhcdx3yckeb3xvmboztptlgtmnu5jp63bvymxtlva"
]
```

Which in a JSON representation would resolve to the following table:

```json
{
  "bafkreiemaanh3kxqchhcdx3yckeb3xvmboztptlgtmnu5jp63bvymxtlva": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCIsInVjdiI6IjAuOC4xIn0.eyJhdWQiOiJkaWQ6a2V5Ono2TWtmUWhMSEJTRk11UjdiUVhUUWVxZTVrWVVXNTFIcGZaZWF5bWd5MXprUDJqTSIsImF0dCI6W3sid2l0aCI6eyJzY2hlbWUiOiJ3bmZzIiwiaGllclBhcnQiOiIvL2RlbW91c2VyLmZpc3Npb24ubmFtZS9wdWJsaWMvcGhvdG9zLyJ9LCJjYW4iOnsibmFtZXNwYWNlIjoid25mcyIsInNlZ21lbnRzIjpbIk9WRVJXUklURSJdfX0seyJ3aXRoIjp7InNjaGVtZSI6InduZnMiLCJoaWVyUGFydCI6Ii8vZGVtb3VzZXIuZmlzc2lvbi5uYW1lL3B1YmxpYy9ub3Rlcy8ifSwiY2FuIjp7Im5hbWVzcGFjZSI6InduZnMiLCJzZWdtZW50cyI6WyJPVkVSV1JJVEUiXX19XSwiZXhwIjo5MjU2OTM5NTA1LCJpc3MiOiJkaWQ6a2V5Ono2TWtyNWFlZmluMUR6akc3TUJKM25zRkNzbnZIS0V2VGIyQzRZQUp3Ynh0MWpGUyIsInByZiI6WyJleUpoYkdjaU9pSkZaRVJUUVNJc0luUjVjQ0k2SWtwWFZDSXNJblZqZGlJNklqQXVPQzR4SW4wLmV5SmhkV1FpT2lKa2FXUTZhMlY1T25vMlRXdHlOV0ZsWm1sdU1VUjZha2MzVFVKS00yNXpSa056Ym5aSVMwVjJWR0l5UXpSWlFVcDNZbmgwTVdwR1V5SXNJbUYwZENJNlczc2lkMmwwYUNJNmV5SnpZMmhsYldVaU9pSjNibVp6SWl3aWFHbGxjbEJoY25RaU9pSXZMMlJsYlc5MWMyVnlMbVpwYzNOcGIyNHVibUZ0WlM5d2RXSnNhV012Y0dodmRHOXpMeUo5TENKallXNGlPbnNpYm1GdFpYTndZV05sSWpvaWQyNW1jeUlzSW5ObFoyMWxiblJ6SWpwYklrOVdSVkpYVWtsVVJTSmRmWDFkTENKbGVIQWlPamt5TlRZNU16azFNRFVzSW1semN5STZJbVJwWkRwclpYazZlalpOYTJ0WGIzRTJVek4wY1ZKWGNXdFNibmxOWkZobWNuTTFORGxGWm5VMmNVTjFOSFZxUkdaTlkycEdVRXBTSWl3aWNISm1JanBiWFgwLlNqS2FIR18yQ2UwcGp1TkY1T0QtYjZqb04xU0lKTXBqS2pqbDRKRTYxX3VwT3J0dktvRFFTeFo3V2VZVkFJQVREbDhFbWNPS2o5T3FPU3cwVmc4VkNBIiwiZXlKaGJHY2lPaUpGWkVSVFFTSXNJblI1Y0NJNklrcFhWQ0lzSW5WamRpSTZJakF1T0M0eEluMC5leUpoZFdRaU9pSmthV1E2YTJWNU9ubzJUV3R5TldGbFptbHVNVVI2YWtjM1RVSktNMjV6UmtOemJuWklTMFYyVkdJeVF6UlpRVXAzWW5oME1XcEdVeUlzSW1GMGRDSTZXM3NpZDJsMGFDSTZleUp6WTJobGJXVWlPaUozYm1aeklpd2lhR2xsY2xCaGNuUWlPaUl2TDJSbGJXOTFjMlZ5TG1acGMzTnBiMjR1Ym1GdFpTOXdkV0pzYVdNdmNHaHZkRzl6THlKOUxDSmpZVzRpT25zaWJtRnRaWE53WVdObElqb2lkMjVtY3lJc0luTmxaMjFsYm5SeklqcGJJazlXUlZKWFVrbFVSU0pkZlgxZExDSmxlSEFpT2preU5UWTVNemsxTURVc0ltbHpjeUk2SW1ScFpEcHJaWGs2ZWpaTmEydFhiM0UyVXpOMGNWSlhjV3RTYm5sTlpGaG1jbk0xTkRsRlpuVTJjVU4xTkhWcVJHWk5ZMnBHVUVwU0lpd2ljSEptSWpwYlhYMC5TakthSEdfMkNlMHBqdU5GNU9ELWI2am9OMVNJSk1waktqamw0SkU2MV91cE9ydHZLb0RRU3haN1dlWVZBSUFURGw4RW1jT0tqOU9xT1N3MFZnOFZDQSJdfQ.Ab-xfYRoqYEHuo-252MKXDSiOZkLD-h1gHt8gKBP0AVdJZ6Jruv49TLZOvgWy9QkCpiwKUeGVbHodKcVx-azCQ",
  "bafkreihogico5an3e2xy3fykalfwxxry7itbhfcgq6f47sif6d7w6uk2ze": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsInVhdiI6IjAuMS4wIn0.eyJhdWQiOiJkaWQ6a2V5OnpTdEVacHpTTXRUdDlrMnZzemd2Q3dGNGZMUVFTeUExNVc1QVE0ejNBUjZCeDRlRko1Y3JKRmJ1R3hLbWJtYTQiLCJpc3MiOiJkaWQ6a2V5Ono1QzRmdVAyRERKQ2hoTUJDd0FrcFlVTXVKWmROV1dINU5lWWpVeVk4YnRZZnpEaDNhSHdUNXBpY0hyOVR0anEiLCJuYmYiOjE1ODg3MTM2MjIsImV4cCI6MTU4OTAwMDAwMCwic2NwIjoiLyIsInB0YyI6IkFQUEVORCIsInByZiI6bnVsbH0.Ay8C5ajYWHxtD8y0msla5IJ8VFffTHgVq448Hlr818JtNaTUzNIwFiuutEMECGTy69hV9Xu9bxGxTe0TpC7AzV34p0wSFax075mC3w9JYB8yqck_MEBg_dZ1xlJCfDve60AHseKPtbr2emp6hZVfTpQGZzusstimAxyYPrQUWv9wqTFmin0Ls-loAWamleUZoE1Tarlp_0h9SeV614RfRTC0e3x_VP9Ra_84JhJHZ7kiLf44TnyPl_9AbzuMdDwCvu-zXjd_jMlDyYcuwamJ15XqrgykLOm0WTREgr_sNLVciXBXd6EQ-Zh2L7hd38noJm1P_MIr9_EDRWAhoRLXPQ"
}
```

For more on this representation, please refer to [§6.1](#61-canonical-json-collection).

# 4. Reserved Capabilities

The following capabilities are REQUIRED to be implemented.

## 4.1 Superuser

The superuser action MUST be denoted `*`. This is the maximum ability and may be applied to any resource (it is the "top" ability).

This is useful in several cases, for example:

1. When delegating all resources, like in a [`my` scheme](#42-owned-resources)
2. To grant the maximum ability when the current ability semantics may be extended later

## 4.2 Owned Resources

The `my` URI scheme represents ownership over a resource — typically by parenthood — at decision-time (i.e. the validator's "now"). Resources that are created after the UCAN was created MUST be included. This higher-order scheme describes delegating some or all ambient authority to another DID.

The use case of "pairing" two DIDs by delegating all current and future resources is not uncommon when a user would like to use multiple devices as "root" but does not have access to all of them directly at all times. An everyday use case for this is a user signing into multiple devices, using them both with full rights.

The format for this scheme is as follows:

``` abnf
ownershipscheme = "my:" target
target = "*" / <scheme> / <uri>
```

### 4.2.1 `my` Delegation By Parenthood

Capabilities being delegated by parenthood (the "apex" delegation) MUST be prefixed with `my:`. Including this prefix disambiguates that the source of this capability is the current issuer's owned resources.

Without this prefix, the ambiguity in provenance can be exploited by a malicious user in an unrelated proof, forcing the validly delegated capability to appear as though it has an invalid ownership claim. The `my:` prefix resolved this issue by making introduction of a capability by parenthood explicit.

#### 4.2.1.1 Example

```js
// By parenthood
{
  "with": "my:email:alice@example.com",
  "can": "msg/send"
}

// The same resource, further delegated
{
  "with": "email:alice@example.com",
  "can": "msg/send"
}
```

### 4.2.2 `my` Wildcard

The wildcard `my:*` resource MUST be taken to mean "everything" (all resources of all types) that are owned by the current DID.

### 4.2.3 Subschemes

A "sub-scheme" MAY be used to delegate some of that scheme controlled by parenthood. For example, `my:dns` delegates access to all DNS records. `my:mailto` selects all owned email addresses controlled by this user.

### 4.2.4 `as:` Redelegation

Redelegating these to further DIDs in a chain MUST use the `as` URI and address the specific parent DID that owns that resource, followed by the resource kind selector. For instance: `as:did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp:*` selects all resources originating from the specified DID, and `as:did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp:mailto` selects email addresses from the DID. 

``` abnf
delegatescheme = "as:" did ":" kind
kind = "*" / <scheme>
```

### 4.2.5 Action

The action for `my:*` or `as:<did>:*` MUST be the [superuser action `*`](#41-superuser). Another ability would not be possible since any other ability cannot be guaranteed to work across all resource types (e.g. it's not possible to `crud/UPDATE` an email address). Recall that the superuser action is special in that it selects the maximum possible action for any resource.

``` json
{"with": "my:*", "can": "*"}
{"with": "as:did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp:*", "can": "*"}

{"with": "my:mailto", "can": "msg/SEND"}
{"with": "as:did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp:mailto", "can": "msg/SEND"}
```

For `my` and `as` capabilities limited to some scheme, the action MUST be one normally associated with that resource. As it belongs to every action hierarchy, this MAY be the [superuser action `*`](#41-superuser).

``` json
{"with": "my:dns", "can": "crud/UPDATE"}
{"with": "my:dns", "can": "*"}
```

## 4.3 UCAN Redelegation

### 4.3.1 `ucan` Scheme

The `ucan` URI scheme defines addressing for UCANs.

``` abnf
ucan = "ucan:" selector
selector = "*" / cid
```

`ucan:*` represents all of the UCANs in the current proofs array. If selecting a particular proof (i.e. not the wildcard), then the [CID](#56-content-identifiers) MUST be used. In the case of selecting a particular proof, the validator MUST check that the delegated content address is listed in the proofs (`prf`) field.

### 4.3.2 `ucan` Actions

The `ucan` scheme MUST accept the following action: `ucan/DELEGATE`. This action redelegates all of the capabilities in the selected proof(s).

`ucan/delegate` is distinct from the superuser ability and acts as a re-export of the selected abilities. If an attenuated resource or capability is desired, it MUST be explicitly listed without the `ucan` URI scheme.

``` js
[
  { 
    "with": "ucan:bafkreihogico5an3e2xy3fykalfwxxry7itbhfcgq6f47sif6d7w6uk2ze",
    "can": "ucan/DELEGATE"
  }, 
  { 
    "with": "ucan:*", 
    "can": "ucan/DELEGATE" 
  }
]
```

# 5. Validation

Each capability has its own semantics, which needs to be interpretable by the target resource handler. Therefore, a validator SHOULD NOT reject UCANs with resources that it does not know how to interpret.

If any of the following criteria are not met, the UCAN MUST be considered invalid.

## 5.1 Time

A UCAN's time bounds MUST NOT be considered valid if the current system time is before the `nbf` field or after the `exp` field. This is called "ambient time validity."

All proofs MUST contain time bounds equal to or broader than the UCAN being delegated. If the proof expires before the outer UCAN — or starts after it — the reader MUST treat the UCAN as invalid. Delegation inside of the time bound is called "timely delegation." These conditions MUST hold even if the current wall clock time is inside of incorrectly delegated bounds. 

A UCAN is valid inclusive from the `nbf` time and until the `exp` field. If the current time is outside of these bounds, the UCAN MUST be considered invalid. When setting these bounds, a delegator or invoker SHOULD account for expected clock drift. Use of time bounds this way is called "timely invocation."

## 5.2 Principal Alignment

In delegation, the `aud` field of every proof MUST match the `iss` field of the outer UCAN (the one being delegated to). This alignment MUST form a chain back to the originating principal for each resource.

```
(Resource)                                                        ─┐
Storage         Root Iss == Owner?                                 ├─ Agents
Owner: Alice ◄─────────────────────── Discharger: StorageService  ─┘
         ▲                                            ▲
         │                                            │           ─┐
┌────────┼────┐                                       │            │
│        │    │  ┌────────────────┐                   │            │
│ iss: Alice  │  │                │  ┌────────────────┼────────┐   │
│ aud: Bob ◄──┼──┼── iss: Bob     │  │                │        │   │
│             │  │   aud: Carol ◄─┼──┼─── iss: Carol  │        │   ├─ UCANs
└─────────────┘  │                │  │    aud: StorageService  │   │
  Root UCAN      └────────────────┘  │                         │   │
                   Delegate UCAN     └─────────────────────────┘   │
                                           Invocation UCAN        ─┘
```

In the above diagram, Alice has some storage. This storage may exist in one location with a single source of truth, but to help build intuition this example is location independent: local versions and remote stored copied are eventually consistent, and there is no one "correct" copy. As such, we list the owner (Alice) directly on the resource.

Alice delegates access to Bob. Bob then redelegates to Carol. Carol invokes the UCAN as part of a REST request to a storage service. To do this, she MUST both provide proof that she has access (the UCAN chain), and MUST delegate access to the discharging storage service. The discharging service MUST then check that the root issuer (Alice) is in fact the owner (typically the creator) of the resource. This MAY be listed directly on the resource, as it is here. Once the UCAN chain and root ownership are validated, the storage service performs the write.

### 5.2.1 Recipient Validation

An agent executing a capability MUST verify that the outermost `aud` field _matches its own DID._ The associated action MUST NOT be performed if they do not match. Recipient validation is REQUIRED to prevent the misuse of UCANs in an unintended context.

The following UCAN fragment would be valid to invoke as `did:key:zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV`. Any other agent MUST NOT accept this UCAN. For example, `did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` MUST NOT run the action associated with that capability.

``` js
{
  "aud": "did:key:zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV",
  "iss": "did:key:zAKJP3f7BD6W4iWEQ9jwndVTCBq8ua2Utt8EEjJ6Vxsf",
  // ...
}
```

A good litmus test for invocation validity by a discharging agent is to check if they would be able to create a valid delegation for that capability.

### 5.2.2 Token Uniqueness

Each remote invocation MUST be a unique UCAN: for instance using a nonce (`nnc`) or simply a unique expiry. The recipient MUST validate that they have not received the top-level UCAN before. For implementation recommentations, please refer to [§8.3](#83-replay-attack-prevention).

## 5.3 Proof Chaining

Each capability MUST either be originated by the issuer (root capability, or "parenthood") or have one-or-more proofs in the `prf` field to attest that this issuer is authorized to use that capability ("introduction"). In the introduction case, this check MUST be recursively applied to its proofs until a root proof is found (i.e. issued by the resource owner).

Except for rights amplification (below), each capability delegation MUST have equal or narrower capabilities from its proofs. The time bounds MUST also be equal to or contained inside the time bounds of the proof's time bounds. This lowering of rights at each delegation is called "attenuation."

## 5.4 Rights Amplification

Some capabilities are more than the sum of their parts. The canonical example is a can of soup and a can opener. You need both to access the soup inside the can, but the can opener may come from a completely separate source than the can of soup. Such semantics MAY be implemented in UCAN capabilities. This means that validating particular capabilities MAY require more than one direct proof. The relevant proofs MAY be of a different resource and action from the amplified capability. The delegated capability MUST have this behavior in its semantics, even if the proofs do not.

## 5.6 Content Identifiers

A UCAN token MUST be referenced as a [base32](https://github.com/multiformats/multibase/blob/master/multibase.csv#L12) [CIDv1](https://docs.ipfs.io/concepts/content-addressing/#identifier-formats), using either the [`0x55` raw data](https://github.com/multiformats/multicodec/blob/master/table.csv#L39) or [`0x0129` `dag-json`](https://github.com/multiformats/multicodec/blob/master/table.csv#L104) multicodec. Note that if using `dag-json`, the UCAN MUST be formatted as `dag-json` prior to signing.

The resolution of these addresses is left to the implementation and end-user, and MAY (non-exclusively) includes the following: local store, a distributed hash table (DHT), gossip network, or RESTful service. Please refer to [§7](#7-token-resolution) for more.

## 5.7 Revocation

Any issuer of a UCAN MAY later revoke that UCAN or the capabilities that have been derived from it further downstream in a proof chain.

This mechanism is eventually consistent and SHOULD be considered the last line of defense against abuse. Proactive expiry via time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.

While some resources are centralized (e.g. access to a server), others are unbound from specific locations (e.g. a CRDT), in which case it will take longer for the revocation to propagate.

Every resource type SHOULD have a canonical location where its revocations are kept. This list is non-exclusive, and revocation messages MAY be gossiped between peers in a network to share this information more quickly.

It is RECOMMENDED that the canonical revocation store be kept as close to (or inside) the resource it is about as possible. For example, the WebNative File System maintains a Merkle tree of revoked CIDs at a well-known path. Another example is that a centralized server could have an endpoint that lists the revoked UCANs by CID.

Revocations MUST be irreversible. If the revocation was issued in error, a unique UCAN MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes revocation stores append-only and highly amenable to caching.

A revocation message MUST conform to the following format:

``` js
{
  "iss": did,
  "revoke": ucanCid,
  "challenge": sign(did.privateKey, `REVOKE:${ucanCid}`)
}
```

This format makes it easy to select the relevant UCAN, confirm that the issuer is in the proof chain for some or all capabilities, and validate that the revocation was signed by the same issuer.

Any other proofs in the selected UCAN not issued by the same DID as the revocation issuer MUST be treated as valid.

Revocations MAY be deleted once the UCAN that they reference expires or otherwise becomes invalid via its proactive mechanisms.

### 5.7.1 Example

```
               Root
        ┌────────────────┐          ─┐
        │                │           │
        │  iss: Alice    │           │
        │  aud: Bob      │           ├─ Alice can revoke
        │  att: [X,Y,Z]  │           │
        │                │           │
        └───┬────────┬───┘           │
            │        │               │
            │        │               │
            ▼        ▼               │
┌──────────────┐  ┌──────────────┐   │  ─┐
│              │  │              │   │   │
│  iss: Bob    │  │  iss: Bob    │   │   │
│  aud: Carol  │  │  aud: Dan    │   │   ├─ Bob can revoke
│  att: [X,Y]  │  │  att: [Y,Z]  │   │   │
│              │  │              │   │   │
└───────┬──────┘  └──┬───────────┘   │   │
        │            │               │   │
        │            │               │   │
        ▼            │               │   │
┌──────────────┐     │               │   │  ─┐
│              │     │               │   │   │
│  iss: Carol  │     │               │   │   │
│  aud: Dan    │     │               │   │   ├─ Carol can revoke
│  att: [X,Y]  │     │               │   │   │
│              │     │               │   │   │
└───────────┬──┘     │               │   │   │
            │        │               │   │   │
            │        │               │   │   │
            ▼        ▼               │   │   │
        ┌────────────────┐           │   │   │  ─┐
        │                │           │   │   │   │
        │  iss: Dan      │           │   │   │   │
        │  aud: Erin     │           │   │   │   ├─ Dan can revoke
        │  att: [X,Y,Z]  │           │   │   │   │
        │                │           │   │   │   │
        └────────────────┘          ─┘  ─┘  ─┘  ─┘
```

In this example, Alice MAY revoke any of the UCANs in the chain, Carol MAY revoke the bottom two, and so on. If the UCAN `Carol->Dan` is revoked by Alice, Bob, or Carol, then Erin will not have a valid chain for `X` since its proof is invalid. However, Erin can still prove the valid capability for `Y` and `Z` since the still-valid ("unbroken") chain `Alice->Bob->Dan->Erin` includes them. Note that despite `Y` being in the revoked `Carol->Dan` UCAN, it does not invalidate `Y` for Erin, since the unbroken chain also included a proof for `Y`. 

## 5.8 Backwards Compatibility

A UCAN validator MAY implement backward compatibility with previous versions of UCAN. Delegated UCANs MUST be of an equal or higher version than their proofs. For example, a v0.9.0 UCAN that includes proofs that are separately v0.9.0, v0.8.1, v0.7.0, and v0.5.0 MAY be considered valid. A v0.5.0 UCAN that has a UCAN v0.9.0 proof MUST NOT be considered valid.

# 6. Collections

UCANs are indexed by their hash — often called their ["content address"](https://en.wikipedia.org/wiki/Content-addressable_storage). UCANs MUST be addressable as [CIDv1](https://specs.ipld.io/block-layer/CID.html#cids-version-1).

Content addressing the proofs has multiple advantages over inlining tokens, including:
* Avoids re-encoding deeply nested proofs as Base64 many times (and the associated size increase)
* Canonical signature
* Enables only transmitting the relevant proofs 

Multiple UCANs in a single request MAY be collected into one table. It is RECOMMENDED that these be indexed by CID. The canonical JSON representation is presented in [§6.1.1](#611-example) MUST be supported. Implementations MAY include more formats, for example to optimize for a particular transport. Transports MAY map their collection to this collection format.

### 6.1 Canonical JSON Collection

The canonical JSON representation is an key-value object, mapping UCAN content identifiers to their fully-encoded base64url strings. A root "entry point" (if one exists) MUST be indexed by the slash `/` character.

#### 6.1.1 Example

``` json
{
  "/": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCIsInVjdiI6IjAuOC4xIn0.eyJhdWQiOiJkaWQ6a2V5Ono2TWtmUWhMSEJTRk11UjdiUVhUUWVxZTVrWVVXNTFIcGZaZWF5bWd5MXprUDJqTSIsImF0dCI6W3sid2l0aCI6eyJzY2hlbWUiOiJ3bmZzIiwiaGllclBhcnQiOiIvL2RlbW91c2VyLmZpc3Npb24ubmFtZS9wdWJsaWMvcGhvdG9zLyJ9LCJjYW4iOnsibmFtZXNwYWNlIjoid25mcyIsInNlZ21lbnRzIjpbIk9WRVJXUklURSJdfX0seyJ3aXRoIjp7InNjaGVtZSI6InduZnMiLCJoaWVyUGFydCI6Ii8vZGVtb3VzZXIuZmlzc2lvbi5uYW1lL3B1YmxpYy9ub3Rlcy8ifSwiY2FuIjp7Im5hbWVzcGFjZSI6InduZnMiLCJzZWdtZW50cyI6WyJPVkVSV1JJVEUiXX19XSwiZXhwIjo5MjU2OTM5NTA1LCJpc3MiOiJkaWQ6a2V5Ono2TWtyNWFlZmluMUR6akc3TUJKM25zRkNzbnZIS0V2VGIyQzRZQUp3Ynh0MWpGUyIsInByZiI6WyJleUpoYkdjaU9pSkZaRVJUUVNJc0luUjVjQ0k2SWtwWFZDSXNJblZqZGlJNklqQXVPQzR4SW4wLmV5SmhkV1FpT2lKa2FXUTZhMlY1T25vMlRXdHlOV0ZsWm1sdU1VUjZha2MzVFVKS00yNXpSa056Ym5aSVMwVjJWR0l5UXpSWlFVcDNZbmgwTVdwR1V5SXNJbUYwZENJNlczc2lkMmwwYUNJNmV5SnpZMmhsYldVaU9pSjNibVp6SWl3aWFHbGxjbEJoY25RaU9pSXZMMlJsYlc5MWMyVnlMbVpwYzNOcGIyNHVibUZ0WlM5d2RXSnNhV012Y0dodmRHOXpMeUo5TENKallXNGlPbnNpYm1GdFpYTndZV05sSWpvaWQyNW1jeUlzSW5ObFoyMWxiblJ6SWpwYklrOVdSVkpYVWtsVVJTSmRmWDFkTENKbGVIQWlPamt5TlRZNU16azFNRFVzSW1semN5STZJbVJwWkRwclpYazZlalpOYTJ0WGIzRTJVek4wY1ZKWGNXdFNibmxOWkZobWNuTTFORGxGWm5VMmNVTjFOSFZxUkdaTlkycEdVRXBTSWl3aWNISm1JanBiWFgwLlNqS2FIR18yQ2UwcGp1TkY1T0QtYjZqb04xU0lKTXBqS2pqbDRKRTYxX3VwT3J0dktvRFFTeFo3V2VZVkFJQVREbDhFbWNPS2o5T3FPU3cwVmc4VkNBIiwiZXlKaGJHY2lPaUpGWkVSVFFTSXNJblI1Y0NJNklrcFhWQ0lzSW5WamRpSTZJakF1T0M0eEluMC5leUpoZFdRaU9pSmthV1E2YTJWNU9ubzJUV3R5TldGbFptbHVNVVI2YWtjM1RVSktNMjV6UmtOemJuWklTMFYyVkdJeVF6UlpRVXAzWW5oME1XcEdVeUlzSW1GMGRDSTZXM3NpZDJsMGFDSTZleUp6WTJobGJXVWlPaUozYm1aeklpd2lhR2xsY2xCaGNuUWlPaUl2TDJSbGJXOTFjMlZ5TG1acGMzTnBiMjR1Ym1GdFpTOXdkV0pzYVdNdmNHaHZkRzl6THlKOUxDSmpZVzRpT25zaWJtRnRaWE53WVdObElqb2lkMjVtY3lJc0luTmxaMjFsYm5SeklqcGJJazlXUlZKWFVrbFVSU0pkZlgxZExDSmxlSEFpT2preU5UWTVNemsxTURVc0ltbHpjeUk2SW1ScFpEcHJaWGs2ZWpaTmEydFhiM0UyVXpOMGNWSlhjV3RTYm5sTlpGaG1jbk0xTkRsRlpuVTJjVU4xTkhWcVJHWk5ZMnBHVUVwU0lpd2ljSEptSWpwYlhYMC5TakthSEdfMkNlMHBqdU5GNU9ELWI2am9OMVNJSk1waktqamw0SkU2MV91cE9ydHZLb0RRU3haN1dlWVZBSUFURGw4RW1jT0tqOU9xT1N3MFZnOFZDQSJdfQ.Ab-xfYRoqYEHuo-252MKXDSiOZkLD-h1gHt8gKBP0AVdJZ6Jruv49TLZOvgWy9QkCpiwKUeGVbHodKcVx-azCQ",
  "bafkreihogico5an3e2xy3fykalfwxxry7itbhfcgq6f47sif6d7w6uk2ze": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsInVhdiI6IjAuMS4wIn0.eyJhdWQiOiJkaWQ6a2V5OnpTdEVacHpTTXRUdDlrMnZzemd2Q3dGNGZMUVFTeUExNVc1QVE0ejNBUjZCeDRlRko1Y3JKRmJ1R3hLbWJtYTQiLCJpc3MiOiJkaWQ6a2V5Ono1QzRmdVAyRERKQ2hoTUJDd0FrcFlVTXVKWmROV1dINU5lWWpVeVk4YnRZZnpEaDNhSHdUNXBpY0hyOVR0anEiLCJuYmYiOjE1ODg3MTM2MjIsImV4cCI6MTU4OTAwMDAwMCwic2NwIjoiLyIsInB0YyI6IkFQUEVORCIsInByZiI6bnVsbH0.Ay8C5ajYWHxtD8y0msla5IJ8VFffTHgVq448Hlr818JtNaTUzNIwFiuutEMECGTy69hV9Xu9bxGxTe0TpC7AzV34p0wSFax075mC3w9JYB8yqck_MEBg_dZ1xlJCfDve60AHseKPtbr2emp6hZVfTpQGZzusstimAxyYPrQUWv9wqTFmin0Ls-loAWamleUZoE1Tarlp_0h9SeV614RfRTC0e3x_VP9Ra_84JhJHZ7kiLf44TnyPl_9AbzuMdDwCvu-zXjd_jMlDyYcuwamJ15XqrgykLOm0WTREgr_sNLVciXBXd6EQ-Zh2L7hd38noJm1P_MIr9_EDRWAhoRLXPQ"
}
```

# 7. Token Resolution

Token resolution is transport specific. The exact format is left to the relevant UCAN transport specification. At minimum, such a specification MUST define at least the following:

1. Request protocol
2. Response protocol
3. Collections format

# 8. Implementation Recommendations

## 8.1 UCAN Store

A validator MAY keep a local store of UCANs that it has received. UCANs are immutable but also time-bound so that this store MAY evict expired or revoked UCANs.

This store MAY be indexed by CID (content addressing). Multiple indices built on top of this store MAY be used to improve capability search or selection performance.

## 8.2 Memoized Validation

Aside from revocation, capability validation is idempotent. Marking a CID (or capability index inside that CID) as valid acts as memoization, obviating the need to check the entire structure on every validation. This extends to distinct UCANs that share a proof: if the proof was previously reviewed and is not revoked, it is RECOMMENDED to consider it valid immediately.

Revocation is irreversible. Suppose the validator learns of revocation by UCAN CID or issuer DID. In that case, the UCAN and all of its derivatives in such a cache MUST be marked as invalid, and all validations immediately fail without needing to walk the entire structure.

## 8.3 Replay Attack Prevention

Replay attack prevention is REQUIRED (per [§5.2.2 Token Uniqueness](#522-token-uniqueness)). The exact strategy is left to the implementer. One simple strategy is maintaining a set of previously seen CIDs. This MAY be the same structure as a validated UCAN memoization table (if one exists in the implementation).

It is RECOMMENDED that the structure have a secondary index referencing the token expiry field. This enables garbage collection and more efficient search. In cases of very large stores, normal cache performance techniques MAY be used, such as Bloom filters, multi-level caches, and so on.

## 8.4 Session Content ID

If many invocations are discharged during a session, the sender and receiver MAY agree to use the triple of CID, nonce, and signature rather than reissuing the complete UCAN chain for every message. This saves bandwidth and avoids needing to use another session token exchange mechanism or bearer token with lower security, such as a shared secret.

```js
{ 
  "cid": cid(ucan)
  "nnc": "ABC", 
  "sig": sign(ucan.iss.privateKey, cid(ucan) + "ABC") 
}
```

# 9. Related Work and Prior Art

[SPKI/SDSI](https://datatracker.ietf.org/wg/spki/about/) is closely related to UCAN. A different format is used, and some details vary (such as a delegation-locking bit), but the core idea and general usage pattern are very close. UCAN can be seen as making these ideas more palatable to a modern audience and adding a few features such as content IDs that were less widespread at the time SPKI/SDSI were written.

[ZCAP-LD](https://w3c-ccg.github.io/zcap-ld/) is closely related to UCAN, and the projects were started around the same time. The primary differences are in formatting, addressing by URL, and an emphasis on linked data rather than inlining for availability.

[CACAO](https://blog.ceramic.network/capability-based-data-security-on-ceramic/) is a translation of many of these ideas to a cross-blockchain invocation model. It contains the same basic concepts but is aimed at small messages and identities that are rooted in mutable documents rooted on a blockchain and lacks the ability to subdelegate capabilities.

[Local-First Auth](https://github.com/local-first-web/auth) uses CRDT-based ACLs and key lockboxes for role-based signatures. This is a non-certificate-based approach, instead of relying on the CRDT and signed data to build up a list of roles and members. It does have a very friendly invitation certificate mechanism in [Seitan token exchange](https://book.keybase.io/docs/teams/seitan). It is also straightforward to see which users have access to what, avoiding the confinement problem seen in many decentralized auth systems.

[Macaroon](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf) is a MAC-based capability and cookie system aimed at distributing authority across services in a trusted network (typically in the context of a Cloud). By not relying on asymmetric signatures, Macaroons achieve excellent space savings and performance, given that the MAC can be checked against the relevant services during discharge. The authority is rooted in an originating server rather than with an end-user.

[Biscuit](https://github.com/biscuit-auth/biscuit/) uses Datalog to describe capabilities. It has a specialized format but is otherwise in line with UCAN.

[Verifiable credentials](https://www.w3.org/2017/vc/WG/) are a solution for data about people or organizations. However, they are aimed at a slightly different problem: asserting attributes about the holder of a DID, including things like work history, age, and membership.

# 10. Acknowledgments

Thank you to [Brendan O'Brien](https://github.com/b5) for real-world feedback, technical collaboration, and implementing the first Golang UCAN library.

Thank you [Blaine Cook](https://github.com/blaine) for the real-world feedback, ideas on future features, and lessons from other auth standards.

Thank you [Dan Finlay](https://github.com/danfinlay) for being sufficiently passionate about OCAP that we realized that capability systems had a real chance of adoption in an ACL-dominated world.

Thanks to the entire [SPKI WG](https://datatracker.ietf.org/wg/spki/about/) for their closely related pioneering work.

We want to especially recognize [Mark Miller](https://github.com/erights) for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

# 11. FAQ

## 11.1 What prevents an unauthorized party from using an intercepted UCAN?

UCANs always contain information about the sender and receiver. A UCAN is signed by the sender (the `iss` field DID) and can only be created by an agent in possession of the relevant private key. The recipient (the `aud` field DID) is required to check that the field matches their DID. These two checks together secure the certificate against use by an unauthorized party.

## 11.2 What prevents replay attacks on the invocation use case?

A UCAN delegated for purposes of immediate invocation MUST be unique. If many requests are to be made in quick succession, a nonce can be used. The receiving agent (the one to perform the invocation) checks the hash of the UCAN against a local store of unexpired UCAN hashes.

This is not a concern when simply delegating since presumably the recipient agent already has that UCAN.

## 11.3 Is UCAN secure against person-in-the-middle attacks?

_UCAN does not have any special protection against person-in-the-middle (PITM) attacks._

Were a PITM attack successfully performed on a UCAN delegation, the proof chain would contain the attacker's DID(s). It is possible to detect this scenario and revoke the relevant UCAN but does require special inspection of the topmost `iss` field to check if it is the expected DID. Therefore, it is strongly RECOMMENDED to only delegate UCANs to agents that are both trusted and authenticated and over secure channels.
