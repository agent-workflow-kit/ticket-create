# ticket-create: implementation plan

This plan implements [interface.md](interface.md) and [design.md](design.md). Each issue is expanded into small test-first steps when it is picked up. Status values: Not Started, In Progress, Complete.

## Conventions for every stage

- **Gateway boundary.** Every subprocess call (ticketctl, agent-spawn, the editor, hooks) goes through `src/ticket_create/gateway.py`. Tests substitute fakes: scripts in `tests/fakebin/` (a fake `ticketctl` and a fake `agent-spawn`, copied per repo) driven by scenario files.
- **Pinned sibling schemas.** Copies of ticketctl's and agent-spawn's schemas for the outputs ticket-create parses live in `tests/schemas/`, pinned to a version. Fakes print output that validates against them. Sibling JSON is read tolerantly (unknown keys ignored).
- **Process and live tests.** Unit and process tests (including pseudo-terminal tests for the review screen, prompt and editor) run in CI. Tests that need a real `claude`, a real ticketctl or a real terminal are marked `live`, are deselected by default and never run in CI.
- **Golden files.** Every `--json` output and every hook payload has a golden file and is validated against `schemas/`.
- **Spike regressions.** Each verified fact in the design's live-verification table (V-1 to V-11) has a test: unit or process where the behavior is ours, `live` where it is a sibling's.
- **Security regressions.** The reader and sanitizer attack corpus, the prompt's stale-input cases and the dedupe scenario table run as unit tests.
- **`make verify`** (ruff, pyright, pytest, markdown lint) passes before every commit. Run `uvx check-jsonschema --check-metaschema` on every schema file at the version CI pins before pushing.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `gateway` | 0 | all subprocess calls |
| `cli` | 0 | argparse, global flags, exit codes, output, error envelope |
| `config` | 0 | load, merge, validate, precedence |
| `state` | 1 | SQLite, migrations, per-draft lease |
| `drafts` | 1 | state machine, attempts |
| `audit` | 1 | `audit.jsonl`, `log.jsonl` |
| `sanitize` | 2 | the one sanitizer |
| `artifact` | 2 | strict reader, slot validation |
| `ticketctl_client` | 3 | reads, plans, result mapping |
| `prefetch` | 3 | create-form facts, duplicates, parent |
| `prompt` | 3 | prompt file, fencing |
| `session` | 4 | agent-spawn calls, kind check, outcomes, orphan recovery |
| `hooks` | 4 | hook execution and payloads |
| `review` | 5 | screen, prompt, editor, terminal snapshot and restore |

## Stage 0: scaffold, config and gateway

**Goal**: A runnable CLI skeleton with config loading, the gateway and the fakes.
**Success Criteria**: `ticket-create --version`, `validate` (offline) and `--help` work; config loads with precedence, per-repo overrides and drop-ins; every config error names its key; the sibling check reports `tool_missing` and `tool_too_old` with install commands.
**Tests**: config precedence and override-exclusion tests; drop-in merge; schema validation of `config.example.yaml`; the fakes validate against the pinned schemas; golden output of `validate --json`.
**Status**: Not Started

## Stage 1: state, drafts and the public files

**Goal**: The draft store with its state machine, attempts, lease and retention, plus the audit and log writers.
**Success Criteria**: `list`, `show`, `discard`, `prune` and `export` work on stored drafts; statuses move only along the transitions in the interface; two processes cannot hold one draft's lease; additive migrations apply; audit and log hold no text.
**Tests**: state-machine transition table; lease contention and takeover; retention boundaries; migration from an older fixture; audit and log schema validation; a grep-style test that no title or body appears in either file.
**Status**: Not Started

## Stage 2: reader, validator and sanitizer

**Goal**: The one reader for untrusted files, the artifact validator and the one sanitizer.
**Success Criteria**: every attack file and display case in the corpus is rejected or neutralized; the validator reports a `code` and `detail` for every failure and never raises.
**Tests**: the 29 file and structure cases (symlink, FIFO, `/dev/zero`, oversize, duplicate keys, deep nesting, bad UTF-8); the 28 display cases rendered in a pseudo-terminal before and after; slot-contract positive and negative samples including every closed pattern; a regression that every pattern compiles in the schema engine CI uses.
**Status**: Not Started

## Stage 3: prefetch, prompt and skill preflight

**Goal**: Everything `draft` does before the session starts.
**Success Criteria**: the prompt file has the trusted header and fenced blocks with a fresh nonce and neutralized closing tags; the prefetch handles every ticketctl exit; the skill preflight finds a `plugin:skill` in the passed plugin dirs or exits 12 `skill_not_found`; a template file is passed as a trusted input file.
**Tests**: prompt golden files with hostile seeds (closing-tag look-alikes, long blocks, control characters); prefetch against the fake ticketctl for each exit code and the invalid-type probe; preflight with present, missing and bare skill names; the project-to-repo mapping fallback order.
**Status**: Not Started

## Stage 4: the draft session

**Goal**: `draft` end to end with the fake agent-spawn, then with the real one.
**Success Criteria**: the launch flags equal the interface table; the kind check refuses a misspelled kind; every row of the Session outcomes table maps to the stated status and exit code; a missing result file falls back to `agent-spawn show`; orphans are found by label; `--continue` resumes; headless drafts need a seed and check `report.implicit`; `pre-draft` and `post-draft` run with payloads that carry no text.
**Tests**: scenario tests per outcome row; a crash between launch and store; result-file replacement, symlink and unwritable cases; hook veto and escalation; the bundled plugin's `SKILL.md` is a single file and the profile denies ticketctl and the other listed tools. Live: V-1 to V-8 against a real `claude` on a cheap model.
**Status**: Not Started

## Stage 5: review, create and sprint

**Goal**: `create` end to end, the only writer.
**Success Criteria**: without `/dev/tty` it exits 13 before anything else; the screen order and caps match the interface; the prompt declines on stale input, paste, EOF and Ctrl-C; the terminal is restored after the session, the editor and a killed child; a changed plan returns to the dry-run; the attempt history drives the "already created" state; `--again`, overrides and `--sprint` behave as specified; a failed sprint step leaves `sprint_failed` with the key; `post-create` and the audit pair are written.
**Tests**: pseudo-terminal tests for every screen state and answer; the editor abort cases; the dedupe scenario table (no edit, body edit, title edit, type edit, a re-run after success, a retry after `outcome_unknown`); every ticketctl exit in the Lower tool exits table; audit pairing by `attempt_id`.
**Status**: Not Started

## Stage 6: live suite, hardening and release

**Goal**: Everything verified live that can be, and the release checklist.
**Success Criteria**: V-12 to V-15 are verified against the released ticketctl and memoctl and the design's table is updated; the macOS checklist (V-16) is done; the README's setup steps are tested on a clean machine.
**Tests**: the `live` suite (a real draft and a real create against a throwaway project the maintainer designates, a wrong skill, a missing artifact, secret stripping, the trust dialog in a session directory); the macOS checklist.
**Status**: Not Started

## Version roadmap

- **0.x**: stages 0 to 6.
- **1.0** requires: the live suite green against released ticketctl, agent-spawn and claude-session; the interface unchanged for two minor releases; V-12 to V-16 settled.
- **1.x** (additive): window placement for interactive drafts; a gated automatic create (a config opt-in plus an enforced gate, never a bare `--yes`); sprint and epic lists in the prefetch; link creation once ticketctl offers it; a `pre-create` hook if a use appears; a `reconcile` command if label lookup in `list` proves too lazy; a TUI picker for drafts.
- **2.0**: nothing planned.
