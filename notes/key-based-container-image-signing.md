# Key-Based Container Image Signing

This overview follows the path from a container image to a deployment decision. The central idea is that the publisher signs the image's digest—not the registry's approval of the image—and a verifier checks that signature using a trusted public key.

## Start with a public/private key pair

A key pair has two related parts with different roles:

- The **private key** is kept under the publisher's control and used to create signatures.
- The matching **public key** can be shared with verifiers and used to check those signatures.

A verifier must already trust the public key it uses. A valid signature shows that the signed data matches that key; it does not, by itself, establish who controls the key or whether the image is safe.

## Identify the exact image with a digest

A container image is represented by structured content, including a manifest (or, for a multi-platform image, an image index). A digest is computed from that representation and identifies that exact image version. If the content changes, its digest changes.

The digest is an external identifier, not a value stamped into the image itself. A registry can resolve a tag to a digest and expose or record that association, but the digest is not embedded in the image's contents.

## Sign the image digest

Once the image is finalized, the publisher computes its digest and signs that digest with the private key. This produces a **separate signature artifact** that attests to the image digest.

The image digest and the signature are different things:

- The **digest** identifies the image.
- The **signature** is evidence that the corresponding private key signed that digest.

Changing the image after signing results in a different digest, so the existing signature will not verify for the changed image.

## Store the image and signature separately

The registry stores the container image and the signature artifact as separate items. The signature is associated with the image digest so that a verifier can find the signature for the image being checked.

The registry is storage and distribution infrastructure. Its presence does not make an image trusted, and it does not itself verify the signature.

## Verify before deployment

A verifier obtains the image and its associated signature, then:

1. Computes or obtains the digest for the image being checked.
2. Checks the signature using a trusted copy of the matching public key.
3. Confirms that the signed digest matches the digest of the image being considered.
4. Applies the relevant deployment policy.

If the signature verifies for that image digest and the policy allows it, the image can proceed to deployment. If verification fails—or the image has changed—the verifier should not treat it as verified.

## In short

**Image → digest → signature made with the private key → image and signature stored separately → verification with a trusted public key → deployment decision.**

