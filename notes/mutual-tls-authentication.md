# Mutual TLS Authentication (mTLS)

TLS protects network communications and ordinarily allows a client to authenticate the server. **Mutual TLS (mTLS)** adds certificate-based client authentication, so both endpoints can authenticate each other. These identity checks help establish who is at each end of a connection; authorization policy separately determines what an identity may do.

This overview focuses on the concepts and overall authentication flow. It does not cover commands, resource setup, or detailed TLS handshake mechanics.

## Why authenticate both endpoints?

In conventional server-authenticated TLS, the client verifies the server, but the server does not necessarily authenticate the client with a certificate. In service-to-service communication, both endpoints may need to establish the identity of the peer before exchanging application data. mTLS provides certificate-based authentication in both directions.

**Key distinction:** Mutual authentication establishes peer identities; it does not by itself grant access to services or operations.

## Public and private key pairs

A public/private key pair consists of two mathematically related keys. The private key is kept secret and can create digital signatures. The corresponding public key can be shared and used to verify those signatures. An endpoint must not give its private key to a peer or a certificate authority (CA).

**Key distinction:** The public key is shareable; the private key remains under its owner's control.

## Proof of private-key possession is not identity

A proof-of-possession exchange can have a party sign fresh or session-specific data with its private key. A valid signature verified with the corresponding public key shows control of that key. It does not, by itself, establish the party's name or workload identity. A trusted association between the public key and an identity is also needed. Using fresh data helps prevent replay of an old signature.

**Key distinction:** A signature can prove control of a key; it does not independently prove who controls it.

## The role of a certificate authority

A certificate authority (CA) checks identity claims according to an enrollment policy and can attest to the association between an approved identity and a public key. The exact validation process depends on the system. Relying parties must decide which CAs and validation processes they trust.

**Key distinction:** A CA provides an identity-checking and attestation service; it does not create universal trust or grant access permissions.

## The CA's own key pair

A CA has its own key pair, separate from the key pair belonging to any endpoint whose certificate it issues. The CA uses its private key to sign certificate data. Verifiers use the corresponding CA public key, obtained through a trusted CA certificate or chain, to check that signature. An endpoint separately uses its own private key to prove control of the public key in its certificate.

**Key distinction:** The endpoint's public key is associated with the endpoint identity; the CA's public key is used to verify the CA's certificate signature.

## What a certificate says

An X.509 certificate is a signed data structure that binds an identity to a public key. It also carries information such as the issuer, validity period, serial number, and extensions. The CA signs the certificate data with its private key, and a verifier checks that signature using the issuer's public key. The CA's signature attests to the identity-to-key binding; it is not the endpoint's proof that it controls the corresponding private key.

**Key distinction:** A certificate contains an identity and public key plus an issuer's signature over the certificate data. The subject's key and the issuer's signing key usually have different roles.

## Root certificates and trust anchors

A root CA certificate contains the CA's own identity and public key. It is commonly self-signed: the CA signs its own certificate data with the private key corresponding to the public key in that certificate. The signature can be verified with that public key, but self-signing does not make the root trustworthy. A client or server trusts a root because it has intentionally configured or otherwise accepted that certificate as a trust anchor. In a hierarchy, certificates from intermediate CAs can chain back to a trusted root.

**Key distinction:** A root's self-signature is verifiable, but trust in the root comes from a separate decision to accept it as a trust anchor.

## Certificate signing requests

A certificate signing request (CSR) asks a CA to issue a certificate. It normally contains the requester's public key and requested identity information, and is signed with the corresponding private key to demonstrate control of that key. The private key itself is not sent. A CSR is only a request; it does not prove that the requester is entitled to the requested identity.

**Key distinction:** A CSR demonstrates control of the proposed key and requests an identity binding; it does not make the identity claim true.

## Certificate issuance

After validating a CSR and deciding that the requester may claim the requested identity, the CA issues a certificate binding that identity to the public key. The CA signs the certificate data with its own private key. The requester's CSR signature and the CA's certificate signature are separate: the first demonstrates control of the proposed key; the second attests to the CA-approved binding. The endpoint's private key is never sent to the CA.

**Key distinction:** The CSR is the signed request; the certificate is the CA-signed result.

## Why CA-based trust scales

Without a CA, peers would need a separate trusted-key distribution process for each peer they communicate with. With CA-based trust, a relying party can validate many issued certificates through a configured set of trusted CA roots and any required certificate chain. Trust stores may contain multiple roots; the model does not require one universal CA.

**Key distinction:** A CA reduces per-peer trust configuration, but relying parties still check identities and apply their own policies.

## Server identity and DNS names

A server certificate identifies the service and contains the server's public key. For a DNS connection, the certificate's Subject Alternative Name (SAN) should include the DNS name the client intended to reach. The client checks that name as well as certificate validity, intended use, and the chain to a trusted CA. Modern TLS clients use SAN for hostname matching rather than relying on the legacy Common Name (CN). The client's public key belongs in its separate client certificate, not in the server certificate.

**Key distinction:** The server certificate carries the server's public key and must identify the intended server name; it does not carry the client's key.

## Client identity

A client certificate identifies the calling client, service, or workload and contains that client's public key. The identity may be represented in different ways, such as a URI SAN for a workload identity; it need not be a DNS hostname. The server validates the certificate chain and checks the client identity against its own policy.

**Key distinction:** Server certificates commonly identify the destination DNS name; client certificates identify the caller according to the system's identity scheme.

## Mutual authentication and the protected connection

In mTLS, each side presents a certificate and demonstrates control of the private key corresponding to the public key in that certificate. The client validates the server's certificate and expected identity; the server validates the client's certificate and identity. TLS provides the protected communication channel. This high-level description does not depend on any particular handshake sequence.

**Key distinction:** Both peers authenticate each other, but each still evaluates the peer identity according to its own trust and validation policy.

## Authentication and authorization

Authentication establishes an identity. Authorization evaluates whether that identity may perform a particular action or access a resource. A valid certificate can establish an identity without granting that identity permission to use every service or operation.

**Key distinction:** Authentication answers “who is at the other end?”; authorization answers “what may they do?”

## Terms

- **Private key:** Secret key controlled by an endpoint or CA; used to create digital signatures.
- **Public key:** Shareable key corresponding to a private key; used to verify signatures made with that private key.
- **CSR:** A signed request for a certificate, containing a public key and requested identity information.
- **Certificate:** A signed data structure that binds an identity to a public key, along with issuer, validity, and other fields.
- **CA:** An authority that issues certificates according to an enrollment policy.
- **Trust anchor:** A root certificate or public key that a verifier has intentionally configured as trusted.
- **SAN:** The X.509 Subject Alternative Name extension, used to state certificate identities such as DNS names, IP addresses, or URIs.
- **Proof of possession:** Evidence that a party controls the private key corresponding to a public key, for example by signing fresh or session-specific data.
