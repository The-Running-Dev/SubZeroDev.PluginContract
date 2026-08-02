# Conformance Suite

Split from `17-testing-strategy.md`, which retains the ecosystem-wide layer model. This document
specifies the suite that decides whether a plugin satisfies the contract.

## Why this exists

The contract is prose until something checks it. The suite is what makes "use the GitHub plugin as a
template" a verifiable claim rather than an aspiration, and it is what lets a plugin written by
someone else be trusted to behave.

It tests **the contract, not the copy**. A plugin scaffolded from the GitHub plugin inherits that
plugin's structure; conformance must not care about structure, only about observable behaviour at the
process boundary.

## Invocation

```bash
subzerodev-conformance --image ghcr.io/…/subzerodev-github@sha256:…
subzerodev-conformance --command "node dist/cli.js"
```

Both targets are supported deliberately. The image target is the real contract. The local-command
target lets a plugin author run conformance in a unit-test loop without a Docker daemon, which is the
difference between running it on every commit and running it once before release.

Where a check is meaningless for the local target — non-root, read-only mounts — it is skipped and
reported as skipped, never as passed.

## Checks

### C1 — Manifest in a bare container

Run `manifest` with no configuration, no secrets, no network, and no mounts.

Asserts: exit 0; stdout is exactly one JSON document; it validates against
`plugin-manifest.schema.json`; the declared `id` and `version` match the image labels.

This is the check most likely to fail first, because loading configuration in a shared startup path
breaks it and that is the natural way to write a CLI.

### C1b — Attested manifest agrees with the runtime manifest

Where the image carries a signed manifest attestation, verify the signature and assert the attested
manifest is byte-identical to what `manifest` prints.

A plugin that attests one set of capabilities and reports another at runtime is rejected. This is the
check that makes reading capabilities before execution trustworthy — without it, the attestation is
a claim rather than a guarantee.

Skipped for local-command targets and for images with no attestation, and reported as skipped.

### C2 — Universal commands

`--help` exits 0 with non-empty stdout. `--version` exits 0 and prints a semantic version matching
the manifest. An unknown command exits **2**, not 1 and not 127.

### C3 — Output channel purity

For each command, in `--output-format json`, with logging forced to `trace`:

Asserts stdout parses as exactly one JSON document validating against the envelope schema.

**Forcing the loudest log level is the point.** At `info` a stray log line may not appear; at `trace`
it will. This is the check that catches a logger left on stdout, and it is the highest-value check in
the suite because that defect breaks every adapter simultaneously and presents as a parse error.

### C3b — Envelope invariants the schema cannot express

`result-envelope.schema.json` enforces the envelope's shape, including that `status` and `exitCode`
agree and that a `failed` or `partial` status carries at least one entry in `errors`. Two rules are
outside what JSON Schema can state, so they are asserted here or they are not asserted anywhere:

| Rule                                    | Why the schema cannot say it      |
| --------------------------------------- | --------------------------------- |
| `data` is at most 256 KiB serialized    | No serialized-size keyword exists |
| `finishedAt` is at or after `startedAt` | No cross-field comparison         |

A third rule was listed here in the first draft — non-empty `errors` on a failure — on the assumption
that a constraint conditioned on a sibling field could not be expressed. It can: the schema already
branches on `status`, and the same branch carries `minItems`. It moved into the schema, where a host
validating without this suite still gets it.

Also asserts the envelope's `exitCode` equals the process exit code the runner observed, and that
each `artifacts[]` entry matches the file on disk in both `bytes` and `sha256`. The envelope is what
a host records; an envelope that disagrees with reality is worse than a missing one, because nothing
downstream re-checks it.

### C4 — Declared artifacts

Asserts every artifact declared `required: true` exists at its declared path after a successful run,
validates against its `schemaRef` where one is declared, and that its reported digest matches the
file.

Also asserts undeclared artifacts are reported rather than silently produced.

### C5 — Exit codes

Drives each condition the plugin can be made to produce and asserts the contract's code:

| Condition                 | Expected |
| ------------------------- | -------- |
| Success                   | `0`      |
| Unknown option or command | `2`      |
| Missing required secret   | `5`      |
| Invalid configuration     | `2`      |
| Cancellation signal       | `130`    |

Conditions a plugin cannot be driven into from outside are skipped and reported as skipped. Partial
success and rate limiting usually fall here, which is why fixture plugins exist.

### C6 — Secret canary

Runs with a known sentinel as the declared secret, then greps stdout, stderr, every artifact, the
cache directory, and the serialized envelope for the sentinel.

Also inspects image layers for the sentinel, catching a secret baked in at build time.

A single occurrence anywhere is a failure. There is no threshold.

### C7 — Container hygiene

Asserts the container does not run as UID 0, tolerates a read-only configuration mount, tolerates a
read-only root filesystem where declared, and requires no TTY.

### C8 — Determinism

Runs the same command twice against identical inputs and asserts every artifact is byte-identical.

The envelope is expected to differ — it carries timestamps — and is excluded. A plugin that embeds a
timestamp in an artifact fails here, which is the intent.

### C9 — Path confinement

Attempts to have the plugin write an artifact declared with a traversing path and asserts refusal.
Both separators are exercised — `../etc`, `..\etc`, `a/..\b`, `/etc/passwd`, `\\server\share`, and
`C:\secret` — because the contract supports Windows plugins and a POSIX-only check passes a host that
is not safe.

This check needs a cooperating fixture, since a well-behaved plugin will not attempt it. It belongs
here because the schema rejects such a path at validation time and this proves the runtime rejects it
too — and the runtime has to do more than the schema can: **canonicalize the resolved path and verify
containment**, since a pattern cannot see where a symlink points.

## Fixture plugins

The suite is only trustworthy if it has been shown to fail. Fixtures exist to prove each check
detects what it claims:

| Fixture            | Purpose                                                                   |
| ------------------ | ------------------------------------------------------------------------- |
| `echo`             | Minimal conforming plugin; the reference every check passes against       |
| `failing`          | Returns each documented exit code on demand                               |
| `timeout`          | Ignores cancellation; proves the host kills it and records `124` or `130` |
| `artifact`         | Produces declared, undeclared, and missing-required artifacts             |
| `leaky`            | Deliberately prints its secret to stdout; **C6 must fail against it**     |
| `noisy`            | Logs to stdout in JSON mode; **C3 must fail against it**                  |
| `nondeterministic` | Embeds a timestamp in an artifact; **C8 must fail against it**            |
| `traversal`        | Declares an escaping artifact path; **C9 must fail against it**           |

The last four are the important ones. A suite that passes everything it is pointed at is not
evidence, and these fixtures are the regression tests for the suite itself.

Fixtures should exist for more than one runtime once a second runtime is supported, so the suite is
not accidentally specialized to Node.

## Reporting

The suite emits a conformance report as JSON: contract version, plugin identity, and per-check
pass, fail, or skip with detail. A plugin's CI publishes it as an artifact.

**Skips are reported prominently.** A run that skips six checks and passes three is not a pass, and a
report that renders it as green is worse than no report.

## Versioning

The suite is versioned with the contract. A plugin claiming contract `1.0` is tested by the `1.0`
suite.

Adding a check is a minor version bump and may fail plugins that previously passed — which is
correct, and why plugins pin the contract version they claim.

## Decisions on previously open points

**Conformance gates publication.** A plugin that fails and publishes anyway makes the contract
advisory, and an advisory contract is one every plugin diverges from in its own direction. Failing
conformance blocks the release job.

**A plugin cannot declare a check inapplicable.** Checks are skipped only as a consequence of what the
manifest already declares — a plugin with no container runtime skips the container-hygiene checks
because it has no container, not because it asked to. A free-form opt-out is self-certification, and
a suite that accepts self-certification is not testing anything.
