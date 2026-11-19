# GKE Workload Identity: One Workload, Two Identity Jobs

A GKE workload can need an identity for two different audiences: Google Cloud APIs and other workloads. **Workload Identity Federation (WIF) for GKE** provides the Google Cloud IAM identity used for API access. **Managed Workload Identities (MWI)** provide X.509 workload certificates used for peer authentication, including mTLS. One workload can use both.

> **Prerequisite:** Review [Mutual TLS Authentication](mutual-tls-authentication.md) for the certificate, trust, and mutual-authentication concepts used here.

This is a conceptual guide, not a step-by-step deployment procedure. It assumes the example GKE clusters are registered to fleets and that Workload Identity Federation is enabled on their fleet memberships.

## Two different identity jobs

| Question | Mechanism | Credential | Used for |
| --- | --- | --- | --- |
| Which workload is calling a Google Cloud API? | Workload Identity Federation for GKE | A short-lived federated access token, used with an IAM principal | IAM authorization on Google Cloud resources |
| Which workload is connecting to this peer? | Managed Workload Identities | An X.509 certificate and its corresponding private key | Peer identity verification over mTLS |

The mechanisms are complementary, not interchangeable. A WIF IAM principal is not the SPIFFE identity in an MWI certificate. An MWI certificate does not, by itself, grant a workload permission to call a Google Cloud API.

**Key distinction:** Authentication establishes an identity; authorization decides what that identity may do.

## Workload Identity Federation for Google Cloud API access

With WIF for GKE, a Kubernetes ServiceAccount (KSA) is represented as a Google Cloud IAM principal. A workload using a Google Cloud client library can obtain short-lived credentials through the GKE metadata server. IAM grants on the target Google Cloud resource determine what the principal can do. This avoids distributing long-lived service account keys.

For direct IAM access, a KSA principal has a form like:

```text
principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/PROJECT_ID.svc.id.goog/subject/ns/NAMESPACE/sa/KSA_NAME
```

`PROJECT_NUMBER` is the number of the project that owns the workload identity pool. `PROJECT_ID.svc.id.goog` is the GKE pool name, and the remaining path identifies the Kubernetes namespace and ServiceAccount. Grant the required IAM role to this principal on the resource it needs to access.

This guide describes the direct KSA-principal path. Service-account impersonation is intentionally out of scope.

## Managed Workload Identities for peer authentication

Managed Workload Identities give a workload an X.509 certificate containing its workload identity. For GKE, the identity is represented as a SPIFFE ID, commonly in this form:

```text
spiffe://PROJECT_ID.svc.id.goog/ns/NAMESPACE/sa/KSA_NAME
```

The SPIFFE ID identifies the workload within a trust domain. The certificate binds that identity to a public key. The workload keeps the corresponding private key and proves control of it during TLS authentication.

In mTLS, both peers present certificates and prove control of their private keys. Each peer validates the other's certificate chain and identity against its configured trust. Authorization remains a separate decision made by the receiving service or its policy.

**Key distinction:** A certificate can authenticate a workload to a peer; it does not automatically authorize every operation on that peer.

## Workload identity pool, CA issuance, and trust

For GKE, fleet Workload Identity Federation is a prerequisite for Managed Workload Identities. In the MWI setup, the workload identity pool is used in `TRUST_DOMAIN` mode and provides the SPIFFE trust-domain context. The pool terminology appears in both features, but the API-access token and peer-identity certificate remain different credentials. The pool configuration has distinct roles for certificate issuance and peer trust:

- **Certificate issuance configuration** uses `inlineCertificateIssuanceConfig.caPools` to map a workload's region to a regional Certificate Authority Service (CAS) CA pool. Workloads receive certificates from the CA pool configured for their region, and the CA pool must be in the matching region.
- **Trust configuration** determines which certificate authorities are trusted for peer validation. The `inlineTrustConfig` field on the workload identity pool configures additional trust domains and their trust anchors.

The **Trust Store** is a conceptual view of the pool's trust configuration and the effective bundles available to workloads. It is not a separate Google Cloud resource. A workload identity pool bound to a CA pool automatically trusts that CA pool's root certificates; the inline trust config is used to add trust for other domains.

A certificate hierarchy typically has a root CA and one or more subordinate CAs. The root is a trust anchor; a subordinate CA issues workload certificates. The subordinate CA's public key verifies the leaf certificate's signature, and the root trust anchor validates the subordinate CA certificate. The workload presents its leaf certificate and the required intermediate certificates so the peer can build the chain to a trusted root. A private signing key is never included in a trust bundle.

### Project and region are separate boundaries

The example uses Project A as the fleet host project and Project B as a member project containing workloads. These are example ownership boundaries, not a requirement that all GKE resources live in those projects. A fleet groups its member clusters; a project owns resources; a region locates regional resources such as CA pools.

The example's custom CA layout uses a separate root hierarchy for each fleet, with regional subordinate CA pools. That is an isolation choice for this example, not a GKE requirement. A CA pool referenced for a workload region must be in the matching region.

## Cross-fleet mTLS trust

The two-fleet example gives Fleet A and Fleet B separate workload identity pools, SPIFFE trust domains, and CA hierarchies. Keeping the roots separate is a design choice; distinct SPIFFE trust-domain names identify different domains, but do not by themselves create separate signing authority.

For mutual mTLS across the fleets:

1. Fleet A's pool must trust Fleet B's root certificate so Fleet A workloads can validate Fleet B certificates.
2. Fleet B's pool must trust Fleet A's root certificate so Fleet B workloads can validate Fleet A certificates.
3. Each workload presents its own certificate and proves possession of its private key during the handshake.

The peer root certificates are public trust material. The CA private keys and issuing hierarchies remain in their respective fleets. Trusting a peer root does not merge the pools or let one fleet issue certificates for the other.

The pool's inline trust configuration represents the configured trust state; it is not a runtime transfer of root bundles between fleets. In the example, each Trust Store is shown with its local and peer root badges for clarity. In GKE, trust in the pool's bound CA pool is automatic, while additional peer-domain trust is configured explicitly.

## Credentials mounted for a workload

A GKE test pod used the read-only mount path:

```text
/var/run/secrets/workload-spiffe-credentials/
```

The inspected pod contained these files:

| File | Purpose or observed contents |
| --- | --- |
| `certificates.pem` | The workload certificate; in the inspected pod, this included the leaf certificate and intermediate certificate. |
| `private_key.pem` | The workload's private key. Keep it secret and do not share it with peers or CAs. |
| `ca_certificates.pem` | CA trust material. The GKE guide describes the root CA certificate as trust material; with a custom hierarchy, the root might validate a subordinate CA that signs the leaf. |
| `trust_bundles.json` | A mapping of trust domains to CA certificates that the workload should trust. The GKE guide says this file is available in versions after `1.35.2-gke.1291000`; check the current guide for the version requirement. |
| `credential-bundle.pem` | Also present in the inspected test pod. Treat this as an observed file, not a universal file contract. |

File contents and availability can vary with GKE version and configuration. In particular, the observed leaf-plus-intermediate chain in `certificates.pem` should not be read as a guarantee that every GKE version mounts the same chain in the same file.

## Summary

- Use **WIF for GKE** when a workload needs a Google Cloud IAM identity to call Google Cloud APIs.
- Use **Managed Workload Identities** when workloads need X.509 identities for peer authentication, such as mTLS.
- A workload can use both mechanisms for different purposes.
- The workload identity pool carries trust-domain configuration. Its certificate-issuance configuration and trust configuration serve different roles.
- For cross-fleet mTLS, both pools must trust the peer fleet's certificates. Keep public trust material distinct from private CA keys.

## Terms

- **Kubernetes ServiceAccount (KSA):** A Kubernetes identity assigned to a workload.
- **Workload identity pool:** The GKE pool associated with workload identities and their trust domain.
- **SPIFFE ID:** A URI-form workload identity, such as `spiffe://PROJECT_ID.svc.id.goog/ns/NAMESPACE/sa/KSA_NAME`.
- **Trust anchor:** A root CA certificate accepted by a verifier as the starting point for validating a certificate chain.
- **CA pool:** A regional Certificate Authority Service resource that contains certificate authorities used for issuance.

## References

- [Workload identities for GKE](https://docs.cloud.google.com/iam/docs/workload-identities)
- [About fleet Workload Identity Federation](https://docs.cloud.google.com/kubernetes-engine/fleet-management/docs/about-fleet-workload-identity-federation)
- [Managed workload identity overview](https://docs.cloud.google.com/iam/docs/managed-workload-identity)
- [Configure managed workload identity authentication for GKE](https://docs.cloud.google.com/iam/docs/create-managed-workload-identities-gke)
- [Workload identity pool REST resource](https://docs.cloud.google.com/iam/docs/reference/rest/v1/projects.locations.workloadIdentityPools)
