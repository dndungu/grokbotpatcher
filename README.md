# grokbotpatcher

A small, dependency-minimal on-device agent that lets an individual Grok Bot
choose which model actually does the thinking for a conversation turn, while
Grok Bot itself stays the interface: chat UI, conversation history, computer,
files, browser, and tool permissions are untouched.

Grok Bot continues to own every conversation and every tool call it grants.
`grokbotpatcher` only decides, per Bot, which backend answers a turn and
translates that turn's context and tool schemas to and from that backend.

## Status

Planning. See [docs/plan.md](docs/plan.md) for the full work breakdown,
[docs/adr/](docs/adr/) for the architectural decisions already made, and
[docs/cli.md](docs/cli.md) for the designed command-line interface.

## Design goals

1. **One static binary on the target machine.** The agent that runs inside
   the Bot's own remote computer should require nothing beyond what the
   target application itself already needs present (see
   [docs/adr/002-shell-out-for-host-syntax-validation.md](docs/adr/002-shell-out-for-host-syntax-validation.md)).
   No separate interpreter runtime, no package manager, no installed script
   dependencies of its own.
2. **Fail closed.** Never modify a host build that has not been verified
   exactly. Never trust a version marker alone when reconciling an upgrade.
   Never invent a tool call the host did not offer.
3. **Reversible by construction.** Every patch keeps a verified backup of
   what it replaced and can be restored atomically.
4. **Per-Bot isolation.** Each Bot's provider and model choice is its own
   state, keyed by a stable identity, never by a conversation or request ID
   that can change underneath it.
5. **Version resilience via tooling, not tribal knowledge.** When the target
   application updates, an LLM-assisted tool proposes the updated injection
   points as a reviewable diff a maintainer signs off on -- never something
   that patches a live install unreviewed.

## License

Apache License 2.0. See [LICENSE](LICENSE).
