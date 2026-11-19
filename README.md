# ticket-create

Drafts tickets with an interactive Claude session and creates them after one explicit confirmation.

## Status

0.1.0-design: the interface is specified in [docs/interface.md](docs/interface.md) and the reasoning is in [docs/design.md](docs/design.md); the implementation plan is [docs/plan.md](docs/plan.md). Nothing is implemented yet.

## Layer

L2 (composite). Works fully standalone; it does not need the `awkit` harness.

## What it does

- `ticket-create draft` runs a Claude session of kind `draft-ticket` in your terminal, interviews you about the ticket and stores the result as a draft with an id.
- `ticket-create create <draft_id>` shows the draft on a review screen, lets you edit it, and after one explicit `yes` creates the ticket through ticketctl, optionally adding it to a sprint.
- Creating a ticket needs a human at a terminal and always asks. There is no option to skip the question.
- The draft skill and the template are configurable: your organization's ticket format lives in your own config or skill, not in this repository.

Bundles a Claude Code plugin named `ticket-create` in `plugin/` with the default skill `ticket-create:draft-ticket`.

## Commands

See [Commands](docs/interface.md#commands) for every flag.

- `ticket-create draft`: make a draft (interactive by default; `--mode headless` for unattended drafts).
- `ticket-create create DRAFT_ID`: review, edit and create.
- `ticket-create list`, `show`, `discard`, `prune`: manage drafts.
- `ticket-create validate [--probe]`, `export`: checks and the public audit and log files.

## Dependencies

| Tool | Binding | Install |
|---|---|---|
| [ticketctl](https://github.com/agent-workflow-kit/ticketctl) | A (hard) | `uv tool install git+https://github.com/agent-workflow-kit/ticketctl` |
| [agent-spawn](https://github.com/agent-workflow-kit/agent-spawn) (0.34.0 or later) | A (hard) | `uv tool install git+https://github.com/agent-workflow-kit/agent-spawn` |

agent-spawn needs [claude-session](https://github.com/agent-workflow-kit/claude-session) and [wtctl](https://github.com/agent-workflow-kit/wtctl) installed, and the ticket system's token variable must be listed in claude-session's `secret_env` so the draft session never receives it. A missing sibling is reported with its install command.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/ticket-create
```

## Configuration

`~/.config/ticket-create/config.yaml` with drop-ins in `config.d/`. See [Configuration](docs/interface.md#configuration) and `config.example.yaml`. Environment overrides use the prefix `TICKET_CREATE_`.

## Hooks

`pre-draft`, `post-draft` and `post-create`; see [Hooks](docs/interface.md#hooks). A gate on the write itself belongs in ticketctl's own `pre-create` hook.

## Exit codes

See [Exit codes](docs/interface.md#exit-codes). In short: 0 done, 13 a human is needed (no terminal for `create`), 14 to 18 the draft session's outcome, 20 refused, 22 a draft's content is missing or invalid.

## Documentation

- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- [interface](docs/interface.md), [design](docs/design.md), [plan](docs/plan.md)

## License

MIT — see LICENSE.
