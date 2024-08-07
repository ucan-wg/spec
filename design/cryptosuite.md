# Cryptosuite

## Signature Algorithms

In an ideal world, a single algorithm (Ed25519) would be available everywhere. For many historical reasons, P-256 and `secp256k1` are the only practical options for many applications, such as the [WebCrypto API] and many cryptocurrency wallets. A design goal of UCAN is to leverage existing public key infrastructure (PKI), and supporting these three algorithms achieves that goal. Requiring multiple signature types for broad interoperability is not an unusual design choice; for example, [JWT] supports multiple signature algorithms for a similar reason.

The NIST report on post-quantum computing will also be released in the next few months, so we anticipate a shift to new algorithms in the medium-term.

## SHA-256

While [BLAKE3] is faster in software, the SHA2 family is already a requirement of the signature algorithms in UCAN's cryptosuite. SHA-256 is also available effectively everywhere, and doesn't need to shipped as a separate package, which lowers implementation/maintenance complexity, and bundle size.

<!-- External Links -->

[BLAKE3]: https://github.com/BLAKE3-team/BLAKE3
[JWT]: https://www.rfc-editor.org/rfc/rfc7519
[WebCrypto API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API
