# E1 -- Host patch engine

Acceptance: `grokbotpatcher patch|doctor|restore|repair|enable|disable`
subcommands exist, verify a target host file against a manifest of reviewed
stock builds, apply the injection payload only when every check passes,
validate and atomically activate the result, and pass an authored acceptance
test suite covering every scenario in "Risks and required test scenarios"
below.

Context: This is the security-critical half of the agent. It must never
modify a host file it cannot verify exactly, and every operation that writes
to the live host must be atomic with a verified rollback path. See ADR 001
(single binary), ADR 002 (syntax validation), and ADR 003 (version marker
and reconciliation).

fidelity: executable

## Wave A (no dependencies)

- [ ] T1.1 Scaffold `cmd/grokbotpatcher` Go module with a `patch` subcommand stub (go.mod, main.go, stdlib only)  Owner: TBD  Est: 1h  verifies: [infrastructure]  acc: [go build ./cmd/grokbotpatcher produces a binary that runs grokbotpatcher patch --help]
- [ ] T1.2 Define the manifest format (Go struct: target-app version, stock-host sha256+byte-count entries, required injection anchors, payload version marker) and a JSON loader  Owner: TBD  Est: 1.5h  verifies: [UC-001]  acc: [the binary loads a fixture manifest.json and reports the parsed marker and anchor count via --dump-manifest]

## Wave B (after Wave A)

- [ ] T1.3 Implement hash + byte-count stock-host verification (crypto/sha256, os.Stat)  Owner: TBD  Est: 1.5h  deps: [T1.2]  verifies: [UC-001, UC-002]  acc: [a fixture host matching the manifest's stock entry verifies as trusted; a 1-byte-modified copy fails closed]
- [ ] T1.4 Implement signed-registry verification (crypto/ed25519) for a secondary source of trusted host fingerprints  Owner: TBD  Est: 2h  deps: [T1.2]  verifies: [UC-006]  acc: [a registry entry signed with a test key is trusted; a tampered entry or wrong-key signature is rejected]
- [ ] T1.5 Implement anchor-based injection: verify every required anchor is present exactly once in the target file, refuse on a missing/duplicated anchor or an unrecognized foreign marker, splice in the injection payload at the anchor point  Owner: TBD  Est: 3h  deps: [T1.2]  verifies: [UC-001, UC-002]  acc: [patching a fixture file with all required anchors present exactly once succeeds; a fixture missing one anchor, or containing one twice, or already carrying an unrecognized marker, fails closed with no file modification]

## Wave C (after Wave B)

- [ ] T1.6 Implement post-patch syntax validation (shell out per ADR 002) plus atomic backup-then-activate (temp file in the same directory, verified backup under a persistent backup path, then rename)  Owner: TBD  Est: 2h  deps: [T1.5]  verifies: [UC-001]  acc: [a generated file with deliberately broken syntax fails validation and the live host is left untouched; a valid generated file is activated atomically with the pre-change file backed up first]
- [ ] T1.7 Implement the prior-payload reconstruction framework: given a live host already carrying a known marker, reconstruct that exact prior payload from the verified stock backup, refuse if reconstruction does not match byte-for-byte  Owner: TBD  Est: 2.5h  deps: [T1.5]  verifies: [UC-003]  acc: [upgrading a fixture host that already carries a known prior marker succeeds only when reconstruction matches exactly; a live host that diverges from the exact reconstruction is refused]
- [ ] T1.8 Implement `doctor`, `restore`, `repair`, `enable`, `disable` subcommands  Owner: TBD  Est: 2.5h  deps: [T1.6]  verifies: [UC-004, UC-005]  acc: [grokbotpatcher restore copies the verified backup over the live host; grokbotpatcher doctor reports live-vs-expected health without modifying anything; grokbotpatcher repair only acts when the live file exactly matches an allowlisted stock entry]

## Wave D (tests and lint)

- [ ] T1.9 Author acceptance tests for verify/patch/refuse scenarios against fixture host files (valid stock match, hash mismatch, byte-count mismatch, missing anchor, duplicated anchor, foreign marker)  Owner: TBD  Est: 3h  deps: [T1.8]  verifies: [UC-001, UC-002]  acc: [go test ./cmd/grokbotpatcher/... has a named test per scenario listed above and all pass]
- [ ] T1.10 Author acceptance tests for upgrade/restore/repair/doctor scenarios, including a killed-mid-write / partial-file fixture  Owner: TBD  Est: 2.5h  deps: [T1.8]  verifies: [UC-003, UC-004, UC-005]  acc: [go test covers exact-reconstruction-required upgrade, restore-with-no-backup-stops, repair-only-on-allowlisted-stock, and a simulated interrupted-write does not leave a corrupt live host]
- [ ] T1.11 Run gofmt and go vet on cmd/grokbotpatcher and fix findings  Owner: TBD  Est: 0.5h  deps: [T1.9, T1.10]  verifies: [infrastructure]  acc: [gofmt -l cmd/grokbotpatcher reports no files; go vet ./cmd/grokbotpatcher/... is clean]

## Risks and required test scenarios

- The manifest's required-anchor strings cannot be authored until a real
  target-application build is available for review (T1.3-T1.5's fixtures use
  synthetic anchor text; T3.3 in E3 is where a real manifest entry gets
  built against an actual reviewed build). Do not treat fixture-passing tests
  here as proof the real target application is patchable -- that proof comes
  from E3.
- The atomic-activate step (T1.6) is the point where a bug could leave a
  user's real installation of the target application in a broken state;
  T1.10's partial-write fixture is not optional.
