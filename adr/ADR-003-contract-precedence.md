# ADR-003: The Contract Outranks Plugin Specifications

## Status

Accepted

## Context

The GitHub plugin was specified before the plugin contract existed. Its specification therefore
carried decisions that are not about GitHub at all: an exit-code table, secret-handling rules,
serialization rules, schema-version compatibility, configuration precedence, and logging levels.

Every one of those is a question a second plugin will face identically.

The cost of leaving them there was not theoretical. When the ecosystem draft later specified its own
exit-code table, the two disagreed — codes `3` and `5` were effectively swapped, so a host reading
exit `5` would have recorded an authentication failure as a partial success. Nothing would have
crashed; the data would simply have been wrong. That contradiction existed in the repository for
several commits before a review found it.

Two documents describing the same thing is not redundancy. It is a promise that they will diverge,
and a guarantee that nobody will notice which one is stale.

## Decision

**The plugin contract outranks every plugin specification.** Where a plugin document and the contract
disagree, the contract is correct and the plugin document has drifted.

Two rules follow.

**Generic decisions live in the contract.** A plugin specification contains only what is true of that
plugin and false of another. Anything a second plugin would face identically — how secrets arrive,
what stdout carries, which exit code means what, how artifacts are declared, what determinism
requires, how configuration resolves — belongs in the contract.

**Plugin specifications reference, never restate.** Copying a contract rule into a plugin document to
make it "self-contained" is what creates the second copy that drifts. A reference costs a reader one
click and costs the system nothing.

### The test for new decisions

When a decision arises, ask: _would a second plugin face this same question?_

If yes, it belongs in the contract, even when only one plugin exists to exercise it. If no, it
belongs in the plugin specification. Where the answer is genuinely unclear, the contract is the safer
home — a rule that turns out to be plugin-specific is easy to relax later, whereas a rule discovered
to be generic after three plugins have each answered it differently is a migration.

### Decisions promoted by this ADR

Moved out of the GitHub plugin's specification and ADR-002 into the contract:

| Decision                                                            | Now in                        |
| ------------------------------------------------------------------- | ----------------------------- |
| Exit-code table, `1` reserved                                       | Contract, Exit codes          |
| Secrets from the environment only; schema cannot represent one      | Contract, Invocation          |
| stdout is machine-only; logs to stderr                              | Contract, Channels            |
| Serialization: UTF-8, LF, stable ordering, `null` over absence      | Contract, Serialization rules |
| Atomic replacement by per-file rename                               | Contract, Serialization rules |
| Schema-version compatibility: same major, regenerate-only below 1.0 | Contract, Compatibility       |
| Configuration precedence, config-relative path resolution           | Contract, Configuration       |
| Logging levels                                                      | Contract, Logging             |
| Determinism as a testable requirement                               | Contract, Determinism         |

Kept in the GitHub plugin, because they are about GitHub:

repository scope and filters; identity on GitHub's numeric ID; capability-flag mapping to GitHub's
`has_*` fields; commit count via the `Link` header; the Search API's separate rate-limit bucket;
collection profiles and their request budgets; summary selection rules; portfolio overrides.

## Consequences

- The GitHub plugin's specification shrank substantially, and what remains is GitHub-specific.
- A second plugin inherits every generic decision without rediscovering it, which is the main point.
- Conformance can test contract rules, because they exist in exactly one place. A rule stated in two
  places cannot be authoritatively tested against either.
- Plugin authors must read two documents rather than one. Accepted: the alternative is a contract
  copy per plugin, each drifting independently.
- ADR-002 in the GitHub plugin is amended rather than rewritten. Its promoted decisions stay recorded
  there as history, marked, so the reasoning is not lost — an ADR records what was decided and why at
  a point in time, and editing that away would defeat the format.
