# CAR Mirror over HTTP

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# 0. Abstract

CAR Mirror is transport agnostic, and MAY be implemented with a RESTful HTTP client/server. This specification describes the protocol for CAR Mirror over HTTP.

## 0.1 Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1. Pull

## 1.1 Endpoint

The endpoint MAY be placed at any route, but it is RECOMMENDED to be exposed at:

```http
POST /api/v0/dag/pull?stream=${bool}
```

Note the absence of a field for the requested URL. Since multiple CID roots MAY be requested at once, this information is instead located in the Requestor Payload.

#### 1.1.1 `stream` Parameter

This field specifies if the Provider should use discrete or streaming. `stream` MUST default to `false`. If streaming is set to `true`, the response MUST be a streaming CAR file and transmitted over HLS.

## 1.2 Requestor Payload

The requestor payload MUST be serialized as [CBOR](https://cbor.io/). This schema is given in [CDDL (RFC8601)](https://datatracker.ietf.org/doc/html/rfc8610).

```cddl
payload = {
  rs: [* cid], ; Requested CID roots
  bk: uint,    ; Bloom filter hash count
  bb: bytes,   ; Bloom filter Binary
}
```

## 1.3 Provider Payload

The response MUST be given as a [CARv1](https://ipld.io/specs/transport/car/carv1/). If the streaming flag was set in the URL, then the CAR MUST be a [streaming CARv1](https://ipld.io/specs/transport/car/carv1/#performance).

## 1.4 Status Codes

Status codes are [as defined in RFC2616 §10](https://www.rfc-editor.org/rfc/rfc2616#section-10), with no additional special meaning. For example, the common cases of success and lack of further CID roots would be:

* Success: `200`
* Unable to find any new root CIDs: `404`

# 2 Push

## 2.1 Endpoint

The endpoint MAY be placed at any route, but it is RECOMMENDED to be exposed at:

```http
POST /api/v0/dag/push?stream={bool}&diff={ipns | dnslink | cid}
```

### 2.1.1 `stream` Parameter

This field specifies if the Provider should use discrete or streaming. `stream` MUST default to `false`. If streaming is set to `true`, the response MUST be a streaming CAR file and transmitted over HLS.

### 2.1.2 `diff` Parameter

The `diff` field is OPTIONAL. It represents a related CID to the one being pushed, and MAY be an IPNS record, DNSLink, or CID. The complete URI MUST be provided.

While a mutable pointer (IPNS or DNSLink) MAY be resolved to a CID before the request is constructed, it is RECOMMENDED that the pointer be given when available. This strategy gives the Provider more flexibility for tracking the last value that they've seen for that pointer, and conveys more about the Requestor's intention.

This field is primarily useful for the [narrowing step](#312-narrowing), and especially during a [cold call](#3131-cold-calls). Since the Requestor does not have knowledge of everything in the Provider's store, this field provides a hint for the Provider of what (finite) context to include as part of the Bloom in their response.

This field MUST NOT be interpreted as a CID root being sent.

## 2.2 Requestor Payload

```cddl
payload = {
  bk : uint,  ; Bloom filter hash count
  bb : bytes, ; Bloom filter Binary
  pl : car,   ; Data payload
}
```

The data payload MUST be given as a [CARv1](https://ipld.io/specs/transport/car/carv1/). If the streaming flag was set in the URL, then the CAR MUST be a [streaming CARv1](https://ipld.io/specs/transport/car/carv1/#performance).

## 2.3 Provider Payload

The requestor payload MUST be serialized as [CBOR](https://cbor.io/). This schema is given in [CDDL (RFC8601)](https://datatracker.ietf.org/doc/html/rfc8610).

```cddl
payload = {
  sr : [* cid], ; Incomplete subgraph roots
  bk : uint,    ; Bloom filter hash count
  bb : bytes,   ; Bloom filter Binary
}
```

## 2.4 Provider Status Codes

Status codes are [as defined in RFC2616 §10](https://www.rfc-editor.org/rfc/rfc2616#section-10), with no additional special meaning. For example, a few common cases would include:

* Complete: `200`
* Success: `202`
