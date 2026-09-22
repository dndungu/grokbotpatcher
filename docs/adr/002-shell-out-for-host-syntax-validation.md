# ADR 002: Shell out to the platform's own JS engine for post-patch syntax validation, do not embed a parser

## Status
Accepted

## Date
2026-09-22

## Context
Grok Bot's desktop host process is assumed to be an Electron/Node-based
application (a working assumption about the target application, to be
confirmed against an actual installed build during E1 -- see
[docs/plans/E1-host-patch-engine.md](../plans/E1-host-patch-engine.md)). If
so, the file this project patches is JavaScript, and after splicing in the
injection payload that file must be proven syntactically valid before it is
ever activated -- a broken host file left in place would break Grok Bot
itself, not just routing. Go has no built-in JavaScript parser. The choice is
between (a) shelling out to the Node runtime already required to run the
target application itself, or (b) embedding a third-party Go JavaScript
parsing library.

## Decision
Shell out to `node --check <file>` (or the closest equivalent the actual
target runtime exposes, confirmed during E1) for post-patch syntax
validation. Do not embed a third-party Go JavaScript parser.

## Consequences
- Avoids depending on a parser whose accepted-syntax surface could silently
  diverge from whatever engine actually executes the target application's
  host file -- a false-negative syntax check on a gate that decides whether
  to activate a patch to someone's live application is a worse failure mode
  than one `exec.Command` call.
- This is the one place the on-device agent still depends on something it
  did not ship itself. It is not a new dependency this project introduces --
  if the assumption above holds, the target application already requires
  that same runtime to exist on that machine in order to run at all.
- If E1's confirmation step finds the target host process is not
  Node/Electron-based, or exposes no equivalent syntax-check facility, this
  ADR must be revisited before E1 proceeds past its verification-and-patch
  tasks.
