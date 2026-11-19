# ticket-create: design

Status: 0.1.0-design (2026-11-19). The normative rules are in [interface.md](interface.md); this document records the reasons, the decisions and the module plan, and links to the interface's headings instead of restating them.

## Role and layer

Layer 2 (composite). A skill wrapper that drafts tickets with an interactive Claude session and creates them through ticketctl after one human confirmation. It works fully standalone (architecture C2): no harness, bus or shared library.

## Uses

| Tool | Binding | Why |
|---|---|---|
| ticketctl | A (hard) | The only ticket writer. ticket-create parses its documented `--json` output and exit codes. There is no second provider in 0.x, so a configurable command (B) would be speculation. |
| agent-spawn | A (hard) | Launches the session in the current terminal (`--placement here`), supplies the result file, the parked-agent labels and the teardown. Same layer, allowed by architecture §3.2. |
| claude-session | none (through agent-spawn) | agent-spawn depends on it; ticket-create never calls it. Its `secret_env` list must name ticketctl's token variable. |
| memoctl | none | Notes reach the session through claude-session's `notes_context`; ticket-create only passes `--repo`. |

See [Dependencies](interface.md#dependencies) for the pinned minimums.

## Constraints (decided)

- `create` requires a human at a terminal and always confirms (architecture §5.2, §12.1, §12.4). There is no option to skip the confirmation and no pending store.
- The organization's ticket format lives in the user's config (a template file) or in a replacement skill, never in this repository.
- The draft session's launch environment holds no ticketctl credential and its permission profile denies ticketctl; ticket-create is the only process that runs `ticketctl create` and `sprint add`. This is guidance for a cooperative session, not a boundary (see D-25 and [Residual risks](interface.md#residual-risks)).
- Untrusted and model-written text travels only in files, never argv or env.

## Resolved open topics

**Draft template.** A small generic bundled skill writes a one-line title and Markdown sections (context, what to do, acceptance criteria, out of scope). A user's own format arrives as a template file in `templates` (looked up by the type given by flag or default type, else by `default`), passed to the session as the trusted input file `inputs/template.md`, or as a whole replacement skill. See [Slot contract](interface.md#slot-contract) and [Configuration](interface.md#configuration).

**Confirm flow.** ticket-create owns the confirmation. It runs ticketctl's `--dry-run`, shows its own review screen, reads one answer from `/dev/tty`, and only then calls ticketctl with `--yes --expect-plan <hash> --dedupe-key <key>`. The hash binds the human's yes to the plan on the screen; ticket-create also checks its own input files before the write, because the hash is not documented to cover the title, type or parent. A hook escalation after the yes is a second human question. See [Review screen](interface.md#review-screen), [Prompt and terminal rules](interface.md#prompt-and-terminal-rules) and [Create order](interface.md#create-order).

## Decisions

| # | Decision | Reason | Interface |
|---|---|---|---|
| D-1 | A saved draft with a `draft_id` generated before the session; `draft` stores, `create` confirms and writes; the statuses and the transitions between them are normative in the interface (D-23) | The dedupe key must be stable per draft and a retry must find the draft again | [Draft record and statuses](interface.md#draft-record-and-statuses) |
| D-2 | ticket-create owns the confirmation and passes `--yes --expect-plan`; the dry-run and the write run as the same human actor; a hook escalation is a second human question | An edit step needs its own screen; one prompt, not two; the hash binds the answer to the plan and ticket-create checks its own input files before the write; a different actor changes the attribution paragraph and so the hash | [Create order](interface.md#create-order) |
| D-3 | `draft` takes optional seeds and fixed fields; nothing is required | The skill interviews; the wrapper supplies deterministic facts | [draft](interface.md#draft) |
| D-4 | Prefetch create-form facts, near-duplicates and the parent summary (a phrase search on the first line of the seed, best effort); form facts travel in a fenced untrusted block; a post-draft duplicate check shows warnings only | The session has no credentials; form facts make a draft creatable at the first dry-run; provider text is untrusted | [Prefetch](interface.md#prefetch) |
| D-5 | A closed artifact schema bound to the run by `draft_id` and `run_token`; no skill marker | The human reviews every field, so a wrong-skill draft costs a retry and a marker would burden replacement skills | [Slot contract](interface.md#slot-contract) |
| D-6 | Sandbox off, no setting sources, an environment allowlist and a bundled permission profile, each replaceable in config; the profile is guidance, not a boundary (D-25) | Design guide §11 says a wrapper's safety must not depend on another tool's config; an interactive trusted session cannot run sandboxed by default | [Session launch](interface.md#session-launch) |
| D-7 | The review screen offers yes, edit and cancel, and `accept` when a ticket already exists; other fields change through flags that are saved and marked `overridden`; `draft --continue` resumes the parked session when it can and otherwise restarts from the stored draft | Covers the real cases without building a form | [draft](interface.md#draft), [create](interface.md#create), [Editor](interface.md#editor) |
| D-8 | The dedupe key is `ticket-create:<draft_id>` with a generation suffix after `create --again`; every `create` round's dry-run alone decides the screen state and attempt history only adds warnings; site and project are frozen once an attempt could have landed | A key containing title or type creates a duplicate after an edit (verified in a model of ticketctl's rule); a key without them makes ticketctl refuse safely; ticketctl's label and lookup are scoped by site and project | [Dedupe key and attempts](interface.md#dedupe-key-and-attempts) |
| D-9 | `--sprint` is resolved by ticketctl's rules before the screen and added after the ticket exists; a later sprint-only run asks again | `create` has no sprint option; one honest yes; a failure leaves a ticket and a retryable step | [Sprint step](interface.md#sprint-step) |
| D-10 | `create` is terminal-only: no `--yes`, no pending store; headless drafts are the bridge for automation | The architecture's constraint; a stored draft already is the pending record | [Exit codes](interface.md#exit-codes), [draft](interface.md#draft) |
| D-11 | SQLite `state.db` plus public `log.jsonl` and `audit.jsonl`; retention 30 and 90 days | Design guide defaults; attempt history keeps the dedupe rule correct | [Files](interface.md#files), [Audit and log](interface.md#audit-and-log) |
| D-12 | Hooks `pre-draft`, `post-draft`, `post-create`; no `pre-create` | ticketctl's own `pre-create` runs after the confirmation and is where a gate belongs | [Hooks](interface.md#hooks) |
| D-13 | The command set and the exit-code table | The minimum that makes the decisions operable; 22 means a draft's content is missing or invalid | [Commands](interface.md#commands), [Exit codes](interface.md#exit-codes) |
| D-14 | Both siblings are binding A; claude-session and memoctl are not direct dependencies; `--placement here` only | Matches the architecture and the other wrappers; window placement can be added later | [Dependencies](interface.md#dependencies) |
| D-15 | A generic bundled template plus `templates` and `skill` in config; the template has a fixed file name in the session and a type chosen by the session uses `templates.default` | Organization formats stay out of the repository | [Slot contract](interface.md#slot-contract) |
| D-16 | The repo is optional and falls back through the project mapping and the cwd's `origin`; site and project resolve by a fixed chain that ends in ticketctl's own offline selection | Notes and per-repo settings need a repo id, but many tickets have none; no new config key is needed | [draft](interface.md#draft-order) |
| D-17 | The review screen layout (gutter zone, facts block, prompt), the terminal restore, the reader and the one sanitizer, whose replacement set is the union with ticketctl's | Each rule closes an attack that a live check demonstrated; shown text must equal sent text | [Review screen](interface.md#review-screen), [Prompt and terminal rules](interface.md#prompt-and-terminal-rules), [Artifact reader](interface.md#artifact-reader), [Output sanitizing](interface.md#output-sanitizing) |
| D-18 | Copy the artifact, then tear the agent down only when the draft is settled; otherwise leave it parked; a run never destroys a stored valid artifact | A resumable conversation exactly when something went wrong; no residue otherwise | [Session outcomes](interface.md#session-outcomes) |
| D-19 | The config surface; per-repo overrides never select a program or a credential destination | Design guide §5 and §12 | [Configuration](interface.md#configuration) |
| D-20 | An audit pair per create and sprint attempt, plus a record per draft run | Design guide §6 and the `attempt_id` link that `awkit audit` pairs by | [Audit and log](interface.md#audit-and-log) |
| D-21 | A skill preflight for `plugin:skill` names and an exit-14 hint when the session never started working; a headless exit 14 is `failed` with `no_report`, an interactive one is `abandoned` | A wrong skill name in an interactive session sits idle and exits 14 with no other signal; `abandoned` means a human left | [Draft order](interface.md#draft-order), [Session outcomes](interface.md#session-outcomes) |
| D-22 | Interactive drafts run in the caller's cwd, headless drafts in the session directory | A human can answer the trust dialog and the skill can read the repo; unattended runs cannot | [Session launch](interface.md#session-launch) |
| D-23 | The draft state machine (statuses, `rest_status`, transitions, gate table) and the attempt rules are normative in the interface; the attempt outcome starts as `unknown` and is overwritten when ticketctl answers | One transition function removes the unstated outcomes; a crash needs no extra state | [Draft record and statuses](interface.md#draft-record-and-statuses), [Dedupe key and attempts](interface.md#dedupe-key-and-attempts) |
| D-24 | The review screen puts every model-written value (body, notes, field values, duplicate titles, sprint name) in the bar-prefixed zone above the line under one line budget that fits the terminal height; below the line only checked single-line values and the quoted title | One rule a human can learn: a bar means model text; forged lines cannot scroll the real facts off | [Review screen](interface.md#review-screen) |
| D-25 | The bundled permission profile denies `mcp__*`, `memoctl add` and the awkit shell paths; users who want connectors replace the profile; the profile is guidance, not a boundary | A connector that writes tickets would be a second write path around the screen; a shell profile, an approved tool call or an inherited MCP server can still reach one | [Session launch](interface.md#session-launch), [Residual risks](interface.md#residual-risks) |
| D-26 | Environment variables set only `mode`, `model`, `timeout`, `duplicates.*` and `retention.*`; every other key comes from config files and flags | Repository tooling sets the environment, and a repository-level value must never select a program or a place to send credentials | [Environment variables](interface.md#environment-variables) |
| D-27 | A config key `credential_env` (variable names, default empty) is removed from the environment of hooks and the editor; `validate` warns while it is empty | ticket-create names no token variable of its own, and hooks and the editor would otherwise see ticketctl's | [Configuration](interface.md#configuration), [validate](interface.md#validate) |

## Deviations from the design guide

- **Safety defaults (§11).** The `draft-ticket` kind is trusted and the default sandbox is `off`, with an empty setting-sources list. The environment allowlist and a bundled permission profile still apply. Each is a config key. ticket-work records the same kind of deviation.
- **No pending store (§4.4).** A command that requires a human exits 13 without a terminal and leaves nothing pending; the stored draft is the unit of work.
- **Exit 22 (§4.2).** The meaning is "a draft's content is missing or invalid", the same use as in review-run; it also covers ticketctl's own input validation on a dry-run.
- **Environment layer (§5).** Only harmless keys read `TICKET_CREATE_<KEY>` (D-26).
- **`LC_*` narrowing.** The approved base environment list had `LC_*`; claude-session's `--env-allowlist-only` takes plain names, so only `LC_ALL` is on the list.
- **`--json` on the interactive launch.** `draft` also passes `--result-file`; the interactive launch prints one result line after the session. If agent-spawn writes the file without `--json`, `--json` is omitted (a Stage 4 live check, V-18).

## Live verification

| # | Fact | Status |
|---|---|---|
| V-1 | An interactive `draft-ticket` run under `--placement here` ends with exit 0 when the skill reports `done` with an artifact, with auto-exit returning control | verified (agent-spawn 0.34.0, claude-session 0.45.2) |
| V-2 | Quitting before a report exits 14 with `report: null`; a wrong skill name sits idle and exits 14 with `first_working_at` null | verified |
| V-3 | `needs-input` does not end an interactive session; a later `done` supersedes it; ending in `needs-input` exits 13 | verified |
| V-4 | A `done` with no artifact, a non-JSON artifact and a wrong value are all accepted by claude-session and frozen byte for byte | verified |
| V-5 | The frozen artifact is mode 0600 in a 0700 directory; `--ephemeral` does not remove it; `forget` keeps it 30 days, `forget --purge` removes it | verified |
| V-6 | A token-like variable reaches the session by default; `secret_env` and `--env-allowlist-only` remove it with nothing else breaking | verified |
| V-7 | A `--result-file` write failure is only a warning; `--placement here` with no terminal on stdin and stdout exits 2 and leaves no state | verified |
| V-8 | The session needs wtctl installed even with `--worktree none` | verified |
| V-9 | `/dev/tty` opening is the only reliable "a human can be asked" test; an early typed `y` is taken as the answer unless input is flushed | verified (local terminal harness) |
| V-10 | The reader, the sanitizer and the editor rules withstand the attack corpus (29 file and structure cases, 28 display cases) | verified |
| V-11 | The dedupe key without title or type never creates a duplicate in any edit scenario; scenario rows also cover a project override and a site override after an attempt (exit 20 `attempt_recorded`) | verified (model of ticketctl's rule); the two override rows are checked in the dedupe scenario table |
| V-12 | `ticketctl create --dry-run` with no values returns `required_fields`, and a probe with the type `ticket-create-probe` returns the allowed types as candidates | documented in ticketctl's interface; the live check remains; a failure is the warning `form_facts_unavailable` |
| V-13 | The dry-run plan's `description` change includes ticketctl's attribution paragraph | documented in ticketctl's interface; the live check remains |
| V-14 | A session directory shows (or does not show) Claude Code's trust dialog | to verify; headless drafts always pass `session-dir`, so no default is needed |
| V-15 | Real `memoctl context` loads notes into a draft session when `--repo` is given | to verify when memoctl ships; the mechanism was verified with a stub command |
| V-16 | The macOS terminal, editor and termios behavior | macOS checklist |
| V-17 | A headless `launch --mode headless --no-implicit-done --wait --result-file R --json` exits with the run's outcome code and writes a result object with `run.outcome` and `run.report` | to verify live (Stage 4) |
| V-18 | The interactive launch with `--result-file` and `--json` prints one result line after the session and writes the file | to verify live (Stage 4) |

## Modules

| Module | Responsibility |
|---|---|
| `cli` | argparse commands, global flags, exit codes, output and the error envelope |
| `config` | load, merge and validate config; precedence; the environment layer limited to harmless keys; `credential_env` |
| `state` | SQLite access, migrations, the per-draft lease |
| `drafts` | the transition table, the gate table, the generations and the attempts |
| `resolve` | the site, project, type and repo chain, the skill preflight and the sprint selectors |
| `prefetch` | the ticketctl reads and the duplicate check |
| `prompt` | the prompt file, nonce fencing, truncation |
| `session` | the agent-spawn calls (launch and resume), the kind check, outcome mapping, label recovery, the continue order |
| `artifact` | the strict reader and the slot validation |
| `review` | the review screen (gutter zone, facts block, budget), the prompt, the editor and the terminal snapshot and restore |
| `sanitize` | the one sanitizer |
| `ticketctl_client` | the ticketctl calls and the mapping of its exits and plans |
| `audit` | `audit.jsonl` and `log.jsonl` |
| `hooks` | hook execution |
| `gateway` | every subprocess call |

## Data model

One table `drafts` (the fields of [Draft record and statuses](interface.md#draft-record-and-statuses), including `generation` and the `sprint` step state) and one `attempts` table keyed by draft and attempt number `n` (with `generation`, `site`, `project`, `attempt_id` and `ticketctl_attempt_id`), plus a `meta` table for the schema version. Attempts are append-only and are deleted only with the row. Migrations are additive. Artifact text lives in the row; the database file is private.

## Concurrency

A per-draft lease with a holder PID, process start time and expiry; its time to live is 60 seconds and a background thread of the holder renews it every 20 seconds, including while the process waits for a session or a prompt. See [Concurrency](interface.md#concurrency).

## Error handling

Every lower tool's output is parsed tolerantly and its exit code mapped as in [Lower tool exits](interface.md#lower-tool-exits). Provider and hook text is untrusted and sanitized before it is printed. A crash leaves a `drafting` row; its agent is found by label and the row is resolved with `draft --continue` or `discard`. A crash during a write leaves an attempt with outcome `unknown` and the status `unknown`; the next `create` runs the dry-run and decides from ticketctl's answer.

## Test strategy

- All subprocess calls go through `gateway.py`; tests use fake `ticketctl` and `agent-spawn` programs in `tests/fakebin/` and copies of their schemas pinned to a version.
- Golden files for every `--json` output and hook payload; every output validates against `schemas/`.
- A test generated from the interface's transition table, one case per row, and a test of the gate table, one case per cell.
- Terminal tests drive the review screen, the prompt and the editor through a pseudo-terminal harness, including the stale-input and escape-sequence cases, the forged-fields attack at 24 rows, the fold with the gutter at 80 columns and the too-small terminal.
- The reader and sanitizer attack corpus, and the dedupe scenario table (no edit, body edit, title edit, type edit, project override, site override, a re-run after success, a retry after `outcome_unknown`), run as unit tests.
- Tests marked `live` (never in CI) cover the verified facts above with a real `claude` on a cheap model, and also: a resume of an agent that never worked is refused by our own check; a draft session with a stub MCP server at user level; a shell profile that exports a canary variable and a Bash call from the session; the headless `--wait` result (V-17).

## Dependencies

`pyyaml` (config, required by design guide §3), `pydantic` (config validation, §3). Everything else is the standard library. There is no TUI, so no Textual.
