# ADR 001: One static Go binary with subcommands, not separate patch/runtime/wrapper processes

## Status
Accepted

## Date
2026-09-22

## Context
The on-device agent that runs inside a Grok Bot's remote computer has two
distinct jobs: (1) verify and patch the host application so it hands
inference for a turn to this agent instead of its own stock model, and (2)
on each turn, receive the conversation/tool context and produce a response or
a tool-call request by calling an external model provider. It also needs
small operational surfaces: enable/disable, doctor/health-check, restore to
stock, repair after an unexpected host replacement, and periodic
registry-sync for newly trusted host fingerprints.

Splitting these into separate language runtimes or separate wrapper scripts
multiplies the number of things that must be present and version-correct on
a machine we do not control the lifecycle of. Every additional runtime
(interpreter, package manager) is a place a missing or mismatched dependency
can break an install with no way for us to intervene remotely.

## Decision
Ship one statically linked Go binary, `grokbotpatcher`, with subcommands:
`patch`, `doctor`, `restore`, `repair`, `enable`, `disable`, `registry-sync`,
`watchdog`, and `turn` (the per-turn inference process, invoked by the
injected code with conversation context on stdin and a response on stdout;
see [docs/adr/003-injection-marker-versioning.md](003-injection-marker-versioning.md)
for how it gets invoked). No bundled scripting language, no package manager,
no separate wrapper binaries per concern.

## Consequences
- The only external dependency this project's own components introduce is
  the Go toolchain at build time; the shipped artifact is self-contained.
- A JavaScript syntax-validation step is still required after patching,
  since the file being patched is the target application's own script, not
  ours -- see ADR 002 for why that specific step still shells out rather than
  embedding a parser.
- Subcommand dispatch (rather than five or more small binaries) keeps the
  operational surface easy to reason about and easy to test as one Go
  module, at the cost of a slightly larger single binary than any one
  concern would need alone -- an acceptable trade given the target
  environment's constraint is "things that can be missing or wrong," not
  binary size.
