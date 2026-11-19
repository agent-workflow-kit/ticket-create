# ticket-create: implementation plan

This plan implements [interface.md](interface.md) and [design.md](design.md). Each issue is expanded into small test-first steps when it is picked up. Status values: Not Started, In Progress, Complete.

## Conventions for every stage

- **Gateway boundary.** Every subprocess call (ticketctl, agent-spawn, the editor, hooks) goes through `src/ticket_create/gateway.py`. Tests substitute fakes: scripts in `tests/fakebin/` (a fake `ticketctl` and a fake `agent-spawn`, copied per repo) driven by scenario files.
- **Pinned sibling schemas.** Copies of ticketctl's and agent-spawn's schemas for the outputs ticket-create parses live in `tests/schemas/`, pinned to a version. Fakes print output that validates against them. Sibling JSON is read tolerantly (unknown keys ignored).
- **Process and live tests.** Unit and process tests (including pseudo-terminal tests for the review screen, prompt and editor) run in CI. Tests that need a real `claude`, a real ticketctl or a real terminal are marked `live`, are deselected by default and never run in CI.
- **Golden files.** Every `--json` output and every hook payload has a golden file and is validated against `schemas/`.
- **Spike regressions.** Each verified fact in the design's live-verification table (V-1 to V-11 and V-17 to V-19) has a test: unit or process where the behavior is ours, `live` where it is a sibling's.
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
| `resolve` | 3 | site, project, type and repo chain, skill preflight, sprint selectors |
| `ticketctl_client` | 3 | reads, plans, result mapping |
| `prefetch` | 3 | create-form facts, duplicates, parent |
| `prompt` | 3 | prompt file, fencing |
| `session` | 4 | agent-spawn calls, kind check, outcomes, orphan recovery |
| `hooks` | 4 | hook execution and payloads |
| `review` | 5 | screen, prompt, editor, terminal snapshot and restore |

## Stage 0: scaffold, config and gateway

**Goal**: A runnable CLI skeleton with config loading, the gateway and the fakes.
**Success Criteria**: `ticket-create --version`, `validate` (offline) and `--help` work; config loads with precedence, per-repo overrides and drop-ins; every config error names its key; the sibling check reports `tool_missing` and `tool_too_old` with install commands. `validate` offline covers the config and the siblings; the skill and template checks land in Stage 3. When ticketctl is released, fill in its minimum version.
**Tests**: config precedence and override-exclusion tests; the environment layer reads only `mode`, `model`, `timeout`, `duplicates.*` and `retention.*`; `credential_env` removes names from the hook and editor environment and `validate` warns while it is empty; `validate` warns `profile_missing_deny` for a replaced profile that lacks a deny for `ticketctl`, `agent-spawn`, `ticket-create` or `mcp__*`; drop-in merge; schema validation of `config.example.yaml`; the fakes validate against the pinned schemas; golden output of `validate --json`.
**Status**: Not Started

## Stage 1: state, drafts and the public files

**Goal**: The draft store with its state machine, attempts, lease and retention, plus the audit and log writers.
**Success Criteria**: `list`, `show`, `discard`, `prune` and `export` work on stored drafts; statuses move only along the transitions in the interface (a run never destroys a stored valid artifact); the gate table holds for every status; attempts are append-only and the attempt outcome starts as `unknown`; the lease renews in the background; two processes cannot hold one draft's lease; every write is conditional on the lease token; a live `drafting` draft is refused with exit 20 before any lease is taken; the stored seed and warnings round-trip; additive migrations apply; audit and log hold no text.
**Tests**: a test generated from the interface's transition table, one case per row (T1 to T19); a gate-table test, one case per cell, run before the lease (a live `drafting` draft gives 20, not 12; `create`, `discard` and `draft --continue` share one `enter` rule); T5 applies the fixed values of the seed, clears overrides and recomputes warnings, and a continuation rebuilds its header from `seed`; origin is sticky (an interactive continuation never relabels a headless draft); lease contention, renewal and takeover, including a takeover after a simulated suspend that leaves the old holder unable to write; retention per class (`open`: drafting, ready, failed, abandoned; `done`: created, sprint_failed, unknown) measured from `updated_at`; migration from an older fixture; audit and log schema validation; a grep-style test that no title or body appears in either file.
**Status**: Not Started

## Stage 2: reader, validator and sanitizer

**Goal**: The one reader for untrusted files, the artifact validator and the one sanitizer.
**Success Criteria**: every attack file and display case in the corpus is rejected or neutralized; the validator reports a `code` and `detail` for every failure and never raises.
**Tests**: field-name grammar (including the reserved names, `epic` among them, for the artifact and `--set`), number forms (exponents, booleans and non-finite values rejected), a leading dash in any model-derived value; trailing-newline values for every grammar (full match); a Unicode type name; a tab in a title or field name; a `--set` file over the artifact cap; an artifact whose reported name is not `draft.json`; the 29 file and structure cases (symlink, FIFO, `/dev/zero`, oversize, duplicate keys, deep nesting, bad UTF-8); the 28 display cases rendered in a pseudo-terminal before and after; slot-contract positive and negative samples including every closed pattern; a regression that every pattern compiles in the schema engine CI uses.
**Status**: Not Started

## Stage 3: prefetch, prompt and skill preflight

**Goal**: Everything `draft` does before the session starts.
**Success Criteria**: the prompt file has the trusted header and fenced blocks with a fresh nonce and neutralized closing tags; the prefetch handles every ticketctl exit; the skill preflight finds a `plugin:skill` in the passed plugin dirs or exits 12 `skill_not_found`; a template file is copied to a private `template.md` and passed as the trusted input file (read through the reader, deleted when the run ends); site and project resolve by the documented chain; the form facts go in a fenced untrusted block; the bundled skill exists and works with the slot contract.
**Tests**: prompt golden files with hostile seeds (closing-tag look-alikes, long blocks, control characters); prefetch against the fake ticketctl for each exit code and the invalid-type probe; preflight with present, missing and bare skill names; the project-to-repo mapping fallback order and the project and site chain table; a search text from an 800-character seed is cut at 80 characters on a word boundary; a dry run of `draft` writes the prompt under `prompts/`, deletes it and leaves no row or hook call; `--idea-file -` with endless stdin, terminal stdin and over-cap stdin; the repo id derivation from `origin` (scp form, default port, `.git`); the site chain via `sites resolve`. Task: write the bundled skill (`plugin/skills/draft-ticket/SKILL.md`): how it learns the template file name and the path on the start line, the artifact path and the continuation prompt.
**Status**: Not Started

## Stage 4: the draft session

**Goal**: `draft` end to end with the fake agent-spawn, then with the real one.
**Success Criteria**: the launch flags equal the interface table; the kind check refuses a misspelled kind; every row of the Session outcomes table maps to the stated status and exit code; a missing result file falls back to `agent-spawn show`; an agent is found by its stored id, else by label and kind (exactly one, not torn down); every error before a session exists deletes a new draft and leaves a continued one as it was; `--continue` adopts a finished headless run, resumes a parked agent, starts a new session from a stored artifact, refuses a live session with exit 20 `live`, and refuses with nothing to continue from (`draft_not_resumable`); a continuation never changes a stored artifact's status; the live check holds for `--continue`, `discard` and `prune` in every status (a `ready` draft whose continuation died while its agent runs is live), and an agent-spawn error, a timeout or two label hits is exit 12 `lower_failed` (`unknown` in `list`, a skip in `prune`), never an orphan; a headless launch passes `--wait` and reads the result, or falls back to `agent-spawn show`; headless drafts need a seed and check `report.implicit`; `pre-draft` and `post-draft` run with payloads that carry no text.
**Tests**: scenario tests per outcome row; a fake agent-spawn scenario for a headless launch with and without a `--wait` result and a result file with `dry_run` or `run: null`; a headless exit 14 is `failed` with `no_report`, an interactive one is `abandoned`; a crash between launch and store; `--continue` cases: adopt a finished headless run, a live run (exit 20 `live`), kill during a continuation of a `ready` draft leaves it creatable, a stored artifact survives a crash, headless `--continue`, a failed resume falls back to the stored artifact once; signals before and during the launch; the kill-by-signal and the pre-run error rows (exit 11, 12, 13, 20, 21 with no run); result-file replacement, symlink and unwritable cases; hook veto and escalation; a refused launch whose teardown fails keeps the row as `failed` with `launch_failed`; `origin` is written at T3 before a headless launch, and an adopted run's mode comes from its record; the bundled plugin's `SKILL.md` is a single file and the profile denies ticketctl and the other listed tools. Live: V-1 to V-8 and V-17 to V-19 against a real `claude` on a cheap model; a resume of an agent that never worked is refused by our own check; V-19 (the template input path).
**Status**: Not Started

## Stage 5: review, create and sprint

**Goal**: `create` end to end, the only writer.
**Success Criteria**: without `/dev/tty` it exits 13 before anything else; the screen order and the row allocator match the interface; the prompt declines on stale input, paste, EOF and Ctrl-C; the terminal is restored after the session, the editor and a killed child; the dry-run runs as the human actor and a changed plan returns to the screen; the dry-run alone decides the state (`ready`, `already_created`, `blocked`) and the attempt history only adds warnings; a project or site override after an attempt exits 20 `attempt_recorded`; the screen puts every model-written value, including the sprint name and the full title, above the line under one row allocator with a floor of two rows per item, and no fact row is folded; every width on the screen follows the one width rule (printable ASCII 1 cell, every other code point 2); while any item is cut, `y` is refused until the `v` viewer has shown every cut item; `accept` is a read that never calls a create; a hook escalation is a second human question; `--again`, overrides, `accept` and `--sprint` (including a sprint-only run with its own screen) behave as specified; a failed sprint step leaves `sprint_failed` with the key; every exit of `create` passes one finish step; `post-create` and the audit pair are written.
**Tests**: pseudo-terminal tests for every screen state and answer, including the forged-fields attack at 24 rows, the screen row allocator at 24 rows with 20 fields, tab and wide-character lines, the autowrap-off check, a padded title, a test that the last row before the prompt is always a fact row, the fold with the gutter at 80 columns, the too-small terminal and Ctrl-C (exit 130); the width test (every code point the sanitizer leaves alone, drawn in a pseudo-terminal: the real cursor advance is at most the counted width, printable ASCII 1 cell and every other code point 2, and a row of a base letter plus 8 marks of any 2-cell code point fits the screen with no hidden tail); the closed prompt lines at 40 and 80 columns with autowrap on, the cut marker clipped and never folded; the `v` viewer cases (a cut item blocks `y`, paging with Enter and `q`, `q` before the last page leaves the item unseen, `y` accepted after every cut item was shown, refused again after an edit, an override or a resize, a `y` typed against a stale size sends nothing, page size `max(R - 2, 3)`, refusal below 5 rows); one finish-step case per exit of `create` (created, reused, partial, failed, unknown, declined, blocked, Ctrl-C, internal error) asserting cleanup, audit pairing (every opened `attempted` record, sprint add included, is closed; `unknown` when the answer is unknown), the `post-create` result and lease release; the signal tests (SIGINT, SIGHUP and SIGTERM at the prompt, in the editor, during the `post-create` hook and in the write window: `finish` always completes; a created ticket exits with the write's code, never 130, 129 or 143; the exit is 130, 129 or 143 only when nothing was written); a failed audit write at step 6 exits 1 with the draft unchanged and no ticketctl call; the sprint step's failure reaches `post-create` as `partial`; `accept` with a vanished match writes nothing and calls no create; kill between the create and the sprint step leaves `sprint_failed`; a sprint-only run with an extra flag exits 2; `--again` generation reset; the review offer after `draft` (not with `--json` or `--result-file`, never headless); the plan body not a string exits 12 `lower_failed`; a fixed field the artifact changed is kept and shown as a suggestion adopted with `create --type`; the editor abort cases; the dedupe scenario table (no edit, body edit, title edit, type edit, project override, site override, a re-run after success, a retry after `outcome_unknown`); every row of the Lower tool exits table; the dry-run runs as the human actor; a file swapped between plan and write exits 1 `input_changed`; attribution disabled does not loop; escalation yes and no with decision ids forwarded; audit pairing by `attempt_id`.
**Status**: Not Started

## Stage 6: live suite, hardening and release

**Goal**: Everything verified live that can be, and the release checklist.
**Success Criteria**: V-12 to V-15 and V-17 to V-19 are verified against the released ticketctl and memoctl and the design's table is updated; the macOS checklist (V-16) is done; the README's setup steps are tested on a clean machine.
**Tests**: the `live` suite (a real draft and a real create against a throwaway project the maintainer designates, a wrong skill, a missing artifact, secret stripping, the trust dialog in a session directory, a draft session with a stub MCP server configured at user level, a shell profile exporting a canary variable with a Bash call from the session, and a check that `validate` lists the profile's denies and warns `profile_missing_deny` for a profile without them); the macOS checklist.
**Status**: Not Started

## Version roadmap

- **0.x**: stages 0 to 6.
- **1.0** requires: the live suite green against released ticketctl, agent-spawn and claude-session; the interface unchanged for two minor releases; V-12 to V-19 settled.
- **1.x** (additive): window placement for interactive drafts; a gated automatic create (a config opt-in plus an enforced gate, never a bare `--yes`); sprint and epic lists in the prefetch; link creation once ticketctl offers it; a `pre-create` hook if a use appears; a `reconcile` command if label lookup in `list` proves too lazy; a TUI picker for drafts.
- **2.0**: nothing planned.
