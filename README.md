# UCAN 0.8 Spec

## Editors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)
* [Philipp Krüger](https://github.com/matheus23), [Fission](https://fission.codes)
* [Daniel Holmgren](https://github.com/dholms)

## Contributors 

# 0. Introduction

User Controlled Authorization Network (UCAN) is a secure, local-first, user-originated authorization and revocation scheme. It provides public-key verifable, delegatable, expressive, openly extensible [capabilities](https://en.wikipedia.org/wiki/Object-capability_model) by extending the familiar [JWT](https://datatracker.ietf.org/doc/html/rfc7519) structure. UCANs achieve public verifiability via chained certificates, and [decentralized identifiers (DIDs)](https://www.w3.org/TR/did-core/). Verifyable chain compression is specified via [content addressing](https://en.wikipedia.org/wiki/Content-addressable_storage). UCAN improves on the familiaity and adoptability of similar schemes, like [SPKI/SDSI](https://theworld.com/~cme/html/spki.html). UCAN inverts the [Macaroon](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf) model by allowing creation and discharge by any agent with a DID, including peer-to-peer beyond traditional cloud computing.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1. Motivation



NOTE TO SELF: who cares about the below? What is the problem that you're solving?!

Combine capabilities

Traditional client-server architectures store data in a [multitenant](https://en.wikipedia.org/wiki/Multitenancy) way. Since all data is kept together, _____. [Access control lists (ACLs)](https://en.wikipedia.org/wiki/Access-control_list) _______________. You can think of this as being a single horizontal volume per app, with vertical stripes; one for each user. ACLs are intended to keep those stripes separate from each other. Sometimes this code is broken, and users are accidentally given access to each others data.


* Nonexportable keys

* attenuation & contextual confinment (macaroon paper)

* Each UCAN includes a constructive assrtion of what it is allowed to do (note: not a predicate)


* ambient authority, confused deputy

User-controlled data is a 90° rotation of the above picture. Users are in control of their data: it's stored single tenant, with apps asking for stripes of data. Here, the user is in control, and apps ask for permission to see some portion of the user's data.


What if you're writing a collaborative application, and it needs access across user stores? Not a problem: users can grant access to their stores, and the caller can glue these rights together.

![](./overlapping_rights.jpg)

# 2. Terminology

## 2.1 Resource

## 2.2 Potency

The potency (also known as a "caveat") are the represented rights to some resource. Each potency type has its own elements and semantics. They may by unary, support a semilattice, be monotone, and so on. Potencies may be considered on their own — separate from resources — and applied to different resources. Potencies are extensible, and definiable to match any resource. For example:

APPEND is a potency for WNFS paths. The potency OVERWRITE also implies the ability to APPEND. Email has no such tiered relationship. You may SEND email, but there is no ”super send”.

## 2.3 Scope

An authorization scope is the tuple `resource x potency`. Scopes compose, so a list of scopes can be considered the union of all of the inner scopes.

![](./assets/scope_union.jpg)

You can think of this as ”scoping” the total rights of the authorization space down to the relevant volume of authorizations.

Inside this content space, can draw a boundary around some resource(s) (their type, identifiers, and paths or children), and their capabilities (potencies).

As a practical matter, since scopes form a group, you can be fairly loose: order doesn’t matter, and merging resources can be quite broad since the more powerful of any overlap will take precedence (i.e. you don’t need a clean separation).

## 2.4 Delegation

## 2.5 Attenuation

## 2.6 Rights Amplification

A user holding two scopes may have more rights than the sum of their parts. The [classic analogy](FIXME) is a can and can opener. Having access to both gives you the right to access the contents of the can, which you can't access with just the can or just the opener.

## 2.7 Revocation

* Subchain revocation
* DID revocation


# 4. Invocation Security



# 7. JWT Structure

UCANs MUST BE formatted as JWTs, with additional required and optional keys. The overall container of a header, claims, and siganture remain. Please refer to RFC 7519 for more omn this format

## 7.1 Header

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

## 7.2 Payload

The payload MUST describe the authorization claims being made, who is involved, and its validity period.

| Field | Type       | Description                            | Required |
|-------|------------|----------------------------------------|----------|
| `iss` | `String`   | Issuer DID (sender)                    | Yes      |
| `aud` | `String`   | Audience DID (receiver)                | Yes      |
| `nbf` | `Number`   | Not Before Unix Timestamp (valid from) | No       |
| `exp` | `Number`   | Expiratio Time (valid until)           | Yes      |
| `nnc` | `String`   | Nonce                                  | No       |
| `fct` | `Json[]`   | Facts (asserted, signed data)          | No       |
| `prf` | `String[]` | Proofs (nested UCANs)                  | Yes      |
| `att` | `Json[]`   | Attenuations                           | Yes      |

### 7.2.1 Principals

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

### 7.2.2 Time Bounds

`nbf` and `exp` stand for "not before" and "expires at" respectively. These are standard fields from RFC 7519 (JWT). Taken together they represent the time bounds for a token.

The `nbf` field is OPTIONAL. When omitted, the token MUST be treated as valid beginning from the Unix epoch. Setting the `nbf` field to a time in the future MUST delay use of a UCAN. For example, preprovisioning access to conference materials ahead of time, but not allowing access until the day of is achievable with judicious use of `nbf`.

The `exp` field MUST be set. If the time is in the past, the token MUST fail validation.

It is RECOMMENDED to keep the window of validity be as short as possible. By limiting the time range, the risk of a malicious user abusing a UCAN. This is situationally dependent, as trusted devices may not want to  reauthorize it very often. Due to clock drift, time bounds SHOULD NOT be considered exact. A buffer of ±60 seconds is RECOMMENDED.

#### Examples

```json
"nbf": 1529496683,
"exp": 1575606941,
```

### 7.2.3 Nonce

The OPTIONAL nonce parameter `nnc` MAY be any value. A randomly generated string is RECOMMENDED to provide a unique UCAN, though it MAY also be a monotonically increasing count of hash link. This field helps prevent replay attacks, and ensures a unique CID per delegation. The `iss`, `aud`, and `exp` fields together will often ensure that UCANs are unique, but adding the nonce ensures this uniqueness.

This field SHOULD NOT be used to sign arbitrary data, such as signature challenges. See the `fct` field for more.

#### Examples

``` json
"nnc": "1701-D"
```

### 7.2.4 Facts

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

### 7.2.5 Attenuation

The attenuated resources (i.e. output) of a UCAN is an array of heterogeneous resources and capabilities (defined below).

The union of this array must be a strict subset (attenuation) of the proofs plus resources created/owned/originated by the ”iss” DID. This scoping also includes time ranges, making the proof that starts latest and proof the end soonest the lower and upper time bounds.

Each capability has its own semantics, which need to be interpretable by the target resource handler. They consist of at least a resource and a capability, generally adhering to the form:

``` json
{
  "with": $URI,
  "can": $CAPABILITY
}
```

#### 7.2.5.1 Resources

URI

This merely a unique identifier to indicate the type of thing being described. For example, “wnfs“ for the WebNative File System, “email” for email, and “domain“ for domain names.


This value depends on the resource type. A resource identifier is a canonical reference of some sort. For instance, a DNSLink, email address, domain name, or username.

These values may also include the wildcard (*). This means ”any resource of this type”, even if not yet created, bounded by proofs. These are generally used for account linking. Wildcards are not required to delegate longer paths, as paths are generally taken as OR filters.

URI format -- https://datatracker.ietf.org/doc/html/rfc3986

| Value | Meaning |
|-------|---------|
| `"*"`      | Delegate all resources of any type that are in scope        |
| `{"with": "wnfs://user.example.com/path", ...}`      | File paths in our file system        |
| `{"with": "app://*", ...}` | All apps that the iss has access to, including future ones |
| `{"with": "app://myapp.fission.app", ...}` | A URL for the app (ideally the auto-assigned one) |

#### Examples

``` json
"att": [
  {
    "with": "wnfs://boris.fission.name/public/photos/",
    "can": "OVERWRITE"
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

### 7.2.6 Proofs

The `prf` field MUST contain UCAN proofs (the ”inputs” of a UCAN). As they need to be independently verifiable, proofs MUST either be the fully encoded version of a UCAN (including the signature), or the content address of the relevant proof. Attenuations not covered by a proof in the `prf` array MUST BE treated as owned by the issuer DID.

Proofs referenced by content address MUST be resolvable by the recipient, for instance over a DHT or database. Since they have no availability concern, inlined proofs SHOULD be preferred, unless the proof chain has become too large for a request.

#### Examples

``` json
"prf": ["eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsInVhdiI6IjAuMS4wIn0.eyJhdWQiOiJkaWQ6a2V5OnpTdEVacHpTTXRUdDlrMnZzemd2Q3dGNGZMUVFTeUExNVc1QVE0ejNBUjZCeDRlRko1Y3JKRmJ1R3hLbWJtYTQiLCJpc3MiOiJkaWQ6a2V5Ono1QzRmdVAyRERKQ2hoTUJDd0FrcFlVTXVKWmROV1dINU5lWWpVeVk4YnRZZnpEaDNhSHdUNXBpY0hyOVR0anEiLCJuYmYiOjE1ODg3MTM2MjIsImV4cCI6MTU4OTAwMDAwMCwic2NwIjoiLyIsInB0YyI6IkFQUEVORCIsInByZiI6bnVsbH0.Ay8C5ajYWHxtD8y0msla5IJ8VFffTHgVq448Hlr818JtNaTUzNIwFiuutEMECGTy69hV9Xu9bxGxTe0TpC7AzV34p0wSFax075mC3w9JYB8yqck_MEBg_dZ1xlJCfDve60AHseKPtbr2emp6hZVfTpQGZzusstimAxyYPrQUWv9wqTFmin0Ls-loAWamleUZoE1Tarlp_0h9SeV614RfRTC0e3x_VP9Ra_84JhJHZ7kiLf44TnyPl_9AbzuMdDwCvu-zXjd_jMlDyYcuwamJ15XqrgykLOm0WTREgr_sNLVciXBXd6EQ-Zh2L7hd38noJm1P_MIr9_EDRWAhoRLXPQ"]
```

# 8. Validation

## 8.X Time

## 8.x Pricipal Alignment

## 8.X Proof Chaining

## 8.X Rights Amplification

## 8.X Attenuations

## 9. Implementation Recommendations

### 9.X Generalized Store

### 9.X Content Addressing

### 9.X Memoized Validation

# 10. Acknowledgements

* Irakli Gozalishvili
* Brendan O'Brien
* Mark Miller
