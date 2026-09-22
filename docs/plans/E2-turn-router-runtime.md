# E2 -- Turn router runtime

Acceptance: `grokbotpatcher turn` reads conversation context on stdin and
writes a response or tool-call request on stdout, correctly routes to either
a Codex-style provider or an OpenRouter-style HTTP provider, keeps per-Bot
state isolated and atomic, handles in-chat control commands, and suppresses
duplicate replays -- each backed by an authored acceptance test and, once a
first working version exists, a golden-fixture regression suite that locks
the behavior in.

Context: This is the behaviorally subtle half of the agent -- correctness
here depends on getting stable identity, replay suppression, and tool-schema
bridging right, not just calling a provider API. Because this is a greenfield
implementation there is no prior version to port tests from; every scenario
below must be authored from the requirement, then locked into a regression
suite (T2.10) so later changes cannot silently regress it.

fidelity: executable

## Wave A (no dependencies)

- [ ] T2.1 SPIKE: decide the Codex integration approach -- native HTTP/API client in Go, or a subprocess wrapper around an installed Codex CLI -- and record the decision and rationale as a short addendum to this file's Risks section  Owner: TBD  Est: 3h  verifies: [infrastructure]  acc: [this file has a resolved Codex-integration-approach note with a concrete implementation path, before T2.6 starts]
- [ ] T2.2 Scaffold the `turn` subcommand and define the stdin/stdout JSON protocol (config, transcript, tool schemas, stable Bot/agent/conversation identifiers in; text or tool-call JSON out) as Go structs, documented in this file  Owner: TBD  Est: 2h  verifies: [infrastructure]  acc: [a hand-written fixture stdin payload unmarshals into the Go structs with no field loss, and re-marshaling round-trips]

## Wave B (after Wave A)

- [ ] T2.3 Implement the per-Bot state model: one state file per Bot, stable-identity precedence (Bot ID outranks agent/conversation/channel scope), atomic write via temp-file-then-rename, short-lived per-Bot lock  Owner: TBD  Est: 3h  deps: [T2.2]  verifies: [UC-008]  acc: [two Bots sharing a channel retain independent provider/model state; concurrent control commands on the same Bot do not corrupt its state file]
- [ ] T2.4 Design and implement the control-command syntax (proposed: `/route provider`, `/route model`, `/route models`, `/route reasoning`, `/route doctor`, `/route reset`, plus an addressed-Bot form for group contexts), parsed before replay detection or provider inference  Owner: TBD  Est: 2.5h  deps: [T2.2]  verifies: [UC-009]  acc: [an addressed command changes only the named Bot's state; an unrecognized command or unlisted model ID is rejected deterministically, never passed to the provider]
- [ ] T2.5 Implement the OpenRouter-style bridge as a `net/http` client: message format conversion, tool schema translation to the provider's native function-calling format, provider-ID-to-internal-ID remapping, orphan/dangling tool-result handling  Owner: TBD  Est: 4h  deps: [T2.2]  verifies: [UC-010]  acc: [a fixture turn with an orphaned tool result and a dangling call produces a well-formed, sanitized upstream request with no orphaned or dangling entries]

## Wave C (after Wave B; T2.6 additionally depends on the Wave A spike)

- [ ] T2.6 Implement the Codex-style bridge per T2.1's decision, including its tool-call response schema and image-attachment handling with an explicit count/size cap  Owner: TBD  Est: 5h  deps: [T2.1, T2.3]  verifies: [UC-007, UC-010]  acc: [a fixture turn with an attachment and a pending tool call produces the expected Codex-facing request shape, and no tool is ever reported complete before the host actually returns its result]
- [ ] T2.7 Implement the replay/suppression state machine: a durable per-request signature so a repeated delivery of the same request runs inference exactly once, scoped narrowly enough that it never suppresses a genuinely new control command or tool round  Owner: TBD  Est: 4h  deps: [T2.3]  verifies: [UC-011]  acc: [a fixture set replaying the same request 3 times produces exactly one provider call; an interleaved new control command in the same fixture set is still processed]
- [ ] T2.8 Implement redacted audit logging: bounded event names, counts, provider/model receipts, suppression reasons, no raw provider error bodies or credential-shaped strings  Owner: TBD  Est: 1.5h  deps: [T2.3]  verifies: [infrastructure]  acc: [an audit line for a request carrying a well-formed API-key-shaped string has that string redacted; a raw provider 5xx body is never written verbatim to the log]

## Wave D (tests and validation, after Wave C)

- [ ] T2.9a Author acceptance tests for control-command parsing and addressed-Bot scoping  Owner: TBD  Est: 2.5h  deps: [T2.4]  verifies: [UC-009]
- [ ] T2.9b Author acceptance tests for per-Bot identity and state isolation (shared channel, concurrent controls)  Owner: TBD  Est: 2.5h  deps: [T2.3]  verifies: [UC-008]
- [ ] T2.9c Author acceptance tests for replay suppression and interleaved-new-turn handling  Owner: TBD  Est: 3h  deps: [T2.7]  verifies: [UC-011]
- [ ] T2.9d Author acceptance tests for the OpenRouter-style bridge (tool translation, orphan/dangling handling, empty-response retry)  Owner: TBD  Est: 3h  deps: [T2.5]  verifies: [UC-010]
- [ ] T2.9e Author acceptance tests for the Codex-style bridge (tool-call schema, attachment cap, no-premature-completion)  Owner: TBD  Est: 2.5h  deps: [T2.6]  verifies: [UC-007]
- [ ] T2.10 Freeze T2.9a-e's fixtures into a golden-fixture regression suite that runs in CI on every change, so future changes cannot silently regress behavior that was correct once  Owner: TBD  Est: 2h  deps: [T2.9a, T2.9b, T2.9c, T2.9d, T2.9e]  verifies: [infrastructure]  acc: [CI runs the full fixture corpus on every push and fails the build on any diff from the recorded expected output]
- [ ] T2.11 Run gofmt and go vet on cmd/grokbotpatcher and fix findings for the turn subcommand's packages  Owner: TBD  Est: 0.5h  deps: [T2.10]  verifies: [infrastructure]  acc: [gofmt -l reports no files under the turn-router packages; go vet is clean]

## Risks

- T2.1's decision blocks T2.6 and T2.9e; if a CLI-subprocess wrapper is
  required, re-estimate T2.6 upward and note the CLI as an on-device
  dependency this component introduces, distinct from ADR 002's already-
  accounted-for syntax-check dependency.
- Because there is no prior implementation to diff against, T2.9a-e's
  fixtures are the only source of truth for "correct" behavior until real
  usage exists -- write them from the use-case descriptions in
  `.claude/scratch/usecases-manifest.json`, not from assumptions about how a
  similar tool might behave.
