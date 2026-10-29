# ticket-create

Skill-wrapper that drafts tickets with Claude and creates them after confirmation.

## Status

Pre-design: interface not yet specified; see [docs/design.md](docs/design.md).

## Layer

L2 (composite). Works fully standalone; it does not need the `awkit` harness.

## What it does

- Default mode interactive; `draft` runs a session (kind `draft-ticket`) in the current terminal (`--placement here`).
- `create` requires a TTY (exit 13 otherwise) and always confirms before creating.
- The draft-ticket skill and template are configurable (an organization's ticket format lives in the user's config or skill, not here).

Bundles a Claude Code plugin named `ticket-create` in `plugin/` with the default `draft-ticket` skill (`ticket-create:draft-ticket`).

## Indicative commands

Indicative, finalized in [docs/interface.md](docs/interface.md).

- `ticket-create draft`
- `ticket-create create`

## Dependencies

- ticketctl: binding: TBD (A/B/C), decided in this tool's design session
- agent-spawn (same layer; allowed by architecture §3.2): binding: TBD (A/B/C), decided in this tool's design session

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/ticket-create
```

## Documentation

- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))

## License

MIT — see LICENSE.
