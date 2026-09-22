# ADR 003: The injection payload carries its own version marker, and upgrades require exact reconstruction

## Status
Accepted

## Date
2026-09-22

## Context
Once the host file has been patched, a later upgrade of `grokbotpatcher`
itself may need to change the injected payload -- for example, changing how
the per-turn process is invoked, or fixing a bug in the injected code. At
that point the live host file is no longer the original stock build; it
already contains a prior version of this project's own payload. Trusting a
marker string alone as proof of what is actually present would let a
tampered or partially-modified file slip through as "already ours, safe to
overwrite."

## Decision
Every injected payload embeds an exact, unique version marker (e.g.
`GROKBOTPATCHER_ADAPTER_V1`, incrementing per payload change). Before
`grokbotpatcher patch` overwrites a host file that already carries a marker,
it must exactly reconstruct the expected prior payload from a verified stock
backup using that version's published transformation, and refuse to proceed
if reconstruction does not match the live file byte-for-byte. A marker string
being present is necessary but never sufficient.

## Consequences
- Every payload version needs a corresponding reconstruction path in the
  patch engine (E1), so upgrade support has real, growing cost per version --
  budget for it explicitly in future payload-changing work rather than
  treating a marker bump as free.
- This is what makes `repair` (reapplying the patch after the target
  application silently replaces the host file with an allowlisted stock
  build) and `restore` (returning to stock) safe to run automatically or
  near-automatically: both operations only ever act when the live file
  exactly matches something already known, never on a best-effort guess.
- Any change to how the per-turn process is invoked (its argv, its IPC
  contract) is therefore a new marker version and a new reconciliation path,
  not a silent runtime-only change -- see
  [docs/plans/E3-injection-payload-and-rollout.md](../plans/E3-injection-payload-and-rollout.md).
