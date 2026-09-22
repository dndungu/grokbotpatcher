# E3 -- Injection payload and rollout

Acceptance: A concrete injection payload exists that spawns
`grokbotpatcher turn` directly (no interpreter argument needed beyond what
the target application itself already requires, per ADR 002), carries its
own version marker (ADR 003), and has a real manifest entry built against an
actual reviewed target-application build; the agent ships with a safety
toggle so the routing capability can be enabled by default only after it has
been verified end-to-end against a real installation.

Context: E1 and E2 produce components that cannot do anything until the
injection payload itself is designed and a real manifest exists -- the
manifest's required anchors are strings that must be identified by reviewing
an actual build of the target application, not invented in the abstract.
This epic is also the only one that touches a user's real, live installation.

fidelity: executable

## Wave A (depends on E1 and E2 substantially complete)

- [ ] T3.1 Design and implement the injection payload: the minimal script spliced into the target host that checks whether routing is enabled, and if so spawns `grokbotpatcher turn` directly with the turn context piped on stdin  Owner: TBD  deps: [T1.8, T2.2]  Est: 2h  verifies: [infrastructure]  acc: [a fixture host patched with this payload spawns the configured binary directly with the expected stdin content, and does nothing when routing is disabled]
- [ ] T3.2 Assign the first version marker and wire it into the reconstruction framework from T1.7  Owner: TBD  deps: [T3.1, T1.7]  Est: 1h  verifies: [UC-003]  acc: [a fixture host carrying this marker reconstructs correctly against its stock backup]
- [ ] T3.3 Obtain and review an actual reviewed build of the target application; build the first real manifest entry (hash, byte count, real anchor strings identified during review) and validate the full patch flow against it in an isolated test environment  Owner: TBD  deps: [T3.2]  Est: 4h  verifies: [UC-001]  acc: [grokbotpatcher patch succeeds against the real reviewed build in an isolated test environment and the target application's normal behavior is otherwise unaffected]
- [ ] T3.4 Define the on-disk config schema (provider, model, reasoning, enabled flag, safety toggle for the routing capability) and its default values  Owner: TBD  deps: [T3.1]  Est: 1h  verifies: [infrastructure]  acc: [a fresh install with no config file present behaves as routing-disabled until explicitly configured]

## Wave B (after Wave A)

- [ ] T3.5 Build the delivery/staging step that gets the built binary and manifest onto the target machine (transport TBD by E4's outline expansion; this task only covers what happens once the payload is already present)  Owner: TBD  deps: [T3.4]  Est: 2h  verifies: [UC-001]  acc: [staging places the binary and manifest at their expected paths with correct permissions, verified by checksum before grokbotpatcher patch is invoked]

## Wave C (staged rollout -- sequential, each gated on real verification)

- [ ] T3.6 Release 1: ship with the routing capability defaulting to disabled until a maintainer has run the full patch/turn/restore cycle against a real installation and confirmed no regression to the target application's own behavior  Owner: TBD  deps: [T3.5]  Est: 2h  verifies: [infrastructure]  acc: [a full patch-then-restore cycle against a real installation leaves the application in its original working state, confirmed manually]
- [ ] T3.7 Release 2: flip the default once T2.10's regression suite is green and a live end-to-end run (real installation, real provider call, real tool-call round trip) has been observed and recorded  Owner: TBD  deps: [T3.6]  Est: 2h  verifies: [infrastructure]  acc: [a fresh install after this release routes a real turn through a real provider end to end, observed directly, not assumed]

## Wave D (docs)

- [ ] T3.8 Write the user-facing install/restore/troubleshooting documentation and the implementation-level architecture document, describing what was actually built (not a plan of what would be built)  Owner: TBD  deps: [T3.7]  Est: 2h  verifies: [infrastructure]  acc: [README.md's Status section is updated to reflect a working, released agent, and a docs/ARCHITECTURE.md exists describing the real implementation]

## Risks

- This is the only epic that touches a real user's live installation; T3.3's
  isolated-test-environment requirement and T3.6/T3.7's staged, gated rollout
  are both non-negotiable regardless of schedule pressure.
- T3.3 cannot be scheduled purely on engineering estimate -- it depends on
  having lawful, hands-on access to a real reviewed build of the target
  application to inspect. Flag this as an external dependency, not a task
  duration risk.
