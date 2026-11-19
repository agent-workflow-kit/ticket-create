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
- The draft session never holds the ticket system's credentials and never runs ticketctl; ticket-create is the only writer (architecture §13.1, ticketctl's trust boundary).
- Untrusted and model-written text travels only in files, never argv or env.

## Resolved open topics

**Draft template.** A small generic bundled skill writes a one-line title and Markdown sections (context, what to do, acceptance criteria, out of scope). A user's own format arrives as a template file keyed by issue type in `templates`, passed to the session as a trusted input file, or as a whole replacement skill. See [Slot contract](interface.md#slot-contract) and [Configuration](interface.md#configuration).

**Confirm flow.** ticket-create owns the confirmation. It runs ticketctl's `--dry-run`, shows its own review screen, reads one answer from `/dev/tty`, and only then calls ticketctl with `--yes --expect-plan <hash> --dedupe-key <key>`. The hash binds the human's yes to exactly the plan on the screen. See [Review screen](interface.md#review-screen), [Prompt and terminal rules](interface.md#prompt-and-terminal-rules) and [Create order](interface.md#create-order).

## Decisions

| # | Decision | Reason | Interface |
|---|---|---|---|
| D-1 | A saved draft with a `draft_id` generated before the session; `draft` stores, `create` confirms and writes | The dedupe key must be stable per draft and a retry must find the draft again | [Draft record and statuses](interface.md#draft-record-and-statuses) |
| D-2 | ticket-create owns the confirmation and passes `--yes --expect-plan` | An edit step needs its own screen; one prompt, not two; the hash binds the answer to the plan | [Create order](interface.md#create-order) |
| D-3 | `draft` takes optional seeds and fixed fields; nothing is required | The skill interviews; the wrapper supplies deterministic facts | [draft](interface.md#draft) |
| D-4 | Prefetch create-form facts, near-duplicates and the parent summary; a post-draft duplicate check shows warnings only | The session has no credentials; form facts make a draft creatable at the first dry-run | [Prefetch](interface.md#prefetch) |
| D-5 | A closed artifact schema bound to the run by `draft_id` and `run_token`; no skill marker | The human reviews every field, so a wrong-skill draft costs a retry and a marker would burden replacement skills | [Slot contract](interface.md#slot-contract) |
| D-6 | Sandbox off, no setting sources, an environment allowlist and a bundled permission profile, each replaceable in config | Design guide §11 says a wrapper's safety must not depend on another tool's config; an interactive trusted session cannot run sandboxed by default | [Session launch](interface.md#session-launch) |
| D-7 | The review screen offers yes, edit and cancel; other fields change through flags that are saved and marked `overridden`; `draft --continue` resumes the session | Covers the real cases without building a form | [draft](interface.md#draft), [create](interface.md#create), [Editor](interface.md#editor) |
| D-8 | The dedupe key is `ticket-create:<draft_id>` and attempts are recorded; a later create of an existing draft shows "already created" | A key containing title or type creates a duplicate after an edit (verified in a model of ticketctl's rule); a key without them makes ticketctl refuse safely | [Dedupe key and attempts](interface.md#dedupe-key-and-attempts) |
| D-9 | `--sprint` is resolved to a concrete sprint before the screen and added after the ticket exists | `create` has no sprint option; one honest yes; a failure leaves a ticket and a retryable step | [Sprint step](interface.md#sprint-step) |
| D-10 | `create` is terminal-only: no `--yes`, no pending store; headless drafts are the bridge for automation | The architecture's constraint; a stored draft already is the pending record | [Exit codes](interface.md#exit-codes), [draft](interface.md#draft) |
| D-11 | SQLite `state.db` plus public `log.jsonl` and `audit.jsonl`; retention 30 and 90 days | Design guide defaults; attempt history keeps the dedupe rule correct | [Files](interface.md#files), [Audit and log](interface.md#audit-and-log) |
| D-12 | Hooks `pre-draft`, `post-draft`, `post-create`; no `pre-create` | ticketctl's own `pre-create` runs after the confirmation and is where a gate belongs | [Hooks](interface.md#hooks) |
| D-13 | The command set and the exit-code table | The minimum that makes the decisions operable; 22 means a draft's content is missing or invalid | [Commands](interface.md#commands), [Exit codes](interface.md#exit-codes) |
| D-14 | Both siblings are binding A; claude-session and memoctl are not direct dependencies; `--placement here` only | Matches the architecture and the other wrappers; window placement can be added later | [Dependencies](interface.md#dependencies) |
| D-15 | A generic bundled template plus `templates` and `skill` in config | Organization formats stay out of the repository | [Slot contract](interface.md#slot-contract) |
| D-16 | The repo is optional and falls back through the project mapping and the cwd's `origin` | Notes and per-repo settings need a repo id, but many tickets have none | [draft](interface.md#draft-order) |
| D-17 | The review screen layout, the prompt, the terminal restore, the reader and the one sanitizer | Each rule closes an attack that a live spike demonstrated | [Review screen](interface.md#review-screen), [Prompt and terminal rules](interface.md#prompt-and-terminal-rules), [Artifact reader](interface.md#artifact-reader), [Output sanitizing](interface.md#output-sanitizing) |
| D-18 | Copy the artifact, then tear the agent down only when the draft is settled; otherwise leave it parked | A resumable conversation exactly when something went wrong; no residue otherwise | [Session outcomes](interface.md#session-outcomes) |
| D-19 | The config surface; per-repo overrides never select a program or a credential destination | Design guide §5 and §12 | [Configuration](interface.md#configuration) |
| D-20 | An audit pair per create and sprint attempt, plus a record per draft run | Design guide §6 and the `attempt_id` link that `awkit audit` pairs by | [Audit and log](interface.md#audit-and-log) |
| D-21 | A skill preflight for `plugin:skill` names and an exit-14 hint when the session never started working | A wrong skill name in an interactive session sits idle and exits 14 with no other signal | [Draft order](interface.md#draft-order), [Session outcomes](interface.md#session-outcomes) |
| D-22 | Interactive drafts run in the caller's cwd, headless drafts in the session directory | A human can answer the trust dialog and the skill can read the repo; unattended runs cannot | [Session launch](interface.md#session-launch) |

## Deviations from the design guide

- **Safety defaults (§11).** The `draft-ticket` kind is trusted and the default sandbox is `off`, with an empty setting-sources list. The environment allowlist and a bundled permission profile still apply. Each is a config key. ticket-work records the same kind of deviation.
- **No pending store (§4.4).** A command that requires a human exits 13 without a terminal and leaves nothing pending; the stored draft is the unit of work.
- **Exit 22 (§4.2).** The meaning is "a draft's content is missing or invalid", the same use as in review-run.

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
| V-11 | The dedupe key without title or type never creates a duplicate in any edit scenario | verified (model of ticketctl's rule) |
| V-12 | `ticketctl create --dry-run` with no values returns `required_fields`, and a probe with a reserved invalid type returns the allowed types as candidates | to verify when ticketctl ships; conservative default: show no form facts and warn |
| V-13 | The dry-run plan's `description` change includes ticketctl's attribution paragraph | to verify when ticketctl ships; conservative default: show a notice that ticketctl adds its attribution line |
| V-14 | A session directory shows (or does not show) Claude Code's trust dialog | to verify; conservative default: headless drafts pass a trusted directory only after the first interactive run |
| V-15 | Real `memoctl context` loads notes into a draft session when `--repo` is given | to verify when memoctl ships; the mechanism was verified with a stub command |
| V-16 | The macOS terminal, editor and termios behavior | macOS checklist |

## Modules

| Module | Responsibility |
|---|---|
| `cli` | argparse commands, global flags, exit codes, output and the error envelope |
| `config` | load, merge and validate config; precedence |
| `state` | SQLite access, migrations, the per-draft lease |
| `drafts` | the draft state machine and the attempts |
| `prefetch` | the ticketctl reads and the duplicate check |
| `prompt` | the prompt file, nonce fencing, truncation |
| `session` | the agent-spawn calls, the kind check, outcome mapping, label recovery |
| `artifact` | the strict reader and the slot validation |
| `review` | the review screen, the prompt, the editor and the terminal snapshot and restore |
| `sanitize` | the one sanitizer |
| `ticketctl_client` | the ticketctl calls and the mapping of its exits and plans |
| `audit` | `audit.jsonl` and `log.jsonl` |
| `hooks` | hook execution |
| `gateway` | every subprocess call |

## Data model

One table `drafts` (the fields of [Draft record and statuses](interface.md#draft-record-and-statuses)) and one `attempts` table keyed by draft and attempt number, plus a `meta` table for the schema version. Migrations are additive. Artifact text lives in the row; the database file is private.

## Concurrency

A per-draft lease with a holder PID, process start time and expiry, renewed while held. See [Concurrency](interface.md#concurrency).

## Error handling

Every lower tool's output is parsed tolerantly and its exit code mapped as in [Lower tool exits](interface.md#lower-tool-exits). Provider and hook text is untrusted and sanitized before it is printed. A crash between a launch and its stored result leaves a `drafting` row and a parked agent; both are found by label and resolved with `draft --continue` or `discard`.

## Test strategy

- All subprocess calls go through `gateway.py`; tests use fake `ticketctl` and `agent-spawn` programs in `tests/fakebin/` and copies of their schemas pinned to a version.
- Golden files for every `--json` output and hook payload; every output validates against `schemas/`.
- Terminal tests drive the review screen, the prompt and the editor through a pseudo-terminal harness, including the stale-input and escape-sequence cases.
- The reader and sanitizer attack corpus, and the dedupe scenario table (no edit, body edit, title edit, type edit, a re-run after success), run as unit tests.
- Tests marked `live` (never in CI) cover the verified facts above with a real `claude` on a cheap model.

## Dependencies

`pyyaml` (config, required by design guide §3), `pydantic` (config validation, §3). Everything else is the standard library. There is no TUI, so no Textual.
