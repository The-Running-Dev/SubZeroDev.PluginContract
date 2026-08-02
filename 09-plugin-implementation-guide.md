# Plugin Implementation Guide

| Field                     | Value                                                        |
| ------------------------- | ------------------------------------------------------------- |
| Owner                     | `SubZeroDev.PluginContract`                                  |
| Status                    | Accepted                                                     |
| Applies to contract       | `1.0.0-draft`                                                |
| Companion to               | `04-plugin-contract.md`, `08-cli-conventions.md`, `17-conformance.md` |

This guide is normative about the **shape** of a plugin's own build plan. It decides nothing about
plugin behavior — `04`, `08`, and `17` already do, and this document references them rather than
restating them. What it adds is the milestone spine, distilled from three plans that converged on it
independently: the GitHub plugin's, the Project Setup plugin's, and the Backlog plugin's. That
convergence is the evidence this spine is reusable rather than invented.

## 1. Purpose and audience

For whoever writes the next plugin's `BUILD-PLAN.md`. Copy the spine below into that document, keep
milestone numbers **local** to the plugin — phase numbers belong to
`SubZeroDev.Ecosystem/18-roadmap.md`, and a build plan that invents its own phase is a defect — and
delete what does not apply, recording every deletion in a **Deferred** list with a reason. A milestone
silently dropped is a scope cut nobody approved.

This is not a second contract. Where this guide and `04-plugin-contract.md` disagree, the contract is
correct and this document has drifted.

## 2. What the contract already decides

A plugin author does not re-decide any of the following. State the choice as a link, not a
restatement — the failure mode this guide exists to prevent is two documents each carrying an
exit-code table that quietly disagree.

| Decision                                                            | Owned by                             |
| --------------------------------------------------------------------- | --------------------------------------- |
| Exit codes, and that `1` is reserved and never assigned              | `04-plugin-contract.md`, Exit codes   |
| Secrets from the environment only; the schema cannot represent one  | `04-plugin-contract.md`, Invocation   |
| stdout is machine-only; logs go to stderr, at every level            | `04-plugin-contract.md`, Channels     |
| The result envelope, and its schema                                   | `result-envelope.schema.json`         |
| Serialization: UTF-8, LF, stable ordering, `null` over omission      | `04-plugin-contract.md`, Serialization |
| Atomic replacement by per-file rename, never a directory swap        | `04-plugin-contract.md`, Serialization |
| Schema-version compatibility: same major accepted, higher refused    | `04-plugin-contract.md`, Compatibility |
| Configuration precedence, and config-relative path resolution        | `08-cli-conventions.md`               |
| Logging levels                                                        | `04-plugin-contract.md`, Logging      |
| Determinism as a testable requirement                                 | `04-plugin-contract.md`, Determinism  |
| Manifest shape, capabilities, and the trust levels                    | `plugin-manifest.schema.json`, ADR-004 |
| The plan-apply pattern for writes to external systems                | `04-plugin-contract.md`, Plan-apply   |
| Required CLI options, output-channel purity, no interactive prompts  | `08-cli-conventions.md`               |
| The conformance checks, C1 through C9                                 | `17-conformance.md`                   |

## 3. The placement test

Before writing a new rule into a plugin's own specification, ask: **would a second plugin face this
same question?** Yes — even while only one plugin exists to exercise it — put it in the contract.
Genuinely unclear — the contract is still the safer home, because relaxing a rule that turns out to
be plugin-specific is cheap, and discovering a rule was generic only after three plugins answered it
three different ways is a migration. (`SubZeroDev.PluginContract/adr/ADR-003`.)

## 4. Repository layout and toolchain

The skeleton is fixed, per `16-repository-layout-and-packaging.md` in the Architecture repository:

```text
src/
tests/
docs/
schemas/
examples/
plugin.yaml
Dockerfile
README.md
CHANGELOG.md
AGENTS.md
CLAUDE.md
BUILD-PLAN.md
```

`AGENTS.md` carries the repository's working invariants; `CLAUDE.md` points at it rather than
repeating it. `plugin.yaml` is authored in YAML and canonicalized to JSON for validation and signing.

The language-specific slots are the author's choice, mapped once per runtime family:

| Runtime    | Package manifest | Lockfile           | Entry point        | Test runner | Line-ending pin           |
| ---------- | ------------------ | ------------------- | -------------------- | ------------- | --------------------------- |
| Node       | `package.json`    | `package-lock.json` | `src/cli.ts`        | Vitest      | `.gitattributes`, LF       |
| Python     | `pyproject.toml`  | lockfile per tool   | `__main__.py`       | pytest      | `.gitattributes`, LF       |
| .NET       | `.csproj`         | `packages.lock.json` | `Program.cs`        | xUnit       | `.gitattributes`, LF       |
| PowerShell | module manifest    | n/a                 | exported cmdlet     | Pester      | `.gitattributes`, LF       |

**If a reference implementation exists for your language family, mirror its toolchain rather than
choosing again.** The point of a reference implementation is that the second plugin does not
re-decide what the first one already settled. `SubZeroDev.Plugins.GitHub` is the current Node
reference; `SubZeroDev.Plugins.Backlog` is the current Python reference. Where none exists, choose and
record the choice in an ADR.

## 5. The spine

| Milestone | Intent                                    | Applies when                                          |
| ----------- | -------------------------------------------- | --------------------------------------------------------- |
| M0        | Ground and the contract surface           | Always                                                    |
| M1        | Configuration, inputs, `validate`         | Always                                                    |
| M2        | The local/offline half, completely        | Always                                                    |
| M3        | Plan store and the approval gate          | Only if the plugin writes outside its own storage        |
| M4a       | Remote read adapter                        | Only if the plugin reads a remote API                    |
| M4b       | `plan` command                             | Only if M3 applies                                        |
| M5        | `apply`, the remote write                  | Only if M3 applies                                        |
| M6        | Live round-trip verification               | Always                                                    |
| M7        | Conformance, signing, release              | Always                                                    |

This table is the first of four places the conditional milestones are stated — see §11.

### M0 — Ground and the contract surface

Toolchain and repository layout in place; `plugin.yaml` validating against
`plugin-manifest.schema.json`; `manifest` succeeding in a **bare container** — no configuration file,
no secrets, no network, no mounts; result envelope emission with `--output-format json`; the logger
explicitly constructed against **stderr**; exit codes wired through; text line endings pinned to LF so
formatting checks agree across platforms; a cross-platform CI matrix with `fail-fast: false`.

The manifest and the envelope come first, deliberately, even though neither is the product a user
wants. Conformance depends on both, and retrofitting an envelope through commands that are already
finished is worse than starting every command with one.

**Exit:** `manifest` runs in a bare container and validates against the schema; stdout carries
exactly one JSON document with logging forced to `trace`; the full check suite is green on every
supported platform; no configuration is read before command dispatch.

### M1 — Configuration, inputs, and `validate`

Configuration versioned and schema-validated at startup, failing exit `2` on anything malformed or
incompatible; precedence and config-relative path resolution per `08-cli-conventions.md`; a
configuration schema **incapable** of representing a raw secret; secret redaction covering
authorization headers, known secret field names, request errors, and nested causes; `validate` doing
every check the plugin can do and none of the work. Domain schemas, where the plugin has them, land
here too, with identifiers that are provider identity serialized as strings.

**Exit:** malformed and incompatible configuration produce stable messages and exit `2`; no
configuration path can carry a token value; a secret canary reaches no stdout, stderr, log record, or
serialized error.

### M2 — The local/offline half, completely

Everything the plugin can do with no network access, finished **completely** before any remote work
starts. It needs no credentials, it is where feedback arrives fastest during development, and for
many plugins it is the half used every day. The guide owns only two rules here; the content is the
author's.

**Exit:** a second run of anything in this half is a no-op; a destructive option is refused against a
dirty working tree; nothing in this half requires a token.

### M3 — Plan store and the approval gate

**Skip this milestone if the plugin writes to nothing outside its own cache and output. Record the
skip in Deferred, with the condition that would reverse it.** See §11 for the complete read-only rule.

Where it applies: a plan keyed by an **opaque random identifier**, never a content hash; the plan
stores its target, the desired and observed state, the action list, a state fingerprint, and
timestamps; a time-to-live; single use; eviction on apply regardless of outcome, because a partially
applied plan is stale by definition; **four distinct refusal paths** — unknown, expired, already used,
fingerprint mismatch — each with its own message; no credential ever written into the plan.

The fingerprint check is the one most often skipped and the one that matters: between plan and apply,
someone may have changed the target by hand, and applying a stale diff over their change is worse than
refusing outright. The gate must be **structural**, not a prose instruction — a different MCP client's
model never reads a comment telling it to wait for approval, and an instruction injected into a
plugin's input cannot fabricate a plan token.

**Exit:** a plan cannot be applied twice, late, or against changed state, and each refusal names
which; no plan file contains a credential.

### M4a — Remote read adapter

Applies to any plugin that reads a remote API, independent of whether M3 applies. Client construction
from the environment-resolved secret; an authenticated connectivity check; pagination; a central
request wrapper carrying conditional requests, rate-limit capture, retries with jitter, error
classification, and redaction; bounded concurrency, starting conservative.

**Exit:** provider-specific types stay confined to the adapter module and never escape into a domain
model; every item is discovered exactly once; errors carry context without secrets.

### M4b — `plan`

**Skip exactly when M3 is skipped.** Desired-versus-observed diff; an action list; a rendering a human
can review, with anything dangerous marked unmissably.

**Exit:** planning an already-correct target yields zero actions; the rendering names every action
before it is taken, not after.

### M5 — `apply`, the remote write

**Skip exactly when M3 is skipped.** `apply` takes **only** the plan identifier — no target, no
content, nothing that would let it act without a plan. Partial failure is exit `4` with a populated
`errors[]`, not an uncaught exception, and the plan is consumed either way. Non-idempotent operations
are never retried automatically; a retry confirms the previous attempt is actually dead first, because
a container stop is asynchronous and retrying blind can run two copies concurrently.

**Exit:** a throwaway target is created and configured end to end; a partial failure reports
accurately and does not replay; `apply` is structurally unreachable without a plan identifier.

### M6 — Live round-trip verification

**Nothing before this milestone proves the plugin works.** Fakes and mocks are written from the same
reading of the API as the implementation, so they agree with any misreading of it — only a real
target catches that class of bug.

Run the full lifecycle against one real, throwaway target. The **second run must be entirely a
no-op** — anything else is a convergence bug to find now, not later. Verify results in the provider's
own interface, not only through the plugin's own output. Exercise a target that already exists and
differs from desired state, to confirm reconciliation rather than only the empty-state path. Fold
every correction this surfaces back into the fixtures used by earlier milestones' tests.

**Exit:** a real target converges, and the second run against it is empty; every correction found here
is committed back into the fixtures; the observed request count matches the documented budget, or the
budget is corrected to match reality.

### M7 — Conformance, signing, and release

All conformance checks pass — C1 through C9 in `17-conformance.md` — with **skips reported as skips,
never as passes**. Container smoke test exercising `manifest`, `validate`, and at least one
fixture-backed work command. Secret canary absent from output, logs, artifacts, cache, errors, **and
image layers**. Signed image and signed manifest attestation, computed over canonical JSON, never the
authored YAML. Documentation complete. Publish.

Conformance gates publication: a plugin that fails conformance and ships anyway makes the whole
contract advisory rather than binding.

**Exit:** the full check suite is green on every supported platform; the image runs both under its own
UID and under a host-user override; an unchanged repeat produces byte-identical artifacts.

## 6. Milestone shape

```markdown
## Milestone N — <verb phrase>

<optional one to three lines of prose explaining why this milestone exists or is ordered here>

- [ ] task, imperative, one line
- [ ] **bolded task** where it is the load-bearing one

**Exit:** <semicolon-joined observable end states, phrased as facts about the system, not
activities, with at least one negative assertion>
```

Use `**Exit:**` uniformly. (One of the three source plans used `**Gate:**` for the identical purpose;
this guide picks one label so a reader comparing plans across plugins is not left wondering whether
the two words mean different things.)

An exit line you cannot observe is a wish, not a criterion. "Implemented X" is not an exit line;
"a second run of X is a no-op" is. The negative assertion is what catches an abstraction that has
quietly become decorative — a criterion with no negative half tends to pass even when something
important silently stopped being true.

Half-numbered milestones (`M3.5`, `M2.5`) are sanctioned for exactly two reasons: inserting a
de-risking vertical slice out of dependency order, or extracting a shared library at the point its
second consumer is already specified. Not for anything else — renumbering later moves the PR-sequence
table and every cross-reference into the plan, and a plan that collides its own local numbering with
the roadmap's phase numbers has already happened once.

## 7. Ordering, one PR per milestone, and the closing sections

**Default ordering:** contracts first, local before remote, the gate before any write, live
verification last. State deliberate departures from that default explicitly, with the reason — an
unexplained departure reads as an accident. A plan with nothing runnable until its final pull request
is itself a defect this ordering exists to avoid; M6 (or an earlier de-risking slice) is what
prevents it.

**One PR per milestone**, each carrying the tests for its own exit criteria and leaving the full
check suite green. Close the plan with a PR-sequence table.

**Definition of done** states end states for the whole plan, not a restatement of the milestones.
Every plugin's includes: contract conformance passes; the plugin runs standalone with no host
present; the manifest declares no capability its commands do not exercise; a second run of anything is
a no-op, or the command is declared non-idempotent with its condition stated; documentation takes a
new user from credential setup to a validated result.

**Standing constraints** — always true, restated in no single milestone. Every plugin's includes:
never log or echo a secret, in any output, including error messages; phase numbering belongs to the
roadmap, milestone numbers stay local; no command and no projected MCP tool takes a credential
parameter; a write command is never callable without a plan token. Add domain-specific invariants —
ordering guarantees, what may never be deleted, which files are ported and must stay ported.

**Deferred** lists every omission, one line each, with a reason and — where relevant — what would
bring it back. "Not yet" with no stated reason is a silent scope cut. The read-only skip from §11 is a
required entry here for any plugin that takes it.

## 8. Evidence that may not be edited

When a ported test, a golden fixture, or a byte-stability check fails after a refactor or an
extraction: **the change broke behavior. That is the signal to stop, not to update the test.** A test
edited to make it pass is evidence destroyed, not a fix.

## 9. Mapping — this spine against the plans it was distilled from

| Spine | GitHub plugin  | Project Setup | Backlog |
| ------- | ---------------- | ---------------- | --------- |
| M0    | M0              | M0             | M0      |
| M1    | M1, M2          | M1             | M1      |
| M2    | M3, M3.5        | M2             | M2      |
| M3    | *(blank — read-only)* | M3       | M3      |
| M4a   | M3              | M4             | M4      |
| M4b   | *(blank — read-only)* | M4       | M4      |
| M5    | *(blank — read-only)* | M5       | M5      |
| M6    | M3.5 (partial), live verification folded into later milestones | M6 | M6 |
| M7    | M8              | M7             | M7, M8  |

The GitHub plugin's blank M3/M4b/M5 cells are the worked read-only instance; Project Setup and Backlog
are the worked gated instances. This table exists so this guide reads as a distillation of three
independent plans, not an invention imposed on them.

## 10. How these plans fail

Failure modes actually observed, so a reviewer knows what to look for:

- A plan restates a contract rule and then disagrees with it — the two exit-code tables that swapped
  the meanings of `3` and `5` before anyone noticed.
- An exit criterion CI cannot enforce, so it is asserted in prose rather than checked by a test.
- Two documents in the same commit number the same milestones differently.
- An `**Exit:**` line stating intent ("supports X") instead of an observation ("a second run of X is
  a no-op").
- A fixed request budget that reality contradicts, left uncorrected instead of folded back.
- Nothing end-to-end runnable until the very last pull request in the sequence.

## 11. Read-only plugins

The complete skip rule, stated once here in addition to the three places above it is mentioned, on
purpose — a rule stated only in an overview table is a rule a plan author skims past.

**A plugin whose manifest declares no write capability against an external system skips M3, M4b, and
M5. It keeps M4a. Record the skip in Deferred, with the condition that would reverse it** — typically
"if this plugin gains a write command."

Three qualifications that are easy to get backwards:

- Such a plugin still writes its own cache and output, so those commands are still side-effecting —
  **`--dry-run` remains required** for them per `08-cli-conventions.md`.
- Conformance check C5's partial-success and rate-limiting conditions will report as **skipped** for a
  plugin that never enters those states from outside. A report showing six skips and three passes is
  not a pass by count; skips are reported prominently for exactly this reason.
- The gate cannot be retrofitted around commands that are already finished, which is why M3 is
  sequenced before M4 in the spine even for a plugin that turns out not to need it — deciding late
  costs more than deciding early and skipping.

`SubZeroDev.Plugins.GitHub` is the worked read-only instance of this guide. `SubZeroDev.Plugins.ProjectSetup`
is the worked instance that takes the plan-apply milestones in full.
