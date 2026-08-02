# Working in the plugin contract

This repository is the most consequential in the ecosystem and the most dangerous to edit carelessly.
Every plugin and the Automator depend on it; it depends on nothing. A change here is a change to a
published interface, not to a document.

## The normative artifact is the schema, not the prose

`04-plugin-contract.md` explains. `schemas/*.json` decides. When they disagree, the schema is what
implementations actually obey, so the prose is the bug — but fix both, because a contract whose prose
lies is worse than one with no prose.

This is why the result envelope existed as prose for as long as it did and was still a defect: the
contract states that the schemas are normative, so a normative artifact that did not exist was the
contract failing its own rule.

**A schema change is not done until it has been shown to reject something.** Validate under ajv
strict mode with positive _and_ negative cases, and record the counts in the commit message. The
manifest schema's `capabilities` block was `additionalProperties: true` for its whole first life —
the entire permission model rested on an object that accepted anything, and nobody noticed because
nothing had ever tried to make it fail.

## What belongs here

Anything a _second_ plugin would face identically. That is the test from ADR-003, and it is the only
test that matters for placement.

Already promoted here from the GitHub plugin, and not to be pushed back down: the exit-code table,
secrets from the environment only, stdout as machine-only, serialization rules, atomic replacement,
schema-version compatibility, configuration precedence, logging levels, determinism.

Not here: anything true of one plugin and false of another. GitHub's numeric repository ID, the
Search API's rate-limit bucket, a plugin's collection profiles.

Where the answer is genuinely unclear, **the contract is the safer home**. A rule that turns out to
be plugin-specific is easy to relax later; a rule discovered to be generic after three plugins have
each answered it differently is a migration.

## Invariants

These are not preferences. A change that breaks one of them is wrong regardless of what it enables.

- **Secrets travel by environment variable only.** Never `argv` — readable through `/proc` on Linux.
  Never a configuration file — that is the thing people commit. Never an MCP tool argument. The
  configuration schema must be _incapable_ of representing a raw token.
- **No secret reaches stdout, stderr, a log at any level, an artifact, the cache, an error message,
  or an image layer.** The conformance suite greps for a canary in all of them and a single
  occurrence fails.
- **stdout is machine-only** in JSON mode: exactly one document, nothing else. One log line on stdout
  corrupts the envelope and breaks every adapter simultaneously.
- **Exit `1` is reserved and never assigned.** Runtimes return it for uncaught exceptions, and "the
  plugin crashed" must stay distinguishable from "the plugin reported a failure" — retry policy
  depends on that difference.
- **Unknown fields are refused under `capabilities`, `secrets`, and `runtimes[].type`**, and ignored
  everywhere else. Failing open on the security surface is not forward compatibility.
- **Signing operates on canonical JSON, never the authored YAML.** Otherwise a formatter run
  invalidates a signature.
- **The plan-apply gate is structural.** An apply command accepts a plan token and nothing else. A
  prose instruction to stop and wait for approval is not a gate — a different MCP client's model
  never reads it, and an instruction injected into a plugin's input cannot fabricate a token.

## The exit-code table is canonical here and nowhere else

`04-plugin-contract.md` holds it. No other document restates it.

This is not tidiness. Two exit-code tables already existed in this project and disagreed — codes `3`
and `5` were effectively swapped, so a host reading exit `5` would have recorded an authentication
failure as a partial success. Nothing would have crashed. The data would simply have been wrong, and
it sat in the repository for several commits.

## Conformance is part of the contract

`17-conformance.md` is authoritative for the check list; `04`'s list is a summary that follows it.

Two envelope rules cannot be expressed in JSON Schema — the 256 KiB cap on `data` and `finishedAt`
ordering. They live in the suite as C3b or they live nowhere. When you add a rule the schema cannot
enforce, put it in the suite in the same commit, or it is decoration.

**Be sure it really cannot be expressed before delegating it.** A third rule — non-empty `errors` on
a failed or partial status — was sent to the suite on the assumption that a constraint conditioned on
a sibling field was inexpressible. It was not: the schema already branches on `status`, and the same
branch carries `minItems`. A rule pushed to the suite unnecessarily is a rule that every
schema-validating host silently skips.

**The suite needs fixtures that make it fail.** `leaky`, `noisy`, `nondeterministic`, and `traversal`
exist to prove the checks detect what they claim. A suite that passes everything it is pointed at is
not evidence.

## Conventions

These hold in every SubZeroDev specification repository. The canonical copy of this block is
`AGENTS.md` in the Architecture repository; it is repeated here because a repository has to stand on
its own.

- **Reference, never restate.** A rule that lives in another document is linked, not copied. Two
  copies of a rule is a promise they will diverge and a guarantee nobody will notice which is stale.
- **The plugin contract outranks plugin specifications.** Where a plugin document and the contract
  disagree, the contract is correct and the plugin document has drifted. See ADR-003 in
  `SubZeroDev.PluginContract`.
- **A decision gets an ADR.** Status is exactly one of `Proposed`, `Accepted`, `Superseded`, or
  `Deprecated`, under a `## Status` heading. An accepted ADR states its context, the decision, the
  consequences _including the costs_, and the alternatives it rejected and why. "Accepted in existing
  practice" is not a status — ratifying current practice is a note in the context.
- **Move, never copy.** A specification has exactly one home. Where another repository needs the
  text, it references a tagged commit rather than duplicating the file.
- **Give reasons.** These documents are read by people deciding what to build. An assertion with no
  reason cannot be evaluated, and cannot be safely revised by someone who was not there when it was
  written.
- **Markdown is Prettier-formatted**, 100 columns, LF endings.

## Versioning

The `$id` of each schema is version-pathed, so publishing `1.1` or `2.0` cannot overwrite a pinned
reference. Keep it that way.

A host accepts a manifest whose `schemaVersion` major matches its own, refuses a higher major with an
error naming both versions, and accepts a higher minor under the unknown-field rules. Any change that
would make an existing valid manifest invalid is a major.

## Before you finish

- Validate every changed schema under ajv strict mode, positive and negative cases.
- If a rule changed, check `17-conformance.md` — an unchecked rule is a suggestion.
- If the change promotes something out of a plugin specification, amend that plugin's document in the
  same commit and leave its ADR marked rather than rewritten. An ADR records what was decided at a
  point in time; editing that away defeats the format.
