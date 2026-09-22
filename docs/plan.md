# grokbotpatcher: a small, version-resilient Grok Bot model router

## 1. Context

Grok Bot lets a user delegate work to a "Bot" that gets its own remote
computer, files, browser, and tool access. `grokbotpatcher` is an on-device
agent that lets each Bot choose which model backend actually answers a turn
-- a Codex-style provider or an OpenRouter-style HTTP provider -- while Grok
Bot itself remains the interface: chat UI, conversation history, computer,
files, browser, and tool execution/permissions are all untouched. The agent
only decides who answers, translates that turn's context and tool schemas to
and from the chosen backend, and hands the answer back to the same
conversation.

### Objectives

1. Ship the on-device agent as a single statically linked Go binary, so the
   target machine needs nothing this project does not already ship itself,
   beyond what the target application requires to exist regardless (ADR 002).
2. Fail closed by construction: never modify a host build that cannot be
   verified exactly; never trust a version marker alone when reconciling an
   upgrade; never let a provider invent a tool call the host did not offer.
3. Make every patch reversible: a verified backup always exists, and restore
   is atomic.
4. Add an LLM-assisted maintainer tool that proposes updated injection points
   as a reviewable diff when the target application updates, so version
   support does not depend on manual diffing alone -- never something that
   patches a live install unreviewed.
5. Ship as a staged, gated rollout against real installations, not a
   big-bang release -- see E3.

### Non-goals

- A general-purpose Grok Bot automation framework. This project's scope is
  narrowly: verify, patch, route, restore.
- A socket- or daemon-based IPC design. The injected payload spawns the
  per-turn process directly with context on stdin and a response on stdout;
  see E3.
- Running the LLM-assisted anchor-discovery tool (E5) unattended against a
  real installation. It only ever produces a maintainer-reviewable diff.
- A specific delivery/transport mechanism for getting the agent onto a
  user's machine is deliberately left open (E4, outline fidelity) until the
  target application's actual installable surface is confirmed hands-on.

### Constraints and assumptions

- `kazi` is on PATH in this repo; engineering tasks below carry `acc:` lines
  for kazi's just-in-time predicate lane.
- Grok Bot's desktop host process is assumed to be Electron/Node-based (see
  ADR 002); this is a working assumption to confirm during E1/E3, not a
  verified fact at planning time.
- No manifest of real target-application source anchors exists yet -- it
  cannot, until a real reviewed build is available to inspect (T3.3). Every
  task before E3's Wave A uses synthetic fixtures.
- This repo has no `docs/design.md`, `docs/devlog.md`, or existing
  `docs/roadmap.md`; `docs/roadmap.md` is created as part of this write.
- Licensed Apache License 2.0 (see [LICENSE](../LICENSE)), public/open
  source, per the repo owner's direct decision on 2026-09-22.

### Success metrics

- `grokbotpatcher` builds as a single static binary with no runtime
  dependency this project introduces beyond the syntax-check step in ADR 002.
- Every use case in `.claude/scratch/usecases-manifest.json` has a passing
  authored acceptance test (tracked via T1.9, T1.10, T2.9a-e) and, for the
  turn router, a green golden-fixture regression suite (T2.10).
- T3.3's real-manifest validation and T3.6/T3.7's staged rollout both have a
  recorded, directly-observed pass -- not an assumed one -- before the
  routing capability defaults to enabled for a fresh install.

## 2. Scope and Deliverables

### In scope

- Host patch engine (E1): verify, patch, restore, repair, doctor.
- Turn router runtime (E2): per-turn inference bridging, per-Bot state,
  control commands, replay suppression.
- Injection payload design, version marker, and staged rollout (E3).
- Outline-level scoping for delivery/installation (E4) and the LLM-assisted
  maintainer anchor-discovery tool (E5), expanded to executable fidelity once
  E1-E3 land and a real target-application build has been reviewed.

### Out of scope

See Non-goals above.

### Deliverables

| ID | Description | Owner | Acceptance criteria |
| --- | --- | --- | --- |
| D1 | `grokbotpatcher` binary: patch engine subcommands | TBD | Passes E1's authored acceptance suite |
| D2 | `grokbotpatcher` binary: turn subcommand | TBD | Passes E2's authored acceptance suite and golden-fixture regression suite |
| D3 | Injection payload + real manifest + staged rollout | TBD | T3.3 validated against a real reviewed build; two-release staged default flip observed |
| D4 (outline) | Delivery/installation mechanism | TBD | Expanded to executable fidelity via T4.0 once D1-D3 are stable |
| D5 (outline) | LLM-assisted anchor-discovery tool | TBD | Expanded to executable fidelity via T5.0 once D1 is stable and a real version-update case study exists |

## 3. Checkable Work Breakdown

Split layout, since the executable-fidelity epics exceed 25 tasks combined.

### E1 -- Host patch engine -> docs/plans/E1-host-patch-engine.md (0/11)

### E2 -- Turn router runtime -> docs/plans/E2-turn-router-runtime.md (0/13)

### E3 -- Injection payload and rollout -> docs/plans/E3-injection-payload-and-rollout.md (0/8)

### E4 -- Delivery and installation (outline)

fidelity: outline

How the built binary and manifest actually reach a user's machine and get
staged for `grokbotpatcher patch` to run -- transport, any needed diagnostic
access, and version/signature checks on the target application before
touching anything. Left as outline because the right approach depends on
what interface the target application actually exposes for this kind of
guided setup, which is confirmed hands-on during E3, not assumed here.

- [ ] T4.0 PLAN: expand E4 to executable fidelity (informed by E3's real-build review and staged-rollout experience)  Owner: pool  Est: 1h  kind: plan  delivers: [docs/plans/E4-delivery-and-installation.md at fidelity: executable]  deps: [T3.7]  acc: [parse_plan.py sees E4 with >= 5 tasks, every task carries acceptance criteria, deps resolve, fidelity flipped to executable]

### E5 -- LLM-assisted maintainer anchor-discovery tool (outline)

fidelity: outline

Given an old and a new build of the target application, propose where the
required injection anchors now live, or flag that the injection point
changed structurally, and draft a candidate manifest entry -- always as a
maintainer-reviewable diff, never applied without passing the existing
verification gate. Left as outline until a real historical version bump
exists to use as a retrospective test case for proposal quality.

- [ ] T5.0 PLAN: expand E5 to executable fidelity (informed by E1's manifest format and at least one real version-bump case study)  Owner: pool  Est: 1.5h  kind: plan  delivers: [docs/plans/E5-anchor-discovery-tool.md at fidelity: executable]  deps: [T1.9]  acc: [parse_plan.py sees E5 with >= 5 tasks, every task carries acceptance criteria, deps resolve, fidelity flipped to executable]

## 4. Parallel Work

### Tracks

| Track | Epic(s) | Depends on |
| --- | --- | --- |
| A: Patch engine | E1 | none |
| B: Turn router | E2 | none (T2.6/T2.9e depend on T2.1 within the track) |
| C: Injection and rollout | E3 | Track A (T1.7, T1.8) and Track B (T2.2) substantially complete |
| D: Delivery (outline) | E4 | Track C complete (T3.7) |
| E: Maintainer tooling (outline) | E5 | Track A tests complete (T1.9) |

Tracks A and B run fully in parallel.

### Waves

```
### Wave 1: Foundation (4 agents)
- [ ] T1.1 Go module scaffold (patch engine)          verifies: [infrastructure]
- [ ] T1.2 Manifest format + loader                    verifies: [UC-001]
- [ ] T2.1 Codex integration spike                     verifies: [infrastructure]
- [ ] T2.2 Turn subcommand scaffold + protocol structs verifies: [infrastructure]

### Wave 2: Core verification and state (6 agents)
- [ ] T1.3 Hash + byte-count verification              verifies: [UC-001, UC-002]
- [ ] T1.4 Signed-registry verification                verifies: [UC-006]
- [ ] T1.5 Anchor-based injection                       verifies: [UC-001, UC-002]
- [ ] T2.3 Per-Bot state model                          verifies: [UC-008]
- [ ] T2.4 Control-command parsing                      verifies: [UC-009]
- [ ] T2.5 OpenRouter-style bridge                      verifies: [UC-010]

### Wave 3: Activation and bridges (5 agents)
- [ ] T1.6 Syntax validation + atomic activate          verifies: [UC-001]
- [ ] T1.7 Prior-payload reconstruction framework        verifies: [UC-003]
- [ ] T1.8 doctor/restore/repair/enable/disable          verifies: [UC-004, UC-005]
- [ ] T2.6 Codex-style bridge                            verifies: [UC-007, UC-010]
- [ ] T2.7 Replay/suppression state machine              verifies: [UC-011]

### Wave 4: Tests and audit (9 agents)
- [ ] T2.8 Redacted audit logging                        verifies: [infrastructure]
- [ ] T1.9 Patch-engine acceptance tests (verify/patch)  verifies: [UC-001, UC-002]
- [ ] T1.10 Patch-engine acceptance tests (upgrade/restore/repair)  verifies: [UC-003, UC-004, UC-005]
- [ ] T2.9a Control-command acceptance tests             verifies: [UC-009]
- [ ] T2.9b Identity/state-isolation acceptance tests    verifies: [UC-008]
- [ ] T2.9c Replay/suppression acceptance tests          verifies: [UC-011]
- [ ] T2.9d OpenRouter bridge acceptance tests            verifies: [UC-010]
- [ ] T2.9e Codex bridge acceptance tests                 verifies: [UC-007]
- [ ] T1.11 gofmt/govet (patch engine)                    verifies: [infrastructure]

### Wave 5: Regression suite and lint (2 agents)
- [ ] T2.10 Golden-fixture regression suite               verifies: [infrastructure]
- [ ] T2.11 gofmt/govet (turn router)                      verifies: [infrastructure]

### Wave 6: Injection payload (4 agents)
- [ ] T3.1 Injection payload design/implementation         verifies: [infrastructure]
- [ ] T3.2 First version marker + reconstruction wiring     verifies: [UC-003]
- [ ] T3.3 Real manifest entry against a reviewed build      verifies: [UC-001]
- [ ] T3.4 Config schema                                     verifies: [infrastructure]

### Wave 7: Staging (1 agent)
- [ ] T3.5 Delivery/staging step                             verifies: [UC-001]

### Wave 8: Staged releases (sequential -- not parallel)
- [ ] T3.6 Release 1: routing default disabled
- [ ] T3.7 Release 2: routing default enabled

### Wave 9: Docs (1 agent)
- [ ] T3.8 Real architecture/usage documentation
```

Wave 8 is explicitly sequential -- do not compress it into one release even
if both are code-complete earlier.

## 5. Timeline and Milestones

| Milestone | Exit criteria | Depends on |
| --- | --- | --- |
| M1: Patch engine parity against fixtures | T1.11 done; full authored acceptance suite passes on synthetic fixtures | Wave 4 (Track A) |
| M2: Turn router parity against fixtures | T2.11 done; golden-fixture regression suite green | Wave 5 (Track B) |
| M3: Real build validated, routing default disabled | T3.6 done; patch/restore cycle observed against a real installation | Wave 8 (first release) |
| M4: Routing capability is the shipped default | T3.7 done; live end-to-end run observed and recorded | Wave 8 (second release) |
| M5: Delivery and maintainer tooling scoped | T4.0 and T5.0 complete, both epics flipped to executable fidelity | M4 |

## 6. Risk Register

| ID | Risk | Impact | Likelihood | Mitigation |
| --- | --- | --- | --- | --- |
| R1 | Fixture-based tests (M1/M2) pass while the real target application's actual structure differs in ways that break the real patch | High | Medium | T3.3 is a hard gate: no real-build validation, no further rollout, regardless of how clean fixture tests look |
| R2 | Codex-style integration cannot be cleanly implemented as a native Go HTTP/API client | Medium | Medium | T2.1 spike gates T2.6/T2.9e before further estimate is committed; CLI-subprocess fallback is pre-approved in this plan |
| R3 | A patch-splice bug corrupts a user's real, live installation during atomic activation | High | Low | T1.6's atomic temp-file-then-rename plus mandatory verified backup; T1.10's partial-write fixture; T3.6's staged rollout with routing off by default |
| R4 | A future payload version change breaks the upgrade path for users already on a prior version | High | Low | T1.7/T3.2's exact-reconstruction requirement (ADR 003); never trust a marker string alone |
| R5 | LLM-assisted anchor proposals (E5) get trusted without review, defeating the fail-closed model | High | Low (process risk, not yet built) | E5's scope explicitly requires every proposal to pass the existing verification gate; T5.0 must specify this as a hard constraint before E5 is expanded |
| R6 | Two-release rollout (T3.6, T3.7) gets compressed into one release under schedule pressure | Medium | Medium | Documented as sequential and non-negotiable in E3 and in this plan's Wave 8 |
| R7 | No real target-application build is available when T3.3 is scheduled | Medium | Medium | Flagged explicitly as an external dependency in E3's risk section, not folded into an engineering time estimate |

## 7. Operating Procedure

Definition of done for every task in this plan:

- Authored tests written and passing for every behavior (T1.9, T1.10,
  T2.9a-e are the parity proof, not optional cleanup).
- `gofmt` and `go vet` clean (T1.11, T2.11).
- PR merged to `main` via rebase, CI green.
- For E3's real-installation work specifically: a directly observed pass
  against a real reviewed build (T3.3, T3.6, T3.7) -- fixture-only or
  staging-only is not done for that epic's tasks.
- Reported honestly: state what was actually observed against a real
  installation, not what fixtures predicted.
- If the repo adopts versioned releases, create one after merge.

## 8. Progress Log

- 2026-09-22: Change Summary -- initial plan for the grokbotpatcher
  greenfield project. Scoped as an independent Go implementation from first
  principles: verify/patch a Grok Bot host, route per-Bot turns to a
  Codex-style or OpenRouter-style provider, and add LLM-assisted maintainer
  tooling for future version support. Created ADR 001-003 and E1-E3 at
  executable fidelity (32 tasks total), E4-E5 as outline epics with single
  planning tasks (T4.0, T5.0). No progress yet.

## 9. Hand off Notes

- Start with Track A (E1) and Track B (E2) Wave 1 in parallel -- neither has
  external dependencies. T2.1 (Codex spike) is on the critical path for
  T2.6/T2.9e; prioritize it early.
- The single most load-bearing fact for anyone picking this up cold: nothing
  before E3 can be validated against a real installation, because no real
  manifest of injection anchors can exist until a real reviewed build has
  been inspected (T3.3). Treat E1/E2's fixture-based test passes as proof of
  internal correctness, not proof the real patch works.
- Licensed Apache License 2.0 and public: every commit, issue, and PR is
  visible to anyone from this point forward. This raises the bar on the
  "never commit target-application source" rule below from a hygiene
  preference to a hard requirement with no private fallback to catch a
  mistake.
- Use case manifest: `.claude/scratch/usecases-manifest.json`.
- Never commit any source code or content belonging to the target
  application itself into this repository -- manifests record hashes and
  structural facts about a reviewed build, not copies of its source.

## 10. Appendix

- ADRs: docs/adr/001-single-go-binary-on-device-agent.md,
  docs/adr/002-shell-out-for-host-syntax-validation.md,
  docs/adr/003-injection-marker-versioning.md.
- docs/cli.md -- designed command-line interface (subcommands, flags, exit
  codes) that E1/E2/E3's tasks implement against.
- README.md -- project overview and design goals.
- LICENSE -- Apache License 2.0.
