# ADR-002: Plugins Run Standalone; the Automator Is Optional

## Status

Accepted

Originally drafted as ADR-004 in a single global sequence, renumbered when the specifications split
by destination repository. Numbering is per repository.

## Context

The draft phrased this as an aspiration — plugins "should be manually executable wherever
practical" — which is not a decision anyone can be held to. "Wherever practical" is exactly the
clause that erodes: the first plugin that needs a queue, an execution record, or a credential broker
takes it from the host, and standalone execution quietly becomes a mode that used to work.

It erodes in a specific direction. A plugin that assumes an orchestrator will retry it stops handling
its own failures. A plugin that assumes the host resolves its secrets stops reading the environment.
Each is individually reasonable and collectively fatal to running the thing by hand.

The GitHub plugin made the stakes concrete. It was specified as an Automator plugin, and the
dependency ran through its documentation, its naming, and its build plan — while the actual need was
to run it manually against a personal account, with the Automator arriving later if at all.

## Decision

**Every plugin is fully usable from a terminal with no host present. The Automator is an integration
layer over the same contract, never a prerequisite for it.**

Concretely:

- The CLI is the primary surface. Whatever the Automator can invoke, a person can invoke, with the
  same commands, the same options, and the same envelope on stdout.
- Secrets come from the environment, so `export`-then-run works exactly as a broker-injected
  environment does. There is no host-only credential path.
- Configuration is a file the plugin reads itself, resolved relative to that file, so a working
  directory does not change behavior.
- A plugin handles its own retries, timeouts, and partial failures, and reports them in the envelope.
  It never assumes something upstream will re-run it.
- The manifest is obtainable without a host, and without running the plugin's real work.

**The test:** if a change makes a command meaningless outside the Automator, the change is wrong. The
host may add scheduling, history, approvals, and credential brokering _around_ a run. It may not
become part of what a run needs to succeed.

## Consequences

- The CLI contract carries the weight the draft assigned to the host, so it must be stable, versioned
  and documented for humans, not only for adapters.
- Development and debugging need no orchestrator, which is the practical reason this holds up: the
  standalone path is the one authors use daily, so it stays working without discipline.
- CI can invoke a plugin directly, and does — the same binary, no host in the loop.
- Some duplication is accepted. Retry and timeout handling exists in the plugin and again in the
  Automator, at different scopes. That is the price of the plugin being complete on its own.
- The Automator cannot introduce a feature that plugins depend on. Anything it offers must degrade to
  the standalone path, which constrains its design in exchange for keeping this decision true.

## Alternatives considered

**Host-first, with a standalone escape hatch.** Would let the Automator own retries, secrets, and
scheduling once instead of per plugin. Rejected: the escape hatch is the path nobody exercises, so it
is the path that breaks, and the first plugin's actual requirement was to run without a host at all.

**Standalone "wherever practical", as drafted.** Rejected as unenforceable. It states a preference
where a constraint is needed, and no reviewer can point at a diff and say it violates a preference.
