# CID Configuration

A single CID configuration makes canonicalization for signatures (and lookup) significantly simpler. The base encoding is set for when rendered for signatures, still is flexible for network lookup (e.g. with an IPFS gateway subdomain).

## SHA-256

While BLAKE3 is faster in software, the SHA2 family is already a requirement of the signature algorithms in UCAN's cryptosuite. SHA2 is also available effectively everywhere.

## `base58tc`

The choice of [`base58btc`] retains compatibility with common CID tools, and forces a canonical CID encoding. The other common base at time of writing is `base32`, which is popular due to its compatability with various web technologies such as subdomain handling in browsers. Unfortunately, `base32` is case-insensitive, which poses a problem for CID canonicalization since multiple CIDs map to the same underlying hash.
