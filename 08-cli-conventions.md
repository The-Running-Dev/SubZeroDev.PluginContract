# CLI Conventions

Split from `08-powershell-and-cli.md`. The Automator PowerShell module and generated wrappers moved
to `SubZeroDev.Automator/08-clients.md`.

These conventions bind any plugin that exposes a CLI, which is the normative surface. They are part
of the contract, not advice — conformance tests most of them.

## Principle

Manual execution is a first-class use case, not a debugging affordance. A plugin that only works
under the Automator has failed ADR-004 and is much harder to develop and diagnose.

## Command shape

```text
<binary> <command> [options]
```

- POSIX-style long options: `--output-format`, not `/OutputFormat` or `-outputFormat`.
- Short aliases only for the universal ones: `-h`, `-v`.
- Command names are lowercase, hyphen-separated, stable, and match the manifest's command IDs.
- Subcommands nest at most one level. `plugin export markdown` is a command called `export` with a
  format option, not a two-level tree.

## Required options

| Option                         | Behavior                                                      |
| ------------------------------ | ------------------------------------------------------------- |
| `--help`, `-h`                 | Usage to stdout, exit 0. Also valid per command.              |
| `--version`, `-v`              | Plugin version to stdout, exit 0.                             |
| `--output-format <text\|json>` | Selects the output channel contract. Default `text`.          |
| `--json`                       | Alias for `--output-format json`.                             |
| `--config <path>`              | Configuration file location.                                  |
| `--log-level <level>`          | One of `error`, `warn`, `info`, `debug`, `trace`.             |
| `--quiet`                      | Suppress non-essential stderr output. Does not affect stdout. |
| `--dry-run`                    | Required for any command with side effects.                   |

### Resolving the output-format contradiction

The original set specified `--output-format text|json` in `08` while the plugin contract described a
`--json` mode. Those are the same thing described twice, and an implementer would have had to guess.

`--output-format` is canonical because it extends — YAML or a table format can be added without a
second boolean flag. `--json` remains as the short form people actually type.

**`--output-format json` means exactly this:** stdout carries the result envelope and nothing else.
Not a progress line, not a log record, not a trailing newline of human text. Everything else goes to
stderr. This is the rule conformance checks by parsing stdout as a single JSON document.

## Channels

| Channel | Carries                                                                            |
| ------- | ---------------------------------------------------------------------------------- |
| stdout  | The result envelope in JSON mode; the human summary in text mode; help and version |
| stderr  | Logs, progress, diagnostics, warnings — always, in both modes                      |

**Structured loggers default to stdout.** Pino does; so do several others. The logger must be
explicitly constructed against stderr, and this is the single most likely way to break every adapter
at once, because it presents as a JSON parse error rather than a logging bug.

## Exit codes

Defined once in `04-plugin-contract.md`. Plugins do not restate the table; they implement it.

## Interaction

- **No interactive prompts**, ever, unless explicitly requested by a flag. A host is never there to
  answer, and a prompt in a container is an indefinite hang rather than an error.
- **No TTY assumptions.** Colour and progress rendering are disabled when stdout is not a TTY, and
  always in JSON mode.
- `stdin` is read only when a command documents it.

## Configuration precedence

```text
CLI option → environment variable → configuration file → built-in default
```

Paths inside a configuration file resolve **relative to that file**, not to the working directory, so
a config behaves identically wherever the CLI is invoked from.

Secrets are excluded from this chain entirely: they come from the environment only. See the contract.

## Dry run

Any command with side effects supports `--dry-run`, which performs every read and validation, reports
exactly what would change, and writes nothing.

For a command that publishes to an external system, dry run should be the **default**, with execution
requiring an explicit flag. The Requirements Compiler's publishing commands are the current example.

## Shell completion

Plugins should be able to emit completion scripts for PowerShell, Bash, Zsh, and Fish. This is
optional and not conformance-tested.

## Decisions on previously open points

**JSON is never implied. It is always explicit.** Inferring from a non-TTY stdout is friendlier right
up until it is wrong — a plugin run through a pipe in a shell script would silently switch format,
and the failure surfaces as a parse error far from its cause. One flag is a small price for a mode
that never changes underneath anyone.

**A plugin may omit a CLI only if it is remote-API-only.** Any plugin with a container or process
runtime has one, because the CLI is the normative surface. A remote-API-only plugin must serve an
equivalent manifest endpoint and honour the same envelope and status semantics; what it is exempt
from is the argument syntax, not the contract.
