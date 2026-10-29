# ticket-create: design

Status: Seeded from awkit architecture v1.4; not yet designed.

## Role and layer

Layer 2 (composite). Skill-wrapper that drafts tickets with Claude and creates them after confirmation. Works fully standalone (architecture C2).

## Responsibilities (architecture §5.2)

Skill-wrapper: draft and create tickets.

- Commands: `draft`, `create`.
- Uses: ticketctl, agent-spawn (same layer; allowed by section 3.2).
- Notes: Default mode interactive; `draft` runs the session (kind `draft-ticket`) in the current terminal (`--placement here`). `create` requires a TTY (exit 13 otherwise) and always confirms before creating. The draft-ticket skill and template are configurable (an organization's ticket format lives in the user's config or skill, not here).

Section numbers above refer to `../awkit-docs/architecture.md`.

## Uses

- ticketctl: binding TBD (A/B/C), decided in this design session
- agent-spawn (same layer; allowed by architecture §3.2): binding TBD (A/B/C), decided in this design session

## Cross-cutting sections to read

In `../awkit-docs/architecture.md` (plus `../awkit-docs/design-guide.md` throughout):

- §5.5: capability slots: `draft-ticket`, kind `draft-ticket`, default mode interactive; design guide §11 (skill-wrapper requirements)
- §8.1: kinds (`draft-ticket`: not third-party-reading, caller's cwd)
- §9.2: launch flags and `--placement here`
- §12.1, §12.4: create tickets: interactive, always confirm; commands that require a human exit 13 without a TTY
- §13.2: prompt injection: draft text is agent-generated and travels in files
- §14: audit of ticket creation

## Capability slot

Default skill: `ticket-create:draft-ticket` (bundled in `plugin/skills/draft-ticket/`). The slot contract (inputs given to the skill, artifact format expected back) is specified in `docs/interface.md`.

## Constraints (decided)

From architecture §20: `create` requires a TTY.

Packaging: The bundled plugin is shipped inside the package and located via `importlib.resources.files('ticket_create') / 'plugin'`.

## Open topics

From architecture §20:

- draft template
- confirm flow

## Decisions

| Decision | Rationale | Date |
|---|---|---|

## Questions for the design session

1. What is the default draft template, and how does the draft-ticket skill's artifact map onto `ticketctl create --body-file` and fields?
2. What does the confirm flow look like (preview, edit, field mapping review) before `create` calls ticketctl?
