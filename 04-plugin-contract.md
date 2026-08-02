# Plugin Contract

| Field            | Value                                                     |
| ---------------- | --------------------------------------------------------- |
| Contract version | 1.0.0-draft                                               |
| Status           | Draft — blocking items resolved, open questions remain    |
| Destination repo | Its own repository, versioned and tagged independently    |
| Supersedes       | The earlier `setup-llm/docs/` contract draft, now deleted |

This contract lives in its own repository so that a Node plugin, a Python plugin, and the .NET
Automator can all consume it without any of them depending on the others. It is versioned and tagged
independently, which is what lets a plugin pin a contract version.

## Precedence

**This contract outranks every plugin specification.** Where a plugin document and this one disagree,
this one is correct and the plugin document has drifted.

The practical rule for authors: a plugin specification should contain only what is true of _that
plugin_ and false of another. Anything true of plugins generally — how secrets arrive, what stdout
carries, which exit code means what, how artifacts are declared, what determinism requires — belongs
here and is **referenced, not restated**.

Restating is how the two documents diverge. The GitHub plugin's specification originally carried its
own exit-code table, its own secret-handling rules, and its own serialization rules; each was a
generic decision written in a plugin-specific place, and one of them had already drifted into a
direct contradiction before anyone noticed.

When a new decision arises, the first question is which document owns it. If a second plugin would
face the same question, the answer is this one.

## Definition

A plugin is an independently versioned capability that exposes one or more commands through a stable
manifest and invocation contract.

A plugin is not defined by its implementation language or packaging format. Docker is the preferred
distribution and isolation mechanism, and the only one that can enforce the capability model, but it
is not the definition.

## The normative surface

A plugin is defined by what it does at the process boundary: the commands it accepts, the JSON it
writes to stdout, the exit code it returns, and the artifacts it leaves behind.

A process boundary is the only surface every language already shares. Expressing this contract as a
language interface — a C# or TypeScript type — would make "plugins in any language" false the moment
it was written. In-process bindings may exist later as per-language conveniences, but they are
optimizations over this contract, never a second definition of it.

It follows that **the JSON Schemas in `schemas/` are the normative artifact, not any generated
types.** Types may be generated from the schemas, and the schemas may in turn be authored in Zod or
similar, but what an adapter in another language consumes is the committed, versioned schema file.

## Two boundaries that are easy to confuse

```text
        ┌──────────── plugin boundary (this document) ────────────┐
host  →  adapter  →  [ CLI → services → provider → vendor SDK ]  →  artifacts
                                        └── provider boundary ──┘
```

- The **provider boundary** is internal: GitHub versus GitLab, OpenAI versus Anthropic. It exists
  only inside plugins that integrate a third-party service, and it is the plugin author's business.
- The **plugin boundary** is external: host versus plugin. Every plugin has it, and it is this
  document.

Conflating them is the likeliest way to get this wrong — for example by promoting a domain type such
as `Project` into the contract, which would only make sense for plugins that happen to produce
projects.

## Required properties

Every plugin must declare:

- a globally stable ID
- a semantic version
- display name and description
- manifest schema version
- one or more commands
- at least one runtime implementation
- input and output schemas per command
- declared secrets
- declared artifacts
- declared capabilities
- license metadata
- compatibility metadata

Each of these is enforced by `schemas/plugin-manifest.schema.json`. Where the schema and this prose
disagree, that is a defect in one of them — report it rather than working around it.

## Identity

### Plugin ID

```text
subzerodev.github
subzerodev.requirements
subzerodev.documentation
subzerodev.container-ps-generator
```

Lowercase, stable, never reused for a different capability.

### Command ID

```text
sync   list   stats   export   validate
```

Fully qualified: `subzerodev.github.sync`.

### Name mapping

One capability has several names in several systems. The mapping is mechanical, and stating it
prevents the five-way drift recorded in `REVIEW.md` (C12):

| Surface          | Form                           | Example                                     |
| ---------------- | ------------------------------ | ------------------------------------------- |
| Plugin ID        | `subzerodev.<name>`            | `subzerodev.github`                         |
| CLI binary       | `subzerodev-<name>`            | `subzerodev-github`                         |
| CLI alias        | `sz-<name>`                    | `sz-github`                                 |
| Container image  | `<registry>/subzerodev-<name>` | `ghcr.io/the-running-dev/subzerodev-github` |
| Language package | per-ecosystem convention       | `@subzerodev/plugin-github`                 |

The long form is canonical: manifests, documentation, and examples use `subzerodev-github`. The
`sz-` form is a convenience alias for interactive use, in the way `kubectl` is often aliased to `k`.
Automation should not depend on the alias.

## Required commands

| Command       | Required       | Behavior                                                  |
| ------------- | -------------- | --------------------------------------------------------- |
| `manifest`    | Yes            | Print the manifest as JSON. Exit 0.                       |
| `validate`    | Yes            | Check configuration and readiness without doing the work. |
| `--help`      | Yes            | Usage. Exit 0.                                            |
| `--version`   | Yes            | Plugin version. Exit 0.                                   |
| Work commands | Plugin-defined | Declared in the manifest.                                 |

`manifest` replaces the `describe` and `capabilities` commands in the original draft, which were
listed as two commands with no stated difference between them. One command returns the whole
manifest; a host that wants only the capabilities section reads that key.

### Published as a signed attestation

The manifest is also attached to the image as a signed OCI referrer, in canonical JSON, so a host can
read and verify what a plugin needs **without executing it**. Running an untrusted container to ask
what it wants is backwards; installation and capability review happen against the attestation, and
the container starts only once the host has decided to allow it.

Conformance checks that the attested manifest and the `manifest` command agree. A plugin whose
runtime answer differs from what it attested is rejected. See `adr/ADR-004`.

### The bare-container requirement

**`manifest` must succeed with no configuration file, no secrets, no network, and no mounted
volumes.** A host must be able to ask an unconfigured image what it is and what it would need before
it is prepared to run it.

This is easy to break by loading configuration in a shared startup path that runs before command
dispatch. Conformance tests it directly.

## Channels

| Channel   | Carries                                                           |
| --------- | ----------------------------------------------------------------- |
| stdout    | Machine output: the manifest, or the result envelope in JSON mode |
| stderr    | Human text and structured logs, always                            |
| Exit code | The outcome                                                       |

**Logs must never reach stdout.** One log line on stdout corrupts the envelope and breaks every
adapter simultaneously, presenting as a parse bug rather than a logging bug. This is not
hypothetical: several common structured loggers — Pino among them — write to stdout by default, so
the logger must be explicitly constructed against stderr.

Precisely:

- `manifest` always writes one JSON document to stdout.
- A work command writes the envelope to stdout in JSON mode, and a human summary otherwise.
- `--help` and `--version` write human text to stdout; they are for people.
- Everything else goes to stderr.

## Manifest

Authored in YAML, validated and canonicalized as JSON. See `schemas/plugin-manifest.schema.json` and
`examples/plugin.yaml`.

```yaml
schemaVersion: "1.0.0"
id: subzerodev.github
name: SubZeroDev GitHub
version: 1.0.0
description: Collects and normalizes GitHub repository metadata.
license: MIT

commands:
  - id: sync
    description: Synchronize repository metadata.
    inputSchema: schemas/sync.input.schema.json
    outputSchema: schemas/sync.output.schema.json
    timeout: PT15M
    idempotency: idempotent
    artifacts:
      - name: projects
        path: projects.json
        mediaType: application/json
        schemaRef: schemas/projects.schema.json
        required: true

runtimes:
  - id: docker
    type: docker
    image: ghcr.io/the-running-dev/subzerodev-github
    digest: "sha256:…"
    entrypoint: ["subzerodev-github"]

secrets:
  - id: github-token
    required: true
    environmentVariable: GITHUB_TOKEN
    description: GitHub personal access token with read access to repositories.

capabilities:
  network:
    required: true
  filesystem:
    read: [config]
    write: [cache, output]

compatibility:
  minimumAutomatorVersion: "1.0.0"
  operatingSystems: [linux, windows, macos]
  architectures: [amd64, arm64]
```

Fields that carry weight:

- **`secrets` declares names, never values.** This is how a host knows what to inject without the
  plugin ever describing a credential it holds.
- **`capabilities` is the security surface.** It is strictly schema-constrained, and unknown keys are
  refused rather than ignored — see Compatibility below.
- **`digest`** pins the image immutably. A version tag is mutable and is acceptable only for
  development runtimes.
- **`idempotency`** is an enumeration, not a boolean: `idempotent`, `conditional`, or
  `non-idempotent`. `conditional` requires an `idempotencyCondition` string stating what the
  condition is, because a retry policy cannot act on an unexplained "sometimes".

## Invocation

| Input         | Mechanism                                                                        |
| ------------- | -------------------------------------------------------------------------------- |
| Configuration | Read-only mount at `/etc/subzerodev/plugin.config.json`, or `--config`           |
| Secrets       | Environment variables only, named by the manifest                                |
| Cache         | Writable mount, `SUBZERODEV_PLUGIN_CACHE`, default `/var/lib/subzerodev/cache`   |
| Output        | Writable mount, `SUBZERODEV_PLUGIN_OUTPUT`, default `/var/lib/subzerodev/output` |

Secrets travel through the environment and nowhere else. Not `argv`, which is world-readable via
`/proc` on Linux; not the configuration file, which is the thing people commit.

The cache and output variable names are deliberately **plugin-neutral**, so an adapter can mount
working directories without per-plugin knowledge.

Additional requirements:

- The container runs as a non-root user.
- No TTY is required, and no run may prompt for input.
- A run must tolerate a read-only configuration mount.
- All inputs normalize into one command input object before execution.

## Result envelope

Normative definition: `schemas/result-envelope.schema.json`. The example below illustrates it and
does not replace it.

```json
{
  "schemaVersion": "1.0.0",
  "plugin": { "id": "subzerodev.github", "version": "1.0.0" },
  "command": "sync",
  "status": "partial",
  "summary": "Synchronized 40 of 42 repositories.",
  "startedAt": "2026-07-28T10:00:00Z",
  "finishedAt": "2026-07-28T10:02:13Z",
  "data": {},
  "warnings": [],
  "errors": [
    {
      "code": "repository_statistics_unavailable",
      "message": "Statistics endpoint did not settle within the retry budget",
      "subject": "repository:12345",
      "retryable": true
    }
  ],
  "artifacts": [
    {
      "name": "projects",
      "path": "projects.json",
      "bytes": 48213,
      "sha256": "…"
    }
  ],
  "metrics": {},
  "exitCode": 4
}
```

Changes from the original draft:

- **`errors[]` added.** The original had `warnings` but no errors, leaving a failing plugin nowhere
  structured to report why.
- **`startedAt` / `finishedAt` / `exitCode` added**, so the envelope stands alone without the host
  having to reconstruct it.
- **`status` is one of** `succeeded`, `partial`, `failed`, `cancelled`, `timedOut`. It duplicates
  what the exit code says, deliberately, so an adapter can branch on a word rather than memorize a
  table.
- **`errors[].retryable`** lets the plugin, which knows most about the failure, advise the retry
  policy instead of the host guessing from an exit code.

### What the schema decides that the prose left open

Writing the schema forced four questions the description had not answered.

**Every field above is required.** `data`, `warnings`, `errors`, `artifacts`, and `metrics` are
emitted as `{}` or `[]` rather than omitted, following the serialization rule that absence is written
rather than implied. A consumer never has to distinguish "no warnings" from "this plugin does not
report warnings".

**`status` and `exitCode` cannot disagree.** The schema constrains each status to its code —
`succeeded` to `0`, `partial` to `4`, `timedOut` to `124`, `cancelled` to `130`, and `failed` to
anything else assignable. The two fields are deliberate duplication, and duplication that is allowed
to drift is how the `3`/`5` collision in ADR-003 happened. Here it is checkable, so it is checked.

**Timestamps are UTC with a `Z` suffix**, not a local offset. One representation keeps host-side
ordering and comparison free of timezone handling.

**Plan tokens are top-level, not inside `data`.** A `plan` object carrying `planId`, `expiresAt`, and
`fingerprint` appears on the read-only half of a plan-apply pair. Inside `data` it would be
plugin-shaped and invisible to a generic host, which defeats the point of a structural gate — a host
must be able to see that an approval is pending without knowing what the plugin does.

Two envelope rules are not expressible in JSON Schema and belong to the conformance suite instead:
the 256 KiB cap on `data`, and `finishedAt` being at or after `startedAt`.

A third — a `failed` or `partial` status carrying at least one entry in `errors` — was written off as
inexpressible and is not. The schema already branches on `status` to constrain `exitCode`, so the same
conditional carries `minItems: 1` on `errors`. It is enforced in the schema, which matters because a
host validating against the schema alone would otherwise accept a failure that says nothing about
why.

### `data` is bounded

**`data` carries a summary, not a payload. It is capped at 256 KiB.** Anything larger must be an
artifact.

This matters because the Automator persists normalized output in execution history on SQLite in
early phases. Without a bound, a plugin returning a multi-megabyte result writes it into the
database on every run.

The distinction is: artifacts are the deliverable, `data` is what a human or a workflow condition
needs to see without opening one.

### Envelope versus execution record

This envelope is what the **plugin emits**. The Automator's execution record — which additionally
carries queue timing, retry history, host and agent metadata, and a log reference — is a different
model, specified in the Automator documents. Conflating the two was an inconsistency in the original
set.

## Exit codes

| Code  | Meaning                                 |
| ----- | --------------------------------------- |
| `0`   | Success                                 |
| `2`   | Usage or validation error               |
| `3`   | Operational failure                     |
| `4`   | Partial success                         |
| `5`   | Authentication or authorization failure |
| `6`   | Rate-limited or quota-exhausted         |
| `124` | Timed out                               |
| `130` | Cancelled or interrupted                |

**`1` is reserved and never assigned.** Most runtimes return it for an uncaught exception, so
leaving it unassigned keeps "the plugin crashed" distinguishable from "the plugin reported a
failure" — a distinction retry policy depends on, because a crash may be transient where a reported
validation failure is deterministic.

`124` and `130` follow `timeout(1)` and `128 + SIGINT`, so shell tooling and container runtimes
already produce them.

Plugins may define additional codes above `6` and must document them in the manifest. This table is
canonical; no other document restates it.

Partial success is a first-class outcome, not a failure: the run kept prior valid state for what
failed, wrote what succeeded, and said so in `errors[]`.

## Compatibility

The compatibility policy the original set left open.

### Manifest schema version

- A host **accepts** a manifest whose `schemaVersion` major matches its own.
- A host **refuses** a higher major, with an error naming both versions.
- A host **accepts** a higher minor and follows the unknown-field rules below.

### Unknown fields

| Location          | Unknown field | Rationale                                                                  |
| ----------------- | ------------- | -------------------------------------------------------------------------- |
| `capabilities`    | **Refuse**    | An unrecognized permission would otherwise be neither granted nor reported |
| `secrets`         | **Refuse**    | Same — a misunderstood secret declaration is a security failure            |
| `runtimes[].type` | **Refuse**    | An unknown runtime cannot be executed                                      |
| Everything else   | Ignore        | Descriptive metadata should not break forward compatibility                |

Failing open on capabilities was the original schema's behavior, by way of
`additionalProperties: true` everywhere. It was not a decision anyone made, and it is the wrong
default for the security surface.

## Idempotency, retries, and timeouts

Commands declare `idempotent`, `conditional`, or `non-idempotent`. `conditional` must state its
condition.

Rules:

- Non-idempotent commands are never retried automatically unless explicitly configured per step.
- **A retry must confirm the previous attempt is dead before starting.** Container stop is
  asynchronous; retrying a timeout without confirming termination can run two copies of a
  non-idempotent command concurrently.
- Retry policies key on `errors[].retryable` and on exit code. Exit `2` is never retryable: a usage
  error is deterministic.
- Commands creating external resources should support an idempotency key where the provider allows
  one.

## The plan-apply pattern

Where a command writes to a system outside the plugin's own storage, the write is gated by a token
from a prior read-only call.

1. A read-only command computes what would change and returns an opaque `planId` — random, not a
   hash of the content — plus a rendering a human can review. Both travel in the envelope's
   top-level `plan` block, so a host sees the pending approval without understanding the plugin.
2. The write command takes **only** the `planId`. No target, no content, nothing that would let it
   act without a plan.
3. A plan is single-use, TTL-bounded, and carries a fingerprint of the state it was computed
   against.
4. The write refuses a plan that is unknown, expired, already used, or whose target has changed since
   the plan was taken.

The fingerprint check is the one most often skipped and the one that matters: between plan and apply,
someone may have changed the target by hand. Applying a stale diff over their change is worse than
refusing.

**Why this is in the contract rather than in each plugin.** Three plugins already need it, expressed
three different ways — the release plugin's dry-run default, the package plugin's refuse-on-mismatch,
and the Requirements Compiler's compile-approve-publish. It is one pattern, and a second plugin faces
the same question, which is the test ADR-003 sets.

It also stops being optional under MCP. A prose instruction to stop and wait for approval works when
documentation is loaded alongside; a different client's model never reads it. The gate must be
structural or it does not exist — and an instruction injected into the plugin's input cannot
fabricate a plan token.

`--dry-run` remains required for side-effecting commands. Plan-apply is the stronger form, for
writes where seeing the diff first is not merely polite.

## Optional MCP surface

A plugin may implement an `mcp` command, serving its own commands over the Model Context Protocol so
an AI client can reach it directly. The tool surface is **projected from the manifest**, never
hand-written.

It is optional: a plugin without it is fully conforming and remains reachable through the Automator's
brokered MCP server, which projects the same manifest. Specified in `SubZeroDev.MCP/`.

## Cancellation

Hosts propagate cancellation by signal, container stop, or an agent request. Plugins should clean up
and exit `130`. A plugin that ignores cancellation is killed, and the execution records that it had
to be.

## Artifacts

- Declared in the manifest with a name, path, media type, and schema reference.
- Paths are **relative, normalized, and confined to the output directory**. A path escaping the
  output directory is refused rather than written.
- A declared `required: true` artifact that is absent at exit makes the run a failure, whatever the
  exit code said.
- An artifact written but not declared is retained and flagged in the execution record, not silently
  discarded — it is usually a manifest bug rather than an attack.
- Downstream consumers receive artifact references, not filesystem paths.

## Determinism

Given the same configuration and unchanged upstream data, two runs produce byte-identical artifacts.
Timestamps and per-run identifiers belong in the envelope, which is expected to differ, not in the
artifacts, which are not.

This is what lets a host detect real change by comparing hashes rather than re-reading content.

Where a plugin's output is inherently non-reproducible — a compiled binary, a container image — the
determinism requirement applies to its **reports** rather than to the output itself, and the plugin
states which artifacts are exempt and why.

## Serialization rules

Generic, and therefore here rather than in each plugin:

- UTF-8, LF line endings, one trailing newline.
- Stable property and collection ordering, so identical input produces byte-identical output. Sort
  explicitly; never rely on hash or insertion order.
- **Absent values serialize as `null` rather than being omitted.** A present `null` is deterministic,
  spares consumers from distinguishing "missing" from "unknown", and avoids the friction between
  optional properties and strict compiler settings.
- Where a plugin emits more than one format, all formats represent the same normalized data.
- Each document is replaced by a single atomic rename onto its live path. Directory replacement is
  not used: it is not atomic on Windows and fails when the destination exists.
- Numbers that are identifiers are strings. A 64-bit provider ID does not survive a round trip
  through a JSON number in every language.

## Configuration

- Resolution order: CLI option, environment variable, configuration file, built-in default. Secrets
  are excluded from this chain entirely and come from the environment only.
- Paths inside a configuration file resolve **relative to that file**, not to the working directory,
  so a configuration behaves identically wherever the plugin is invoked from.
- Configuration carries its own version and is schema-validated at startup, failing with exit `2`
  rather than proceeding on a partially understood file.

## Logging

Structured, to stderr, never to stdout. Levels are `error`, `warn`, `info`, `debug`, `trace`.

`info` is the level most often abused: if it fires per item rather than per operation, it is `debug`.

No log record at any level may contain a secret. Redaction covers authorization headers, known secret
field names, request errors, and nested causes — as a backstop, not as permission to log freely.

## Trust and distribution

Four trust levels, all establishable — see `adr/ADR-004` for the mechanism and
`SubZeroDev.Automator/10-security-model.md` for enforcement:

| Level              | Established by                                                                    |
| ------------------ | --------------------------------------------------------------------------------- |
| First-party        | Keyless signature from the organization's pinned release-workflow identity        |
| Signed third-party | Signature matching an identity or key in the operator's allowlist                 |
| Untrusted          | Unsigned, or signed by an identity nobody allowed                                 |
| Development-local  | An image or path the operator named; verification skipped and recorded as skipped |

Signatures are Sigstore cosign, over the image digest and the canonical-JSON manifest. Signing
operates on canonical JSON, never the authored YAML — otherwise a formatter run invalidates a
signature.

Distribution formats — OCI image, NuGet, npm, PyPI, PowerShell Gallery, archive, remote registration
— are separate from execution runtime. A plugin distributed as an npm package may still be executed
by the Docker host.

## Conformance

The contract is only real if it is mechanically checkable. `17-conformance.md` specifies the suite
and is authoritative for the check list; this is the summary of what it asserts:

1. `manifest` succeeds in a bare container and validates against the manifest schema.
2. `--help` and `--version` exit 0; an unknown command exits 2.
3. In JSON mode, stdout parses as exactly one JSON document — the check that catches a stray log
   line — and that document validates against the result-envelope schema.
4. The envelope satisfies the two rules the schema cannot express: `data` is at most 256 KiB
   serialized, and `finishedAt` is at or after `startedAt`.
5. Every declared artifact appears where declared and validates against its schema, and every
   `artifacts[]` entry in the envelope matches the file on disk in size and digest.
6. Declared exit codes are produced for their conditions, and the envelope's `exitCode` equals the
   process exit code.
7. A secret canary appears in no output, log, or artifact.
8. The image runs as non-root and tolerates a read-only config mount.
9. Two identical runs produce byte-identical artifacts.
10. Paths escaping the output directory are refused.

The suite is what makes "use this plugin as a template" verifiable rather than aspirational, and it
is the deliverable that turns this document from prose into a contract.

## Decisions on previously open points

**Does every plugin need a CLI?** Yes, for any plugin with a container or process runtime — the CLI
_is_ the normative surface. A remote-API-only plugin is exempt but must serve an equivalent manifest
endpoint and the same envelope and status semantics. Exempting CLIs more broadly would create a
second contract.

**Runtime-specific options.** A `runtimes[].options` object, schema-validated per runtime type and
never read by plugin code. The plugin should not know which host is running it; the host needs
somewhere to put its own settings.

**Signing.** Sigstore cosign, keyless by default. See `adr/ADR-004`.

**Multiple runtimes per command.** Allowed. Selection is deterministic: an explicitly requested
runtime wins; otherwise policy; otherwise the first runtime with enforcement level `enforced` in
**manifest order**. Manifest order is the tie-break, so the author controls it and the result never
depends on map iteration.
