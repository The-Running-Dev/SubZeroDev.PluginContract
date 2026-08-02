# ADR-001: A Plugin Is a Capability, Not an Image

## Status

Accepted

Originally drafted as ADR-002 in a single global sequence, renumbered when the specifications split
by destination repository. Numbering is per repository.

## Context

The first plugin is a Docker image, and the early documents described plugins in Docker's terms —
build the image, pull the image, run the image. Left alone, that vocabulary becomes the definition,
and "plugin" and "container" turn into synonyms.

They should not be. Several of the planned plugins are natural Docker citizens; the Work Items
library binds to Python; a PowerShell module has no image at all; and a remote API endpoint that
satisfies the same command contract is a plugin by every criterion that matters. If the contract is
written in Docker's terms, each of those becomes an exception, and a contract that accumulates
exceptions is one that will eventually be replaced.

What this settles is what a plugin actually _is_, because everything downstream follows from the
answer: the shape of the manifest, whether the contract is a process boundary or a language
interface, and what conformance can assert.

## Decision

**A plugin is a capability defined by its manifest and its command contract. The runtime that
delivers it is an implementation detail, declared in `runtimes[]`.**

The supported runtime types are `docker`, `process`, `dotnet`, `node`, `python`, `powershell`, and
`remote`. The set is closed and an unknown type is refused, because a runtime that cannot be executed
is not forward compatibility — it is a silent no-op.

Two consequences are load-bearing rather than incidental.

**The contract is a process boundary.** Commands, stdout JSON, exit codes, and artifacts are the
surface, because a process boundary is the only thing every language already shares. Expressing the
contract as a C# or TypeScript interface would make "plugins in any language" false the moment it was
written.

**Capability enforcement is per runtime host, and the manifest states which level applies.** Only
Docker enforces the declared capabilities; process-based hosts enforce none of them. That difference
is real, and pretending otherwise would let an operator believe a `process` runtime is sandboxed. The
plugin declares what it needs, the host declares what it can enforce, and the operator sees both.

## Consequences

- The manifest, not the image, is the unit of identity. This is why it must be obtainable without
  running the plugin — resolved by publishing it as a signed OCI attestation in ADR-004.
- The Automator needs a runtime host per runtime type, which is real work a Docker-only design would
  have avoided. Accepted: the alternative is a second, incompatible plugin concept the first time
  something is not an image.
- Docker remains the preferred isolation and distribution mechanism, and the only one where the
  capability model is enforced rather than advisory. Preferred is not required.
- Conformance tests the process boundary, so it applies unchanged to every runtime type.

## Alternatives considered

**Define a plugin as a Docker image.** Simplest, and it matches the first implementation exactly.
Rejected: it makes non-container plugins exceptions to the contract rather than instances of it, and
the ecosystem already contains several that would be exceptions on day one.

**Define a plugin as a language interface with adapters.** Would give first-class tooling in the host
language. Rejected: it privileges one language and demotes the rest to bindings, which is the
opposite of the goal. In-process bindings may still exist as per-language conveniences, but as
optimizations over this contract rather than a second definition of it.
