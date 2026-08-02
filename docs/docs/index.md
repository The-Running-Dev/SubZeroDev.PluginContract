---
title: 'SubZeroDev.PluginContract'
description: 'The SubZeroDev plugin contract: manifest and result-envelope schemas, CLI conventions, and conformance.'
sidebar_position: 1
---

# SubZeroDev Plugin Contract

What a plugin is, what it must do at the process boundary, and how to check that it does.

Every plugin and the Automator depend on this repository. It depends on nothing — which is why it is
its own repository rather than a folder inside Platform or the architecture repository, and what
lets a plugin pin a contract version.

## Contents

| Path                                  | Covers                                                         |
| ------------------------------------- | -------------------------------------------------------------- |
| `04-plugin-contract.md`               | The contract itself                                            |
| `08-cli-conventions.md`               | Command-line conventions plugins follow                        |
| `09-plugin-implementation-guide.md`   | The reusable milestone spine for a plugin's own build plan      |
| `17-conformance.md`                   | The suite that decides whether a plugin satisfies the contract |
| `schemas/plugin-manifest.schema.json` | **Normative** manifest schema                                  |
| `schemas/result-envelope.schema.json` | **Normative** result-envelope schema                           |
| `examples/plugin.yaml`                | Reference manifest                                             |
| `adr/`                                | The four decisions this contract rests on                      |

## What a plugin is

A capability defined by its manifest and its command contract. The runtime that delivers it —
Docker, .NET, Node, Python, PowerShell, a process, a remote endpoint — is an implementation detail
declared in `runtimes[]`.

The contract is a **process boundary**: the commands a plugin accepts, the JSON it writes to stdout,
the exit code it returns, and the artifacts it leaves behind. That is the only surface every language
already shares. Expressing this as a C# or TypeScript interface would make "plugins in any language"
false the moment it was written.

It follows that **the JSON Schemas are the normative artifact, not any generated types.** Types may
be generated from the schemas, and the schemas may be authored in Zod or similar, but what an adapter
in another language consumes is the committed, versioned schema file.

## Plugins run standalone

The Automator is an integration layer over this contract, never a prerequisite for it. Whatever the
Automator can invoke, a person can invoke from a terminal, with the same commands and the same
envelope on stdout. If a change makes a command meaningless outside the Automator, the change is
wrong — see `adr/ADR-002`.

## The decisions

| ADR                                | Decides                                                        |
| ---------------------------------- | -------------------------------------------------------------- |
| `ADR-001-plugin-is-capability`     | A plugin is a capability, not an image                         |
| `ADR-002-manual-plugin-execution`  | Plugins run standalone; the Automator is optional              |
| `ADR-003-contract-precedence`      | This contract outranks every plugin specification              |
| `ADR-004-plugin-signing-and-trust` | Sigstore keyless signing; the manifest as a signed attestation |

## Status

Specification and schemas are written; both schemas validate under ajv strict mode. The conformance
suite is specified but not implemented — it is work package W2.1, and the contract is prose until it
exists.
