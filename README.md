# My Crib Notes

A collection of visual and written crib notes. Each topic pairs a cartoon-style overview with a concise technical reference.

## Key-Based Container Image Signing

Key-based container image signing uses a publisher’s private key to sign a digest that identifies the exact image. The resulting signature is stored separately from the image in the registry and associated with that digest. Before deployment, a verifier checks the signature with a trusted matching public key and confirms that the digest corresponds to the image being considered; only images that pass
verification and the applicable deployment policy proceed.

- [Cartoon (PDF)](notes/key-based-container-image-signing.pdf)
- [Written notes (Markdown)](notes/key-based-container-image-signing.md)

## Mutual TLS Authentication (mTLS)

Mutual TLS extends TLS by having both the client and server authenticate each other with certificates and prove control of their corresponding private keys. TLS protects the connection, while authorization policy separately determines what each authenticated identity may access.

- [Cartoon (PDF)](notes/mutual-tls-authentication.pdf)
- [Written notes (Markdown)](notes/mutual-tls-authentication.md)
