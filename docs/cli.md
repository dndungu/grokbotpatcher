# CLI design

One static binary, `grokbotpatcher`, dispatching to subcommands via the
standard library `flag` package (each subcommand gets its own
`flag.NewFlagSet`; no third-party CLI framework, per this project's Go
conventions). This document is the interface contract E1/E2/E3's tasks
implement against; update it in the same change as any flag/output change.

## Global conventions

- Invocation: `grokbotpatcher <subcommand> [flags]`.
- `-h` / `--help` on the bare binary or any subcommand prints usage and
  exits 0.
- `--version` on the bare binary prints the build version and exits 0.
- `--json` (supported by every subcommand that produces output) switches
  from human-readable text to a single-line JSON object on stdout, for
  scripting and for the delivery/installer layer (E4) to consume without
  screen-scraping text.
- `--config <path>` overrides the default config file location. Default is
  platform-specific and defined once E3's real deployment path is known
  (T3.4); until then, tasks may assume an environment-variable override
  (`GROKBOTPATCHER_CONFIG`) for local development and tests.
- `--quiet` suppresses non-error human-readable output; has no effect with
  `--json`, which is already minimal.
- Every subcommand's human-readable output goes to stdout; errors go to
  stderr regardless of `--json`, so `--json | jq` never has to filter noise.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Operation refused by a verification gate (fail-closed: unknown host, anchor mismatch, reconstruction mismatch, signature mismatch) -- this is an expected, correct outcome, not a crash |
| 2 | Usage error (bad flags, missing required argument) |
| 3 | Unexpected internal error (I/O failure, corrupt state file, provider unreachable where retry is not applicable) |

A caller (including E4's installer) can distinguish "the tool correctly said
no" (1) from "the tool broke" (3) without parsing text.

## Subcommands

### `grokbotpatcher patch`

Verify and patch the target host file. See E1/UC-001, UC-002, UC-003.

- `--manifest <path>` (required) -- path to the manifest describing trusted
  stock-host fingerprints and required injection anchors for this target
  version.
- `--host <path>` (default: the platform's known host file location, once
  defined) -- path to the file to verify and patch.
- `--backup-dir <path>` (default: the platform's persistent backup
  location) -- where the verified stock backup is written.
- `--dry-run` -- run every verification step and report the outcome without
  writing anything. Useful for `doctor`-style pre-flight checks and for
  T1.9/T1.10's test harness.
- `--allow-unknown-host` -- bypasses the stock-hash allowlist check. Exists
  only for test fixtures in E1/E2's authored test suites; the binary refuses
  to run with this flag set unless the environment variable
  `GROKBOTPATCHER_TEST_MODE=1` is also set, so it can never appear in a real
  install command by accident (mirrors the fail-closed principle: an escape
  hatch that requires two independent, deliberate signals to activate).
- Exit 0: patched and activated. Exit 1: a verification step refused (the
  human/JSON output names which one: hash, byte count, anchor, foreign
  marker, or reconstruction mismatch).

### `grokbotpatcher doctor`

Report health without modifying anything. See E1/UC-004, UC-005.

- No required flags. `--manifest <path>` optional, to check against a
  specific manifest instead of the currently active one.
- Reports: whether the live host carries a recognized marker, whether it
  reconstructs exactly against the verified backup, whether a verified
  backup exists at all, and the routing-enabled/disabled state from config.
- Always exits 0 (a health report that found problems is still a
  successfully completed report); problems are surfaced in the output body,
  not the exit code. `--json` output includes a `healthy: bool` field for
  scripted checks.

### `grokbotpatcher restore`

Copy the verified backup over the live host and disable routing. See
E1/UC-004.

- `--confirm` (required) -- restore is destructive to the current (patched)
  state; the flag exists so this can never be invoked by a typo or a copied
  command fragment.
- Exit 1 if no verified backup exists (nothing to restore to -- this is a
  refusal, not a crash).

### `grokbotpatcher repair`

Reapply the patch only if the live host exactly matches an allowlisted
stock build. See E1/UC-005. No flags beyond the globals; designed to be
safe to run unattended (by `watchdog`) or by a human.

### `grokbotpatcher enable` / `grokbotpatcher disable`

Toggle the routing-enabled flag in config without touching the patched host
file at all -- the fastest, lowest-risk rollback lever short of a full
`restore`. No flags beyond the globals.

### `grokbotpatcher registry-sync`

Fetch and verify a signed registry of additional trusted host fingerprints.
See E1/UC-006.

- `--registry-url <url>` (default: the project's published registry
  endpoint, TBD).
- `--pubkey <path>` (default: the pinned public key bundled with the
  binary).
- Exit 1 on a signature verification failure (refusal, not a crash) or an
  unreachable endpoint after retry.

### `grokbotpatcher watchdog`

Long-running loop that periodically calls the same logic as `repair` and
`registry-sync` (at most once per interval for the registry check). See
E1/UC-005, UC-006.

- `--interval <duration>` (default: `1h`, matching the "at most once per
  hour" registry-check cadence this project inherits as a design principle
  for not hammering an external endpoint).
- `--once` -- run a single iteration and exit, for testing and for
  invocation from an external scheduler instead of running as a persistent
  process.

### `grokbotpatcher config get <key>` / `grokbotpatcher config set <key> <value>`

Read or write a single config value (`provider`, `model`, `reasoning`,
`enabled`). See E2/UC-008, UC-009's underlying state.

- `config set` validates the value against the known provider/model catalog
  before writing; an unrecognized value is a usage error (exit 2), not a
  silently accepted one.
- This is the CLI-level equivalent of the in-chat control commands defined
  in E2 (`/route provider`, `/route model`, ...) -- both paths write through
  the same per-Bot state model (T2.3), so a maintainer can inspect or fix a
  Bot's state directly without going through chat.

### `grokbotpatcher turn`

The per-turn inference process. Not intended for direct human invocation --
this is what the injected payload spawns (E3/T3.1). Reads the turn context
(config, transcript, tool schemas, stable identifiers) as a single JSON
document on stdin; writes a single JSON document (text or a tool-call
request) to stdout. No flags. Errors are reported as a JSON error object on
stdout with a non-zero exit, never as a bare stack trace, since the caller
here is the injected payload, not a human terminal.

## What deliberately has no CLI flag

- There is no `--force` flag anywhere that bypasses a verification gate
  silently. The one bypass that exists (`patch --allow-unknown-host`) is
  gated behind an environment variable specifically so it cannot be reached
  by flag alone (see above).
- There is no flag to disable the post-patch syntax validation step (ADR
  002). It always runs.

## Open questions for implementation (not blocking planning)

- Exact default paths for `--host`, `--backup-dir`, and `--config` depend on
  confirming the target platform's actual directory layout during E1/E3 --
  tracked as part of T1.1 and T3.4, not decided here.
- Whether `watchdog` runs as a user-level service (systemd unit, launchd
  agent, or platform-appropriate equivalent) or is simply invoked by an
  external scheduler is a T3.5/E4 delivery-mechanism decision, out of this
  document's scope.
