# ticket-create: interface

Status: 0.1.0-design (2026-11-19). This document is the single normative source for ticket-create's rules. `design.md`, the README and the architecture link to its headings and never restate its tables. Section references (architecture §N, design guide §N) point to `../awkit-docs/architecture.md` and `../awkit-docs/design-guide.md`.

ticket-create drafts a ticket with an interactive Claude session and creates it after a human confirmation. `draft` launches a session of kind `draft-ticket` in the current terminal through agent-spawn, validates the artifact the session reports and stores it as a draft. `create` shows the draft on a review screen, lets the human edit it, and after one explicit yes writes it through ticketctl. It is a skill wrapper in layer 2 (architecture §5.2) and works without the harness.

## Overview

- **Draft.** One stored record per ticket under preparation, with a `draft_id` that ticket-create generates before the session starts. It holds the validated artifact, the human's overrides, the session's agent id and the history of create attempts. Everything a retry needs is in the draft.
- **Seed.** Optional text (`--idea`, `--idea-file`) and fixed fields (`--project`, `--type`, `--parent`, `--repo`, `--site`, `--ticket-label`) given to `draft`. Seed text is untrusted and fenced in the prompt. Fixed fields are trusted facts that the skill must not change without asking the human.
- **Run token.** A fresh random 128-bit value (32 lowercase hex characters) generated for every session run, stated in the prompt outside the fence and echoed by the skill in the artifact. It binds an artifact to the run that was launched for it.
- **Origin.** `interactive` for a draft made with a human in the session, `headless` for a draft made unattended. The review screen always shows it.
- **Attempt.** One call of `ticketctl create --yes` (or of the sprint step) with its dedupe key, plan hash and outcome, recorded in the draft.
- **Review screen.** The screen `create` shows before the one confirmation (see Review screen).
- **Human present.** Exactly when `/dev/tty` opens for reading and writing. No other test is used (see Prompt and terminal rules).

ticket-create never lets the draft session reach ticketctl: the session has no credentials and its permission profile denies ticketctl. ticket-create itself, outside the session, is the only process that runs `ticketctl create` and `ticketctl sprint add`.

## Commands

```text
ticket-create draft [--idea TEXT | --idea-file F|-] [--project P] [--type T] [--parent KEY] [--repo R] [--site S] [--ticket-label L]... [--mode interactive|headless] [--skill NAME] [--model M] [--timeout D] [--continue DRAFT_ID] [--no-review] [--dry-run] [--result-file PATH]
ticket-create create DRAFT_ID [--type T] [--project P] [--parent KEY] [--assign me|none|ACCOUNT] [--ticket-label L]... [--set NAME=VALUE|@FILE]... [--sprint open|next|ID] [--board B] [--site S] [--again] [--dry-run] [--result-file PATH]
ticket-create list [--status S]... [--origin interactive|headless] [--all] [--limit N]
ticket-create show DRAFT_ID [--body]
ticket-create discard DRAFT_ID... [--yes]
ticket-create prune [--dry-run]
ticket-create validate [--probe]
ticket-create export [--audit | --log] [--after-seq N]
```

A `DRAFT_ID` is the full 32 hex characters or any unique prefix of at least 6 (no match exits 10; several matches exit 21 `ambiguous_draft` with `details.candidates`).

### Global flags

`--json`, `--config PATH`, `--verbose`, `--quiet`, `--version`, `--actor STRING` (grammar `^(human|auto):[A-Za-z0-9._@-]{1,64}$`; the default is `human:<user>` when a human is present, else `auto:unknown`, where `<user>` is `me.name`) and `--decision-id ID` (repeatable, recorded in the audit). `--yes` exists only on `discard` (see below). There is no `--yes` on `create`: nothing makes ticket creation skip the human's answer.

With `--json`, errors print the envelope of Output by exit code and error envelope. `--result-file PATH` (on `draft` and `create`) also writes the `--json` result to PATH, atomically with mode 0600, whatever the exit code, replacing an existing file; a write failure is a warning on stderr and never changes the exit code. It exists for callers whose terminal the interactive session occupies.

### draft

Runs the Draft order. `--idea TEXT` is a convenience for a short trusted note by a human; its value is treated as untrusted like the file form. `--idea-file F` reads a regular file; `-` reads stdin to the end and is accepted only with `--mode headless` (an interactive session needs stdin for the terminal, so `-` with interactive exits 2 `stdin_conflict`).

An interactive draft needs a terminal on stdin and stdout (agent-spawn's `--placement here` rule). `--json` on stdout would follow the session's screen, so an interactive caller that wants machine output passes `--result-file`. A headless draft needs a seed (`--idea` or `--idea-file`) and runs with no terminal.

`--continue DRAFT_ID` resumes the draft's parked agent with `agent-spawn resume` and replaces the artifact when the session reports `done`. It is accepted for drafts in `drafting` (an orphan), `ready`, `failed` and `abandoned`, and refused for `created`, `sprint_failed` and `unknown` (exit 20 `already_created`). The fixed-field flags and `--idea` cannot be combined with `--continue` (exit 2 `invalid_combination`); a continuing session starts from the stored draft.

`--no-review` stops after the draft is stored. Without it, an interactive draft that reached `ready` on a terminal asks `Review and create now? [y/N]` (the same `/dev/tty` rules as the confirmation prompt) and on yes continues into the `create` flow for that draft.

`--dry-run` runs the Draft order up to and including agent-spawn's `launch --dry-run`, prints the resolved plan (site, project, repo, skill, kind check result, prefetch counts, prompt size, session settings) and launches nothing.

### create

Runs the Create order. It is the only command that writes to the ticket system. Flags override the stored draft: they are saved back to the draft (see Draft record), shown on the review screen as `overridden`, and take effect for the next dry-run. `--ticket-label` given on `create` replaces the draft's label list. `--set NAME=VALUE|@FILE` sets a field by its logical name or ticketctl's other accepted names; the value is inline or `@path` (rich text only via `@path`), saved in the draft. `--again` is human-only (it exits 2 `human_only` with an `auto:` actor) and starts a new attempt with the next counter after a recorded earlier attempt (see Dedupe key and attempts).

`--dry-run` runs everything up to the human prompt with no terminal needed: it prints the screen's facts (the resolved plan with the plan hash, warnings, required fields) and exits 0, or the failure that the screen would show. It records nothing except the stored overrides.

### list and show

`list` prints drafts newest first. By default it hides `created` drafts older than 7 days and all `discarded` rows; `--all` shows everything not yet pruned. `--status` filters (repeatable). Before printing it looks up parked agents by label (`owner=ticket-create`) and shows any whose draft has no result as an `orphan` row (see Labels). `--limit` is 1 to 500 (default 100).

`show` prints one draft: status, origin, site, project, type, repo, title, the attempts and, with `--body`, the body through the sanitizer. Without `--body` only the sizes and hashes are printed.

### discard

Removes drafts and tears down any parked agent of each. On a terminal it asks once per call (`Discard N draft(s)? [y/N]`, `/dev/tty` rules); `--yes` skips the question and is accepted from any actor because the action is local and does not publish (design guide §4.4). Without a human and without `--yes` it exits 13. A draft in `drafting` with a live session is refused (exit 20, `reason: live`). A `created`, `unknown` or `sprint_failed` draft is discarded without touching the ticket.

### prune

Deletes drafts past their retention (see `retention.open` and `retention.done`) and tears down their parked agents. `--dry-run` lists what it would remove. There is no cron block; run it by hand or from a harness rule.

### validate

Offline: loads the config (exit 2 `invalid_config` naming the key), checks that the sibling binaries resolve on `PATH`, runs the skill preflight for the configured skill and checks the template files. `--probe` additionally runs `ticketctl validate --probe --json`, `agent-spawn --version` and `ticketctl --version` against the pinned minimums and `agent-spawn launch --dry-run` with a probe prompt (the kind check, no model call). It prints `{valid, probe, checks[], warnings[]}` and exits 0 whenever it ran; a failed check is data, not an exit code, except a config that does not load.

### export

`export --audit` and `export --log` print the public JSON lines of `audit.jsonl` or `log.jsonl` after `--after-seq N` (a per-file sequence number carried in each record).

## Draft record and statuses

The `drafts` table row is the draft. The artifact copy, the overrides and the attempts are part of it.

| Status | Meaning | Leaves by |
|---|---|---|
| `drafting` | a session run is in progress, or a crash left a run without a result (the `orphan` case) | the run's outcome; `discard`; `prune` |
| `ready` | the artifact was validated and stored | `create`; `draft --continue`; `discard`; `prune` |
| `failed` | the run ended without a valid artifact; `failure` names the cause | `draft --continue`; `discard`; `prune` |
| `abandoned` | the human left the session without reporting (outcome exit 14) | `draft --continue`; `discard`; `prune` |
| `created` | a ticket exists for this draft (created or reused) | `discard`; `prune` |
| `sprint_failed` | the ticket exists but the sprint step did not complete | `create DRAFT_ID --sprint ...` (only the sprint step); `discard` |
| `unknown` | a create ended `outcome_unknown` | `create DRAFT_ID` (a re-run with the same key finds the ticket); `discard` |

`orphan` is not a stored status: it is how `list` presents a `drafting` draft whose recorded agent no longer has a live run, or an agent found by label that has no draft row. It is resolved by `draft --continue` or `discard`.

Stored fields: `draft_id`, `status`, `origin`, `created_at`, `updated_at`, `site`, `project`, `repo` (nullable), `type`, `title`, `body_md`, `parent`, `assignee`, `ticket_labels`, `fields`, `notes`, `overrides` (the names of fields the human changed), `agent_id`, `run_token` of the last run, `artifact` (`sha256`, `size`), `failure` (`code`, `outcome`), `created_key`, `created_url`, `sprint` (selector and resolved id and name) and the `attempts` list. Text fields hold your own draft text; the database is private (mode 0600 in a 0700 directory).

## Draft order

`draft` performs these steps in order; a failure before step 8 leaves the draft in `failed` (or no draft when step 2 has not run).

1. **Prepare.** Load config, resolve the site (flag, then `ticketctl.site`, then ticketctl's own selection with `ticketctl sites resolve`), project, type, parent and repo (flag, then the repo mapped to the project in `projects`, then the cwd's `origin` remote, then none). `--continue` loads the stored draft instead. Check the siblings against the pinned minimums (exit 12 `tool_missing` or `tool_too_old`, naming the install command), run the skill preflight (exit 12 `skill_not_found`), and refuse a headless draft with no seed (exit 2 `missing_seed`). An interactive draft with no terminal on stdin and stdout exits 2.
2. **Record.** Generate `draft_id` and `run_token`, take the per-draft lease and write the `drafting` row. The `pre-draft` hook runs here (exit 11 on a veto, 13 on an escalation; the row is deleted again).
3. **Prefetch** (see Prefetch). Failures here are warnings (the session simply gets less context) except an unreachable ticketctl with no way to know the project, which is exit 12.
4. **Prompt file.** Write the prompt file (see Prompt file) with mode 0600 under the state directory's `prompts/`, created `O_CREAT|O_EXCL|O_NOFOLLOW` with a random name, and the template file (if any) as a trusted `--input-file`.
5. **Kind check.** Run `agent-spawn launch --dry-run --json` with the final flags and require exit 0, `agent_settings.kind == "draft-ticket"`, `reads_third_party == false`, `would_refuse == null`, an empty top-level `warnings` and an empty `session_plan.warnings` and `session_plan.profile.fallback` not true (a misspelled kind exits 0 with only those signals).
6. **Launch.** Snapshot the terminal (termios) and run `agent-spawn launch` (see Session launch) with a fresh `--result-file` path under `results/`. Restore the terminal in a `finally` (see Prompt and terminal rules).
7. **Outcome.** Read the result file. If it is missing or unreadable, find the agent by its labels and read `agent-spawn show <agent_id> --json` instead. Map the outcome with the Session outcomes table. On `done`, read the frozen artifact with the Artifact reader and validate it against the Slot contract.
8. **Store.** Copy the validated artifact into the draft, set `ready` and tear the agent down with `agent-spawn teardown <agent_id>`. On any other outcome apply the table (the agent stays parked). Delete the prompt file and the result file.
9. **Finish.** Run the post-draft duplicate check (see Prefetch), the `post-draft` hook and the audit record, print the result and, on a terminal and without `--no-review`, offer the review.

## Create order

`create` performs these steps; every step that can fail before step 7 changes nothing in the ticket system.

1. **Gate.** Without a human present (`/dev/tty` does not open) exit 13 `needs_human` before anything else; `--dry-run` skips this step. Take the per-draft lease (exit 12 `unavailable`, reason `locked`). Refuse a draft that is not `ready`, `unknown` or `sprint_failed` (exit 20, `reason: draft_not_ready`; `created` is exit 20 `already_created` unless only a sprint step is requested).
2. **Overrides.** Apply the flags to the draft and save them. Resolve `--sprint` to a concrete sprint with `ticketctl sprints` (a selector such as `open` is replaced by its id and name; ambiguity exits 21 and a closed sprint exits 20 `sprint_closed`). `--board` is passed through to ticketctl.
3. **Plan.** Run `ticketctl create --dry-run --json --site S --project P --type T --title-file F --body-file B` with `--parent`, `--assign`, `--label`, `--set NAME=@FILE` and `--dedupe-key` as stored, passing every model-derived text through a 0600 file under `tmp/` and every other value only when it matches its closed grammar (see Slot contract). The result is a plan (or an error that the screen shows: `missing_required_fields` lists the fields to supply, an exit 22 shows ticketctl's reason, `already_exists` and a `dedupe_match` select the "already created" screen state).
4. **Duplicates.** Run the near-duplicate search on the final title (see Prefetch) and keep its hits as warnings.
5. **Review.** Show the review screen and read one answer. `e` edits the title and body (see Editor) and returns to step 3 with the new text. Anything other than `y` or `yes` ends with exit 20 `declined`, leaving the draft as it was (overrides stay saved).
6. **Hold the plan.** The answer is bound to the plan hash on the screen. If the plan changes before step 7 (a new override, an edit) the loop returns to step 3.
7. **Write.** Write the audit `attempted` record, record the attempt, then run `ticketctl create --yes --expect-plan HASH --dedupe-key KEY --actor human:<user>` with the same files and values as step 3. A ticketctl exit 20 `stale_ticket` returns to step 3 with the new plan. Exit 0 sets `created` (or `created` with `reused`); exit 12 `outcome_unknown` sets `unknown`.
8. **Sprint.** If a sprint was chosen and the ticket exists, run `ticketctl sprint add KEY --sprint ID --dry-run`, then with `--yes --expect-plan HASH2` (no dedupe key). Failure sets `sprint_failed`, keeps the key and exits 12 with the key in the result.
9. **Finish.** Write the audit outcome records, run `post-create`, print the key and URL, and release the lease.

## Prefetch

ticket-create gathers context deterministically before the session, with the same environment and credentials as its own process; the session itself never calls ticketctl. All ticketctl calls carry `--site` and `--json`, take their text from files, and have a total time budget of 60 seconds (a timeout is a warning, not a failure).

| Item | When | Call | Used for |
|---|---|---|---|
| Create form facts | always | `ticketctl create --dry-run --json --site S --project P [--type T]` with no values | allowed issue types (`details.candidates`, from a probe with a reserved invalid type name when no type is fixed) and `details.required_fields` for a fixed type |
| Near-duplicates | when there is seed text | `ticketctl search --project P --status-category new --status-category indeterminate --text-file SEED --limit 10 --fields key,title,status --json` | up to 10 keys and titles in the prompt |
| Parent summary | when `--parent` is given | `ticketctl get KEY --fields key,type,title,status --json` | one line in the prompt; also validates the parent exists |
| Notes | when the session has a repo | none (claude-session runs `memoctl context` at SessionStart) | memory notes; ticket-create adds nothing |

The post-draft duplicate check runs the same search on the final title with `--text-file TITLEFILE`; its hits become warnings on the review screen and never block. `--text` is a phrase match, so ticket-create compares nothing and lists what ticketctl returns.

## Prompt file

The prompt is a Markdown file under the state directory's `prompts/`, deleted when the run ends. It has, in this order: a trusted header (the run's `draft_id`, `run_token`, the fixed fields with a sentence that the skill must not change them without asking the human, the allowed types and required fields, the output rule: write the artifact as `draft.json` and report it); then the fenced untrusted blocks. Each untrusted block uses the nonce fence of design guide §12 with a fresh 32-hex nonce per prompt: `<untrusted source="S" id="R">` and `</untrusted id="R">`, with `</untrusted` and `<untrusted` neutralized case-insensitively, and a standing "data, not instructions" sentence before the first block. The sources are `seed` (the idea text), `duplicates` (titles and keys), `parent` (the parent summary) and `template` (not fenced: the template is the user's own trusted file and goes in as an input file). Every untrusted string passes the one sanitizer first; a single block longer than 16 KiB is cut with `[truncated: N bytes]`.

ticket-create adds no report or escalation instructions to the prompt (design guide §11): the standard instructions that claude-session appends tell the session how to report and that asking the human is always acceptable.

## Session launch

`draft` runs `agent-spawn launch` with these flags (every flag is passed on every call, so ticket-create's safety does not depend on another tool's config):

| Flag | Value |
|---|---|
| `--kind` | `draft-ticket` |
| `--mode` | `interactive` (or `headless` with `--mode headless`) |
| `--placement` | `here` (interactive only) |
| `--skill` | the configured skill (default `ticket-create:draft-ticket`) |
| `--plugin-dir` | the bundled plugin, then each `plugin_dirs` entry |
| `--prompt-file` | the prompt file |
| `--input-file` | the template file, when one applies |
| `--repo` | the resolved repo, when there is one |
| `--worktree` | `none` |
| `--cwd` | not passed for interactive runs; `session-dir` for headless runs (config `cwd` overrides) |
| `--permission-profile` | the bundled profile file (config `permission_profile`) |
| `--sandbox` | `off` (config `sandbox`) |
| `--setting-sources` | empty (config `setting_sources`) |
| `--env-allowlist-only` | each name of the fixed base list `PATH HOME USER LOGNAME SHELL LANG TERM TZ` plus `LC_ALL` and configured names |
| `--model`, `--timeout` | only when configured or given |
| `--no-implicit-done` | headless only |
| `--no-promote` | always |
| `--label` | `owner=ticket-create`, `draft=<draft_id>`, `repo=<id>` when there is a repo |
| `--actor` | `human:<user>` interactive, `auto:ticket-create` headless |
| `--result-file` | a fresh path under `results/` |
| `--json` | always |

ticket-create never passes `--ephemeral`, `--wait-timeout` (not allowed with `here`), `--yes`, `--force` or `--override-escalation`. The bundled permission profile sets `defaultMode` to `default`, allows `claude-session report` and writes under `$CSESSION_DIR/artifacts`, and denies `ticketctl`, `agent-spawn`, `ticket-create`, `memoctl delete`, `memoctl cron`, network tools, and writes to every awkit config and state directory. The bundled skill is a single `SKILL.md` with no other files (a second file the skill must read prompts the human for permission).

## Session outcomes

| Outcome (exit) | Draft status | Agent | ticket-create exit | Message |
|---|---|---|---|---|
| done (0), report `done`, `report.implicit` false, valid artifact | `ready` | torn down | 0 | the draft id and next step |
| done (0), artifact missing or invalid | `failed` (`failure.code: artifact_missing` or `artifact_invalid`) | parked | 22 | names the cause (wrong skill name, malformed file) and `draft --continue` |
| done (0) with `report.implicit` true (headless without a report) | `failed` (`implicit_done`) | parked | 22 | the session did not report |
| exited-unreported (14), `first_working_at` set | `abandoned` | parked | 14 | none beyond one line: no draft was made, `--continue` resumes |
| exited-unreported (14), `first_working_at` null | `abandoned` | parked | 14 | the session never started working; names the skill and suggests checking it |
| blocked or needs-input (13) | `failed` (`needs_human`) | parked | 13 | the report reason, sanitized |
| killed (15), timeout (16), lost (17), failed (18) | `failed` (`outcome`) | parked | the same code | the end reason |
| vetoed (11) or not started (12) | `failed` | none | the same code | the lower tool's reason |

In an interactive run a `needs-input` report does not end the session; the human keeps typing and a later `done` report supersedes it. Only a session that ends while still in `needs-input` produces exit 13.

## Slot contract

The `draft-ticket` slot (architecture §5.5) is filled by a skill that receives the prompt file and optionally an input file, and writes one JSON artifact. The skill's name is the config key `skill` (any name Claude Code accepts). A replacement skill must follow this contract.

Inputs: the prompt file (the first line the session sees gives the session directory) and, when a template applies, the template as a file under the session's `inputs/` directory.

Output: the skill writes `draft.json` directly inside `$CSESSION_DIR/artifacts/` and reports it as the artifact of its `done` report. The artifact is one JSON object with exactly these keys (unknown keys are refused):

| Key | Type | Rule |
|---|---|---|
| `schema_version` | integer | `1` |
| `draft_id` | string | equal to the prompt's `draft_id` |
| `run_token` | string | equal to the prompt's `run_token` |
| `type` | string | `^[A-Za-z0-9][A-Za-z0-9 _.-]{0,63}$` |
| `title` | string | 1 to `limits.title_chars` characters after trimming; one line (no line breaks) |
| `body_md` | string | Markdown restricted to what ticketctl converts (no tables, images or raw HTML); at most 32767 characters |
| `parent` | string or null | `^[A-Z][A-Z0-9_]{1,19}-[0-9]{1,9}$` |
| `labels` | array of string | at most 20; each `^[A-Za-z0-9][A-Za-z0-9._:-]{0,63}$` |
| `assignee` | string or null | `me`, `none` or `^[A-Za-z0-9:_-]{1,128}$` |
| `fields` | object | at most 20 entries; name 1 to 100 characters with no `=` and no control characters; value a string of at most 8192 characters, a number, or an array of at most 50 strings |
| `notes` | string | at most 4096 characters; shown to the human |

`parent`, `labels`, `assignee` and `fields` may be omitted. Values that go into argv (`type`, `parent`, `labels`, `assignee`) are restricted to closed character sets above because the artifact is untrusted; title, body and field values only ever travel in files.

A skill's `done` report must attach this artifact. A missing, wrong or unreadable artifact is exit 22; the draft's text is never trusted beyond these checks, and the human reviews every field before anything is sent.

## Artifact reader

One reader opens every untrusted file (the frozen artifact, the template, a `--idea-file`, the editor's file). It opens with `O_NOFOLLOW|O_NONBLOCK`, requires a regular file by `fstat`, checks the size against the cap before reading (`limits.artifact_bytes` for an artifact, 1 MiB for an idea file, 256 KiB for a template, `4 * 32767 + 1` bytes for the editor file), decodes strict UTF-8 without a byte order mark and, for JSON, rejects duplicate keys, `NaN` and `Infinity`, lone surrogates and a nesting depth over 8. For the frozen artifact it also checks that the file's `size` and `sha256` equal `run.report.artifact`, that `report.status` is `done` and `report.implicit` is false. Any failure is a validation failure with a `code` (`artifact_missing`, `artifact_invalid` with a `detail`: `frozen_file`, `json`, `schema`, `draft_id`, `run_token`, `field`), never an internal error; the validator catches every exception.

## Review screen

`create` shows the screen on the terminal's `/dev/tty` (and on stderr when that is the terminal), in this order from top to bottom:

1. The model-controlled body: the Markdown the human would send, as ticketctl will convert it, inside a gutter (every line prefixed with a box-drawing vertical bar and a space), capped at 200 lines and 20000 characters (lines over 200 characters are folded, runs of blank lines collapse to one). Every cut ends with `[... N more characters; sha256 HEX]`. The description shown is the `description` value of the dry-run plan's `changes[]`, which includes ticketctl's attribution paragraph.
2. The trusted facts block, last and directly above the prompt: site, project, type, title (capped at 500 characters), parent, assignee, labels, the fields, the resolved sprint (name and id), `overridden` marks, the draft's origin, the warnings (duplicates, near-limit, body differs from an earlier attempt) and the plan hash.
3. The prompt line.

A screen state changes the facts block and the prompt: `missing_required_fields` lists each field with its kind and allowed values, shows `create DRAFT_ID --set NAME=VALUE` as the way to supply it and offers no `y`; `already_created` shows the existing key and URL and offers accept (mark the draft done) or cancel only; a hook escalation (exit 13 from a ticketctl `pre-create` hook) is shown as a note (the human's answer overrides it only with `--override-escalation`, which ticket-create never passes).

Nothing in the body or the notes can be shown above the facts block as a summary, and no pager is used for model text. A cut body can be read in full with `e`.

## Prompt and terminal rules

- **Presence.** A human is present exactly when `os.open("/dev/tty", O_RDWR | O_NOCTTY)` succeeds. The state of stdin, stdout and stderr is never tested for this (spike evidence: it is wrong in both directions).
- **Prompt.** The confirmation reads one line from `/dev/tty` after forcing canonical mode, echo and signal generation, and flushing pending input with `tcflush(fd, TCIFLUSH)` before and again after the read. Only `y` or `yes` (trimmed, case-insensitive) accepts. Pending extra input, a partial line, an over-long line (over 64 characters), a bracketed paste, EOF and Ctrl-C all decline. The same prompt serves every yes-or-no question in the tool.
- **Restore.** Before launching the session, the editor or any child that uses the terminal, ticket-create takes a termios snapshot of `/dev/tty`. In a `finally` it restores it with `tcsetattr(TCSAFLUSH, snapshot)` and writes the reset sequence that turns off bracketed paste, mouse tracking and the alternate screen and shows the cursor. This happens on every exit, including a signal-killed child.
- **Cooked output.** Everything untrusted that is printed passes the Output sanitizing rule first.

## Editor

`e` writes `title`, a blank line and `body_md` to a file named at random with mode 0600 created `O_EXCL` inside a mode 0700 directory under the state directory's `tmp/`, and runs the editor command from config `editor` (default `$VISUAL`, then `$EDITOR`, then `vi`) with the file path as its last argument (the command string is split with `shlex`, never run through a shell; it comes from the user's own config or environment, never from a draft). The editor inherits the terminal; the snapshot and restore rules apply. After it exits, the file is re-read with the Artifact reader (a symlink or FIFO swap, a size over the cap, an empty result, a non-zero exit or a death by signal aborts the edit and keeps the draft unchanged), the first line becomes the title (at most `limits.title_chars`, no control characters) and the rest the body. The new text is saved to the draft, listed in `overrides` and sent through a new dry-run.

## Dedupe key and attempts

The dedupe key for a draft is `ticket-create:<draft_id>`. `--again` switches to `ticket-create:<draft_id>:<n>` where `n` is the number of earlier attempts that reached ticketctl, starting at 1. The key matches ticketctl's grammar and never contains title or type text; `draft_id` comes from ticket-create's own record, never from the artifact. ticketctl binds a key to the issue type and the title: the same key with the same type and title is `reused` (a changed body is not compared), and the same key with a different type or title is exit 20 `already_exists` (`details.existing_key`).

Every write to ticketctl is an attempt in the draft: `n`, the key, the plan hash, the start time, the outcome (`created`, `reused`, `unknown`, `failed`) and ticketctl's `attempt_id`. The review screen consults the attempts:

- If any attempt reached ticketctl (outcome `created`, `reused` or `unknown`), or the dry-run reports `dedupe_match`, or the draft has a `created_key`, the screen shows `already created as KEY`, the key's URL and, when the body differs from the attempt's body hash, a warning that body edits are not applied to an existing ticket. It offers `accept` (record the draft as `created` with that key) or cancel; `y` is not offered.
- A new attempt for the same draft needs `--again` (a human who has checked the ticket system). It uses the next counter key and runs the normal screen.
- After `outcome_unknown` the next `create DRAFT_ID` re-runs with the same key. If the ticket landed ticketctl answers `reused` (or `already_exists` after a title or type edit, which shows the same screen state); if it did not land the create proceeds.

## Sprint step

`--sprint` is accepted on `create`. It is resolved before the review screen (Create order step 2) and shown by name. After the ticket exists the step runs `ticketctl sprint add KEY --sprint ID --site S [--board B]` with its own dry-run and `--yes --expect-plan`; the human's yes covers it because the sprint was on the screen. `unchanged` (already in the sprint) is success. A `partial`, rejected or closed-sprint result sets `sprint_failed`, keeps `created_key` and exits 12 with the key, the failed step and ticketctl's reason in `details`. `create DRAFT_ID --sprint ID` on a `created` or `sprint_failed` draft runs only this step.

## Hooks

Hooks are configured in the tool's config (design guide §8). Each receives one JSON document on stdin and the environment `TICKET_CREATE_HOOK`, `TICKET_CREATE_ID` (the draft id) and `TICKET_CREATE_OBJECT_TYPE` (`draft`); they inherit the tool's environment minus the credential variables it names. Default timeout 30 seconds.

| Hook | Runs | Veto or escalation | Payload (beyond `tool`, `hook`, `object_type`, `object_id`) |
|---|---|---|---|
| `pre-draft` | before the session launches (Draft order step 2) | exit 11 on a veto, 13 on an escalation (the draft is removed) | `origin`, `site`, `project`, `type`, `mode`, `skill`, `has_seed`, `seed_bytes`, `continue` |
| `post-draft` | when a draft reaches `ready`, `failed` or `abandoned` | none | `origin`, `status`, `failure`, `outcome`, `site`, `project`, `type`, `sizes` (`title`, `body`), `artifact_sha256`, `event_id`, `dedupe_key` |
| `post-create` | after the create and sprint steps | none | `result` (`created`, `reused`, `partial`, `failed`, `unknown`), `site`, `project`, `key`, `sprint_step` (`none`, `done`, `failed`), `attempt` (`n`, `plan_sha256`), `event_id`, `dedupe_key` |

Payloads carry `reads_third_party: false` and `tainted: false`, ids, enums, counts, sizes and hashes, and never the title, body, notes or any model text. There is no `repo` or `labels` key (the tool has no repository notion of its own; a gate on these hooks is scoped `*`). There is no `pre-create`: ticketctl's own `pre-create` hook runs after the human's confirmation and before the provider write, and a gate belongs there.

## Exit codes

| Code | Meaning in ticket-create |
|---|---|
| 0 | a draft is stored, or a ticket is created or reused, or a read-only command succeeded |
| 1 | unexpected or internal error |
| 2 | usage error (reasons below) or `invalid_config` |
| 10 | draft not found |
| 11 | a `pre-draft` hook vetoed, or a lower tool refused the call |
| 12 | an expected failure: a lower tool failed (including `partial`, `outcome_unknown`, `rejected`, the sprint step), a sibling is missing or too old (`tool_missing`, `tool_too_old`), a credential is missing, `skill_not_found`, the session never started, or `locked` |
| 13 | a human is needed: `create` or `discard` without a human present, or the session asked for one |
| 14 to 18 | the draft session's outcome, passed through |
| 20 | refused, with a `reason` |
| 21 | ambiguous: a draft prefix, or a ticketctl site, board or sprint |
| 22 | a draft's content is missing or invalid: the artifact, or ticketctl's own input validation on `create --dry-run` |

Usage reasons (exit 2): `stdin_conflict`, `invalid_combination`, `missing_seed`, `human_only`, `invalid_argument`, `missing_option`, `no_terminal` (an interactive draft with no terminal). Refusal reasons (exit 20): `declined` (the human's answer was not yes; nothing is recorded as pending), `draft_not_ready`, `already_created`, `live`, `sprint_closed`, `already_exists`, `stale_plan`. A refusal always carries `reason` in `--json`.

Exit 130 is Ctrl-C on a prompt (declined) and exit 143 is `SIGTERM` before a session started; a `SIGTERM` after the session started exits 15 and leaves the agent parked. `SIGINT` while a session runs goes to the session's own terminal handling, as in agent-spawn.

## Lower tool exits

| Call | Exit | ticket-create does |
|---|---|---|
| `ticketctl create` or `sprint add` | 0 | records the key, URL and `reused`; `unchanged` is success |
| | 11 | shows the vetoing hook's reason (sanitized) and exits 11 |
| | 12 `outcome_unknown` | sets `unknown`; the next `create` re-runs with the same key |
| | 12 other (`auth_required`, `bad_credentials`, `forbidden`, `rejected`, `unavailable`, `timeout`, `rate_limited`, `partial`) | shows the code and message (sanitized) and exits 12 |
| | 13 `confirmation_required` or `escalated` | `confirmation_required` cannot happen with `--yes`; an escalation is shown as a screen note and the answer is not overridden |
| | 20 `stale_ticket` | re-plans and re-confirms |
| | 20 `already_exists` | the "already created" screen state |
| | 20 `declined` | cannot happen with `--yes`; treated as exit 1 if it does |
| | 21 | asks the human to pick through flags; exits 21 |
| | 22 | shows the reason in the screen; on `--dry-run` exits 22 |
| `agent-spawn launch` and `wait` | 0, 13 to 18 | the Session outcomes table |
| | 11, 12, 20, 21 | passed through with `lower_error` |
| | 22 | cannot happen without `--wait-timeout` |

Every lower tool's JSON is read tolerantly (unknown keys are ignored, a missing required key is a failure). Provider messages, hook stderr and `lower_error` text are untrusted and go through the sanitizer before printing.

## Output by exit code and error envelope

With `--json`, exit 0 prints the command's result object; exits 10 to 13, 20 and 21 print the error envelope and, where a run was adopted, the result as well; exits 2 and 1 print the envelope. The envelope is `{"error": {"code", "message", "reason", "retryable", "outcome_unknown", "draft_id", "details"}}`. `reason` is non-null for exits 2, 20 and 22. `details` may carry `candidates`, `required_fields`, `existing_key`, `plan`, `attempt`, `lower_error`, `hook` and `hook_stderr`; `message`, `lower_error`, `hook_stderr` and provider text are untrusted.

`draft` and `create` results carry the draft (id, status, origin, site, project, type, key and URL when created, the attempts) and, for a session, `agent_id`, `phase`, `outcome`, `end_reason` and `first_working_at`.

## Output sanitizing

Every untrusted string that leaves the process (human output, `--json` values, hook payload strings, logs) passes one sanitizer. It replaces C0 controls except newline and tab, DEL, C1 controls, bidirectional and isolate controls, zero-width and other format characters, the tag-character block, surrogates, private-use and unassigned code points, U+2028 and U+2029, non-space separators and variation selectors with `<U+XXXX>` markers, caps runs of combining marks at 8, and truncates lines over 200 characters with `<...>`. Terminal output of model text additionally goes through the Review screen's layout rule. The sanitizer is applied once, at the point of output.

## Objects

Draft: the stored fields of Draft record and statuses, as printed by `show --json`. Attempt: `n`, `key`, `plan_sha256`, `body_sha256`, `started_at`, `outcome`, `attempt_id`, `ticket_key`. Both are defined in `schemas/defs.json`.

## Configuration

Config is YAML at `~/.config/ticket-create/config.yaml` with drop-ins in `config.d/*.yaml` (design guide §5; `XDG_CONFIG_HOME` is respected). Precedence: flags, then `repos.<id>`, then environment, then global config, then defaults.

| Key | Default | Meaning |
|---|---|---|
| `me.name` | OS user | actor name for `human:<user>` |
| `ticketctl.config` | none | a ticketctl config file passed as `--config` |
| `ticketctl.site` | none | the default `--site` |
| `skill` | `ticket-create:draft-ticket` | the draft skill |
| `mode` | `interactive` | the default mode |
| `model`, `timeout` | none | passed to agent-spawn |
| `cwd` | caller (interactive), session-dir (headless) | the session's working directory |
| `sandbox` | `off` | `--sandbox` |
| `setting_sources` | `[]` | `--setting-sources` |
| `permission_profile` | bundled file | `--permission-profile` |
| `env_allowlist` | `[]` | names added to the fixed base list |
| `plugin_dirs` | `[]` | extra plugin directories |
| `templates` | none | map of type name or `default` to a template file |
| `projects.<site>/<KEY>.repo` | none | the repo id used when `--repo` is absent |
| `projects.<site>/<KEY>.default_type` | none | the type used when none is given |
| `duplicates.enabled`, `duplicates.limit` | `true`, `10` | the near-duplicate searches (limit 1 to 25) |
| `editor` | `$VISUAL`, `$EDITOR`, `vi` | the edit command |
| `retention.open`, `retention.done` | `30d`, `90d` | draft lifetimes (these keys also accept `h` and `d`) |
| `limits.artifact_bytes`, `limits.title_chars` | `65536`, `255` | reader caps |
| `hooks` | none | the hooks listed under Hooks |

`repos.<id>` may override `skill`, `templates`, `sandbox`, `setting_sources`, `env_allowlist`, `plugin_dirs`, `permission_profile`, `cwd` and `model`; it may not set `hooks`, `ticketctl.*` or `editor`, because a repository-level setting must never select a program to run or a place to send credentials. A template path or plugin directory must be absolute and owned by the user.

## Environment variables

Config keys take `TICKET_CREATE_<KEY>` (nested keys joined with `__`; scalar keys only). The tool reads `VISUAL`, `EDITOR`, `TERM`, `PATH`, `HOME` and `XDG_CONFIG_HOME` and `XDG_STATE_HOME`. It passes no credential of its own: ticketctl reads its own token variable, which must be listed in claude-session's `secret_env` so a draft session never receives it.

## Labels

Agents get `owner=ticket-create`, `draft=<draft_id>` and, when there is a repo, `repo=<id>`. These are the keys ticket-create uses to find its agents again (orphan recovery, `--continue`); `agent-spawn list --label owner=ticket-create --json` is the lookup. Ticket labels (Jira labels) are separate: `--ticket-label` and the artifact's `labels`. They are never agent labels and never contain whitespace.

## Files

All under `~/.local/state/ticket-create/` (`XDG_STATE_HOME` respected), directories mode 0700, files 0600: `state.db` (private SQLite, WAL), `log.jsonl` and `audit.jsonl` (public contracts), `prompts/`, `results/`, `tmp/` (random names, deleted when the call ends, stale files pruned after a day).

## Audit and log

`audit.jsonl` has one `attempted` record just before each `ticketctl create --yes` and `sprint add --yes` call and an outcome record after it, paired by `attempt_id` (ticket-create's own), plus one record per `draft` run. Fields: `ts`, `actor`, `decision_id` (list), `action` (`create`, `sprint_add`, `draft`), `target` (`<site>/<project>` or the draft id), `inputs` (`draft_id`, `dedupe_key`, `plan_sha256`, `body_sha256`, `origin`, `overrides`, `agent_id`), `result` (`key`, `reused`, `ticketctl_attempt_id`, `outcome`, `code`), `seq`. `log.jsonl` records each command with counts and hashes. Neither file holds the title, body, notes or any model text, and neither holds secrets.

## Concurrency

A per-draft lease (holder PID plus process start time plus expiry, renewed while held) is taken for `draft --continue`, `create`, `discard` and `prune`; a second caller exits 12 `unavailable` with `reason: locked`. Two `draft` runs for different drafts do not interact. `list` and `show` take no lease.

## Dependencies

| Tool | Binding | Minimum | Used for |
|---|---|---|---|
| ticketctl | A | set at its first release | the only writer: `create`, `sprint add`, and reads (`search`, `get`, `sprints`, `validate`) |
| agent-spawn | A | 0.34.0 | `launch`, `show`, `list`, `teardown`, `resume` |
| claude-session | none (through agent-spawn) | the version agent-spawn requires | its `secret_env` must name ticketctl's token variable; `notes_context` runs `memoctl context` |
| wtctl | none (agent-spawn requires it even with no worktree) | the version agent-spawn requires | installed for agent-spawn |

A missing sibling exits 12 `tool_missing` naming its install command; one below its pinned minimum exits 12 `tool_too_old`.

## Upgrades

The state database has a version and only additive migrations; a database newer than the tool exits 12 `schema_newer`. Config keys are added, never repurposed.

## Residual risks

- The draft session is a trusted, unsandboxed run (the `draft-ticket` kind is not third-party-reading). Text it is shown (seed, duplicate titles, a parent summary) is fenced but can steer its output and its tool use; its permission profile denies the dangerous tools and the human watches it, and a replaced profile or sandbox setting is the user's own choice.
- Nothing stops a process that can write to the terminal's input queue after the prompt is shown; the flush and the fixed layout cover input that arrived earlier.
- A process with the same user id can read `state.db`, the prompts and the drafts, and can edit the config.
- ticketctl's confirmation is not a boundary against a caller that can allocate a pty; ticket-create's boundary is that the draft session cannot run ticketctl and holds no credential.
- claude-session keeps a session's frozen artifact for 30 days after `forget`, so a draft's text outlives the draft there; `forget --purge` removes it.
- An unattended headless draft spends the user's model tokens and its cost is not recorded (interactive runs report no `cost_usd`).

## Platform notes

Linux and WSL are tested; macOS is a local checklist (`/dev/tty`, termios restore, the editor). The interactive draft needs no tmux. Python 3.12 or later.
