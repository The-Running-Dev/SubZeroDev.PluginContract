# ADR-004: Plugin Signing and the Trust Root

## Status

Accepted

## Context

The contract defines four trust levels — first-party, development-local, signed third-party, and
untrusted — and makes them govern runtime selection, network access, filesystem access, secrets, and
MCP exposure.

Two of the four could not be established. There was no signing mechanism, no trust root, and no
verification step, so "signed third-party" was unimplementable and "untrusted" had nothing to
contrast with. The contract consequently exposed only the two levels that could be verified, which is
honest but limits the system to plugins published by one organization.

There is also a security inversion in the current design that signing resolves. A host must know what
capabilities a plugin wants _before_ deciding whether to run it — but the only way to obtain the
manifest is to run the container's `manifest` command. **Executing an untrusted container to ask what
it needs is backwards**, and no amount of care inside the plugin fixes it.

## Decision

### Sign OCI images with Sigstore cosign

Plugins are distributed primarily as OCI images and are already pinned by digest. Cosign signs images
natively and stores the signature as an OCI artifact beside them, so nothing new is needed to
distribute a signature.

**Two signing modes, both supported:**

| Mode                                | Used for                                            | Trade-off                                                                            |
| ----------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Keyless — OIDC via Fulcio and Rekor | First-party and most third-party publishing from CI | No long-lived private key exists to leak or rotate; requires network at signing time |
| Key pair                            | Air-gapped or offline publishing                    | Works anywhere; reintroduces key custody, which is the usual failure mode            |

Keyless is the default because the most common signing failure is not a broken algorithm — it is a
private key that leaked, expired unnoticed, or lived on one person's laptop. Binding the signature to
a CI workflow identity removes the key entirely.

### Publish the manifest as a signed attestation

The manifest is attached to the image as a signed OCI referrer, in canonical JSON.

This is the part that changes behaviour rather than just adding a signature: **the host reads and
verifies a plugin's declared capabilities without executing anything.** Installation, policy
evaluation, and capability review all happen against verified metadata, and the container is started
only after the host has decided it is willing to.

The `manifest` command remains required. It is the local development path and, in conformance, the
check that the command and the attestation agree — a plugin whose attested manifest differs from what
it reports at runtime is rejected.

### The trust root

| Trust level        | Established by                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| First-party        | Keyless signature whose certificate identity matches the organization's release workflow, pinned by OIDC issuer and workflow ref |
| Signed third-party | Signature matching an identity or public key in the operator's configured allowlist                                              |
| Untrusted          | Image with no valid signature, or a valid signature from an identity nobody has allowed                                          |
| Development-local  | An image or path the operator named explicitly; verification skipped and recorded as skipped                                     |

For first-party plugins the pinned identity is the workflow, not a person: issuer
`https://token.actions.githubusercontent.com`, subject the release workflow at a tag ref. A signature
produced by any other workflow — including a pull-request build in the same repository — does not
satisfy it.

**There is no global registry of trusted publishers and no implicit trust.** A third-party plugin is
trusted because a specific operator added a specific identity to an allowlist, which is the only
model that does not require someone to be the internet's certificate authority.

### Verification points

- **At install**, before the manifest is registered or any capability is granted. This is the
  decision point.
- **At run**, by digest equality against the verified digest recorded at install. Cheap, and it
  catches a poisoned local image cache without re-verifying signatures on every execution.
- **On policy change**, re-verify affected plugins, because the allowlist may have shrunk.

### Offline verification

Cosign bundles the certificate and the transparency-log inclusion proof with the signature, so
verification does not require network access at run time. The Fulcio root is pinned locally and
refreshed on a schedule.

The honest trade-off: an offline installation verifies against a trust root that may be stale, and
cannot consult the transparency log for anything published since its last refresh. For a homelab or
air-gapped deployment that is the correct trade — the alternative is a system that stops working when
a network is unavailable.

### Revocation

**Revocation is operator-driven, not cryptographic.** Sigstore has no revocation primitive; the
transparency log provides detection, not prevention.

Three mechanisms, in the order they take effect:

1. **Digest quarantine** in the Automator registry. Immediate, local, already an administrative
   action.
2. **A revocation list** of digests and identities, consulted at install and on policy re-verification.
3. **Allowlist removal** for a third-party identity, which invalidates every future install and, on
   re-verification, existing ones.

None of these reaches a plugin already running. Quarantine prevents new executions; stopping current
ones is a separate, explicit cancel.

## Consequences

- All four trust levels become establishable, so third-party plugins are possible without granting
  them first-party privileges.
- Capability review happens before execution rather than after, which closes the inversion described
  above.
- Untrusted plugins have a meaningful definition: unsigned or unrecognized. Policy can restrict them
  to the Docker host, with no secrets and no network unless explicitly granted.
- Signing must operate on **canonical JSON**, never on the authored YAML. Two byte-different YAML
  files with identical meaning would otherwise produce different signatures, and a formatter run
  would invalidate a signature.
- Publishing gains a hard dependency on the release workflow's OIDC identity, so releases cannot be
  cut from a developer machine without falling back to key-pair mode.
- The Automator gains configuration surface — allowlist, revocation list, pinned roots — that must be
  backed up. A restored system without its allowlist trusts nothing, which is the correct failure
  direction.
- Verification is a network operation at install time in keyless mode. Installing a new plugin on an
  air-gapped host requires the signature bundle to have been fetched alongside the image.

## Alternatives considered

**GPG signatures with a web of trust.** Familiar and fully offline. Rejected: key distribution and
revocation are the hard parts, and the web of trust model has repeatedly failed to achieve adoption
even among people who understand it.

**Notary v1 / Docker Content Trust.** Purpose-built for images. Rejected: effectively unmaintained,
with a key management story that pushed most adopters away.

**No signing; rely on digest pinning alone.** Digest pinning already guarantees you run the bytes you
recorded. Rejected because it answers "did this change" and not "who published it" — which is exactly
the question a third-party plugin raises.

**A central registry of trusted publishers.** Better user experience. Rejected: it makes this project
responsible for vouching for third parties, which is a governance burden entirely disproportionate to
its size.
