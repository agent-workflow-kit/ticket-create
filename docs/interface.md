# ticket-create: interface

Status: 0.1.0-design (2026-11-19). This document is the single normative source for ticket-create's rules. `design.md`, the README and the architecture link to its headings and never restate its tables. Section references (architecture §N, design guide §N) point to `../awkit-docs/architecture.md` and `../awkit-docs/design-guide.md`.

ticket-create drafts a ticket with an interactive Claude session and creates it after a human confirmation. `draft` launches a session of kind `draft-ticket` in the current terminal through agent-spawn, validates the artifact the session reports and stores it as a draft. `create` shows the draft on a review screen, lets the human edit it, and after one explicit yes writes it through ticketctl. It is a skill wrapper in layer 2 (architecture §5.2) and works without the harness.

## Overview

- **Draft.** One stored record per ticket under preparation, with a `draft_id` that ticket-create generates before the session starts. It holds the validated artifact, the human's overrides, the session's agent id and the history of create attempts. Everything a retry needs is in the draft.
- **Seed.** Optional text (`--idea`, `--idea-file`) and fixed fields (`--project`, `--type`, `--parent`, `--repo`, `--site`, `--ticket-label`) given to `draft`. Seed text is untrusted and fenced in the prompt. Fixed fields are trusted facts that the skill must not change without asking the human; a fixed field the artifact changed is shown with the warning `fixed_field_changed` and the human decides.
- **Run token.** A fresh random 128-bit value (32 lowercase hex characters) generated for every session run, including a continuation, and stated in the prompt outside the fence. The artifact echoes the token of the newest prompt. It binds an artifact to the run that was launched for it.
- **Origin.** `interactive` for a draft made with a human in the session, `headless` for a draft made unattended. It is set from the mode of the run that last stored the artifact. The review screen always shows it.
- **Attempt.** One call of `ticketctl create --yes` with its dedupe key, plan hash and outcome, recorded in the draft. The sprint step is not an attempt.
- **Review screen.** The screen `create` shows before the one confirmation (see Review screen).
- **Human present.** Exactly when `/dev/tty` opens for reading and writing. No other test is used (see Prompt and terminal rules).

ticket-create never lets the draft session reach ticketctl through its own call: the session's launch environment holds no ticketctl credential and its permission profile denies ticketctl. This is guidance for a cooperative session, not a boundary against a shell profile that exports the token again, an MCP connector, or a human who approves a tool call (see Residual risks); ticket-create itself is the only process that runs `ticketctl create` and `ticketctl sprint add`.

## Commands

```text
ticket-create draft [--idea TEXT | --idea-file F|-] [--project P] [--type T] [--parent KEY] [--repo R] [--site S] [--ticket-label L]... [--mode interactive|headless] [--skill NAME] [--model M] [--timeout D] [--continue DRAFT_ID] [--no-review] [--dry-run] [--result-file PATH]
ticket-create create DRAFT_ID [--type T] [--project P] [--parent KEY] [--assign me|none|ACCOUNT] [--ticket-label L]... [--set NAME=VALUE|@FILE]... [--sprint open|next|ID] [--board B] [--site S] [--again] [--dry-run] [--result-file PATH]
ticket-create list [--status S]... [--origin interactive|headless] [--all] [--limit N] [--no-agents]
ticket-create show DRAFT_ID [--body] [--field NAME]
ticket-create discard DRAFT_ID... [--yes]
ticket-create prune [--dry-run]
ticket-create validate [--probe]
ticket-create export (--audit | --log) [--after-seq N]
```

A `DRAFT_ID` is the full 32 hex characters or any unique prefix of at least 6 characters; a shorter prefix exits 2 `invalid_argument`, no match exits 10, and several matches exit 21 `ambiguous_draft` with `details.candidates`. `export` needs exactly one of `--audit` and `--log` (else exit 2 `missing_option`).

Flag grammars: `--project` `^[A-Z][A-Z0-9_]*$`; `--site` `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`; `--assign` `me`, `none` or `^[A-Za-z0-9][A-Za-z0-9:_.-]{0,127}$`; `--sprint` `open`, `next` or `^[0-9]{1,18}$`; `--board` an id or a name of 1 to 128 characters with no control characters; `--ticket-label` the artifact label grammar, and `--ticket-label ''` clears the list. A value that does not match exits 2 `invalid_argument`.

### Global flags

`--json`, `--config PATH`, `--verbose`, `--quiet`, `--version`, `--actor STRING` (grammar `^(human|auto):[A-Za-z0-9._@-]{1,64}$`; the default is `human:<user>` when a human is present, else `auto:unknown`, where `<user>` is `me.name` normalized to the actor grammar: disallowed characters become `_` and the name is cut to 64 characters) and `--decision-id ID` (repeatable, recorded in the audit and forwarded to every ticketctl write and every agent-spawn launch and resume). `--yes` exists only on `discard` (see below). There is no `--yes` on `create`: nothing makes ticket creation skip the human's answer. `create` exits 2 `human_only` with an `auto:` actor (not with `--dry-run`, which builds its plan as `human:<user>`).

With `--json`, exit 0 prints the result object; an exit that adopted a run or built a plan prints that one document, the result object with the error as its `error` member; other failures print the envelope of Output by exit code and error envelope. `--result-file PATH` (on `draft` and `create`) holds the same document, written atomically with mode 0600, whatever the exit code, replacing an existing file; a write failure is a warning on stderr and never changes the exit code. It exists for callers whose terminal the interactive session occupies. The member names of the result objects are those of `schemas/cli/`.

### draft

Runs the Draft order. `--idea TEXT` is a convenience for a short trusted note by a human; its value is treated as untrusted like the file form. `--idea-file F` reads a regular file; `-` reads stdin to the end and is accepted only with `--mode headless` (an interactive session needs stdin for the terminal, so `-` with interactive exits 2 `stdin_conflict`).

An interactive draft needs a terminal on stdin and stdout (agent-spawn's `--placement here` rule). `--json` on stdout would follow the session's screen, so an interactive caller that wants machine output passes `--result-file`. A headless draft needs a seed (`--idea` or `--idea-file`) and runs with no terminal.

`--continue DRAFT_ID` follows the Continue order. It is accepted for drafts in `drafting` (an orphan), `ready`, `failed` and `abandoned` (see the Gate table).

`--no-review` stops after the draft is stored. Without it, an interactive draft that reached `ready` on a terminal asks `Review and create now? [y/N]` (the same `/dev/tty` rules as the confirmation prompt) and on yes continues into the `create` flow for that draft; after that question `--json` and `--result-file` carry the `draft` result.

`--dry-run` writes no row and takes no lease, runs no hook, needs no terminal, runs the read-only prefetch, writes the prompt file under `tmp/` and deletes it, and runs the kind check; it prints the plan (site, project, repo, skill, kind check result, prefetch counts, prompt size, session settings) with an in-memory `draft_id` and launches nothing.

### create

Runs the Create order. It is the only command that writes to the ticket system. Flags override the stored draft: they are applied in memory and saved to the draft (see Draft record and statuses) only by a call that reaches the prompt with a human present; they are shown on the review screen as `overridden`, and `--dry-run` saves nothing. `--ticket-label` given on `create` replaces the draft's label list. `--set NAME=VALUE|@FILE` sets a field by its logical name or ticketctl's other accepted names; an inline value is a string, an array only through `@FILE`, rich text only via `@FILE`, and `--set NAME=` removes the field from the draft. `--set NAME=@FILE` is read by the Artifact reader. `--again` starts a new generation (see Dedupe key and attempts).

`--dry-run` runs everything up to the human prompt with no terminal needed: it prints the screen's facts (the resolved plan with the plan hash, the screen state, warnings, required fields) and exits 0 for the `ready` and `already_created` states (the state is in the result) and 22 for `blocked`.

### list and show

`list` prints drafts newest first. By default it hides `created` drafts older than 7 days; `--all` shows everything not yet pruned. `--status` filters (repeatable). Unless `--no-agents` is given, which makes it offline, it looks up parked agents by label (`owner=ticket-create`, see Labels). Rows carry `failure` and `updated_at`; a `drafting` row whose lease is free is shown as `orphan`. There is no `discarded` status (`discard` deletes). `--limit` is 1 to 500 (default 100).

`show` prints one draft: status, origin, site, project, type, repo, title, the attempts and, with `--body`, the body through the sanitizer; it never prints the `run_token`. Without `--body` only the sizes and hashes are printed. `--field NAME` prints one field's value through the sanitizer as text (it cannot be combined with `--json`).

### discard

Removes drafts and tears down any parked agent of each. On a terminal it asks once per call (`Discard N draft(s)? [y/N]`, `/dev/tty` rules; for an `unknown` draft the prompt says a ticket may exist); `--yes` skips the question and is accepted from any actor because the action is local and does not publish (design guide §4.4). Without a human and without `--yes` it exits 13. A draft in `drafting` with a live session or lease is refused (exit 20, `reason: live`). A `created`, `unknown` or `sprint_failed` draft is discarded without touching the ticket. With any refused item the call exits 20 and prints every item's result; a refused or `teardown_failed` item keeps its row. `discard` records the agent's `csession_id` in the output and prints `claude-session forget --purge ID` as the way to delete the frozen artifact early.

### prune

Deletes drafts past their retention (see `retention.open` and `retention.done`) and tears down their parked agents. It skips a draft whose lease is held or whose agent is live (result `skipped`, `reason: locked` or `live`) and exits 0. It also removes `tmp/` files older than a day. `--dry-run` lists what it would remove. There is no cron block; run it by hand or from a harness rule.

### validate

Offline: loads the config (exit 2 `invalid_config` naming the key), checks that the sibling binaries resolve on `PATH` (exit 12 `tool_missing` when one is missing), runs the skill preflight for the configured skill and checks the template files. `--probe` additionally runs `ticketctl validate --probe --json`, `agent-spawn --version` and `ticketctl --version` against the pinned minimums (exit 12 `tool_too_old`) and `agent-spawn launch --dry-run` with a probe prompt (the kind check, no model call). The checks are named `config`, `siblings`, `skill`, `templates`, `ticketctl`, `versions` and `kind_check`. `validate` also lists the resolved permission profile's denies (`profile_denies`) and warns when `credential_env` is empty or the kind check cannot show MCP state. It prints `{valid, probe, checks[], profile_denies[], warnings[]}`; any other failed check is data with exit 0.

### export

`export --audit` and `export --log` print the public JSON lines of `audit.jsonl` or `log.jsonl` after `--after-seq N` (a per-file sequence number carried in each record). `export --json` prints JSON lines too; each line is one record.

## Draft record and statuses

The `drafts` table row is the draft. The artifact copy, the overrides and the attempts are part of it.

| Status | Meaning |
|---|---|
| `drafting` | a session run is in progress, or a crash left one unfinished (`list` shows it as `orphan` when no process holds its lease) |
| `ready` | a validated artifact is stored and no write to the ticket system is outstanding; a non-null `failure` means the last `draft --continue` run failed and the stored artifact is still the last good one |
| `failed` | a run ended without a valid artifact and none is stored; `failure` names the cause |
| `abandoned` | a human left an interactive session without reporting and no artifact is stored |
| `created` | a ticket exists for the draft (`created_key`) and the sprint step is done or was not requested |
| `sprint_failed` | the ticket exists and the sprint step failed |
| `unknown` | a write to the ticket system was started and its outcome is not known |

`orphan` is not a stored status: it applies to `drafting` rows only and is how `list` presents a row whose lease is free. It is resolved by `draft --continue` or `discard`.

`rest_status(draft)` is the status of a draft when no write is in flight: `unknown` when any attempt of the current generation has outcome `unknown`; else `sprint_failed` when `created_key` is set and the sprint step failed; else `created` when `created_key` is set; else `ready`.

### Transitions

| # | From | Event | To | Attempts and notes |
|---|---|---|---|---|
| T1 | none | `draft` passes its checks and the `pre-draft` hook | `drafting` | row with `run_token`; no attempts |
| T2 | none | `pre-draft` veto (exit 11) or escalation (exit 13) | none | the row is not kept |
| T3 | `drafting` (orphan), `ready`, `failed`, `abandoned` | `draft --continue` passes the gate | `drafting` | new `run_token`; a stored artifact and the attempts are kept |
| T4 | the states of T3 | `--continue` vetoed, escalated or refused | unchanged | nothing is written |
| T5 | `drafting` | the run ends `done` with a valid artifact | `ready` | artifact stored, `origin` is the mode of this run, `failure` cleared, agent torn down |
| T6 | `drafting` | any other run outcome (Session outcomes table) | `ready` with `failure` set if an artifact was stored before this run, else `failed` or `abandoned` as the table says | the agent stays parked |
| T7 | `drafting` | the process dies (kill, reboot) | `drafting` | `list` shows `orphan` while no process holds the lease |
| T8 | `created`, `sprint_failed`, `unknown` | `draft --continue` | unchanged | exit 20: `already_created` for the first two, `draft_not_ready` for `unknown` |
| T9 | `ready`, `unknown` | `create` ends before any write (no human, decline, Ctrl-C, blocked screen, an error before the write) | unchanged | overrides given by a human are saved; no attempt |
| T10 | `ready`, `unknown`, `created`, `sprint_failed` | the write starts (`ticketctl create --yes` is about to run) | `unknown` | in one transaction: `generation` set, attempt inserted with outcome `unknown`, status set |
| T11 | `unknown` (from T10) | ticketctl exit 0 (`created` or `reused`) | `created` | attempt `created` or `reused`; `created_key`, `created_url` set |
| T12 | `unknown` (from T10) | exit 20 `stale_ticket` | `rest_status` | attempt `failed`; the loop re-plans |
| T13 | `unknown` (from T10) | exit 20 `already_exists` | `rest_status` | attempt `failed`; the screen shows the `already_created` state |
| T14 | `unknown` (from T10) | exit 11, 13, 21, 22, 10, 2, or exit 12 with any code except `outcome_unknown` | `rest_status` | attempt `failed` |
| T15 | `unknown` (from T10) | exit 12 `outcome_unknown`, exit 1, death by signal, unreadable output, or a crash of ticket-create | `unknown` | attempt stays `unknown` |
| T16 | `ready`, `unknown` | the human answers `accept` on the `already_created` state | `created` | attempt `reused` with the existing key |
| T17 | `created` (from T11) | the sprint step was requested and fails | `sprint_failed` | key kept; exit 12 |
| T18 | `created`, `sprint_failed` | a sprint-only run succeeds or fails | `created` or `sprint_failed` | no attempt; `sprint.step` records it |
| T19 | any except a `drafting` draft with a live session or lease | `discard` | row deleted | the parked agent is torn down; refused with `reason: live` otherwise |
| T20 | any not locked or live | `prune` after retention | row deleted | the parked agent is torn down |

Attempts are append-only. They are never changed after the call ends, `--continue` never touches them, and they are deleted only with the row. The attempt `outcome` starts as `unknown` and is overwritten once ticketctl's answer is known, so a crash needs no extra state. `generation` starts at 0, `--again` adds 1, and the dedupe key of generation g is `ticket-create:<draft_id>` for g = 0 and `ticket-create:<draft_id>:<g>` otherwise.

### Gate

| Status | `create` | `create --again` | `create --sprint S` | `draft --continue` | `discard` |
|---|---|---|---|---|---|
| `drafting` | exit 20 `draft_not_ready` | same | same | allowed only for an orphan (no live session, no lease) | exit 20 `live` while the session or lease is live |
| `ready` | full create | exit 2 `invalid_combination` | an override of the full create | allowed | allowed |
| `failed`, `abandoned` | exit 20 `draft_not_ready` | same | same | allowed | allowed |
| `unknown` | full create with the same key | new generation, full create | an override of the full create | exit 20 `draft_not_ready` | allowed, its prompt says a ticket may exist |
| `created` | exit 20 `already_created` | new generation, full create | sprint-only run | exit 20 `already_created` | allowed |
| `sprint_failed` | sprint-only run (the stored sprint) | new generation, full create | sprint-only run | exit 20 `already_created` | allowed |

Stored fields: `draft_id`, `status`, `origin`, `created_at`, `updated_at`, `site`, `project`, `repo` (nullable), `type`, `title`, `body_md`, `parent`, `assignee`, `ticket_labels`, `fields`, `notes`, `overrides` (the closed list `title`, `body`, `type`, `project`, `site`, `parent`, `assignee`, `labels`, `sprint`, `fields` of what the human changed), `agent_id`, `run_token` of the last run (never printed), `artifact` (`sha256`, `size`), `failure` (`code`, `outcome`, `detail`), `generation`, `created_key`, `created_url`, `sprint` (`selector`, `id`, `name`, `board`, `step`) and the `attempts` list. Text fields hold your own draft text; the database is private (mode 0600 in a 0700 directory).

## Draft order

`draft` performs these steps in order; a failure before step 8 leaves the draft in `failed` (or no draft when step 2 has not run). `draft --continue` follows the Continue order.

1. **Prepare.** Load config. Site: `--site`, else `ticketctl.site`. Project: `--project`; else, when the site is known and `projects` names exactly one `<site>/<KEY>` for it, that key; else, when the site is unknown and `projects` names exactly one entry, that entry (and its site); else exit 2 `missing_option` naming `--project`. A site still unknown after the project is known is resolved with `ticketctl sites resolve <PROJECT>-1 --json` (ticketctl's own offline selection by project; the key's number is a placeholder), and `ticketctl sites --json` with exactly one site selects it; several candidates exit 21 `ambiguous` with the site names. Type: `--type`, else `projects.<site>/<KEY>.default_type`, else none. Repo: `--repo`, then `projects.<site>/<KEY>.repo`, then the cwd's `origin` remote, then none. Check the siblings against the pinned minimums (exit 12 `tool_missing` or `tool_too_old`, naming the install command), run the skill preflight, and refuse a headless draft with no seed (exit 2 `missing_seed`). An interactive draft with no terminal on stdin and stdout exits 2 `no_terminal`.

   **Skill preflight.** For `plugin:skill` look for `skills/<skill>/SKILL.md` in each plugin directory passed (the bundled one and `plugin_dirs`; the plugin name is the manifest's name, not the directory's) and exit 12 `skill_not_found` naming the directories searched. A bare skill name cannot be checked, so a warning is printed and the launch goes ahead; a user or project skill also needs `setting_sources` to include its source. The check applies to `--skill NAME`. Template and plugin files must have a base name of 1 to 128 characters of `[A-Za-z0-9._-]`. `validate` runs the same check.
2. **Record.** Generate `draft_id` and `run_token`, take the per-draft lease and write the `drafting` row. The `pre-draft` hook runs here; a veto (exit 11) or escalation (exit 13) of a new draft deletes its row again.
3. **Prefetch** (see Prefetch). Failures here are warnings (the session simply gets less context) except an unreachable ticketctl with no way to know the project, which is exit 12.
4. **Prompt file.** Write the prompt file (see Prompt file) with mode 0600 under the state directory's `prompts/`, created `O_CREAT|O_EXCL|O_NOFOLLOW` with a random name, and the template file (if any) as a trusted `--input-file`.
5. **Kind check.** Run `agent-spawn launch --dry-run --json` with the final flags and no `--result-file`, and require exit 0, `agent_settings.kind == "draft-ticket"`, `reads_third_party == false`, `would_refuse == null`, an empty top-level `warnings` and an empty `session_plan.warnings` and `session_plan.profile.fallback` not true (a misspelled kind exits 0 with only those signals).
6. **Launch.** Snapshot the terminal (termios) and run `agent-spawn launch` (see Session launch) with a fresh `--result-file` path under `results/`. Restore the terminal in a `finally` (see Prompt and terminal rules).
7. **Outcome.** Read the result (see Session launch for the headless reading). The result is unusable when the file is missing, is not JSON, has `dry_run` or has `run: null`; then find the agent by its labels and read `agent-spawn show <agent_id> --json` instead. Map the outcome with the Session outcomes table. On `done`, read the frozen artifact with the Artifact reader and validate it against the Slot contract.
8. **Store.** Copy the validated artifact into the draft, set `ready` (T5) and tear the agent down with `agent-spawn teardown <agent_id>`. On any other outcome apply the table (T6; the agent stays parked). Delete the prompt file and the result file.
9. **Finish.** Run the post-draft duplicate check (see Prefetch), the `post-draft` hook and the audit record, print the result and, on a terminal and without `--no-review`, offer the review.

## Continue order

`draft --continue DRAFT_ID [--mode M]` performs the Draft order with these changes.

1. **Prepare.** Load the stored draft (exit 10 if missing) and apply the Gate table (exit 20 `already_created` or `draft_not_ready`). The fixed-field flags and `--idea` are refused (exit 2 `invalid_combination`). A `pre-draft` veto or escalation changes nothing (exit 11 or 13).
2. **Pick the way.** Look the agent up with `agent-spawn list --label draft=<draft_id> --label owner=ticket-create --json`. The agent is resumable when exactly one such agent is not torn down, is `parked`, and `agent-spawn show` lists a run with `first_working_at` set (a session in which nothing was ever sent cannot be resumed). If it is resumable, resume it. Otherwise, if the draft stores an artifact, tear down any parked agent of the draft and launch a new session whose prompt carries the stored title, body, fields and notes as a fenced untrusted block `previous`. Otherwise exit 20 `draft_not_resumable` (nothing to continue from; discard the draft and run `draft` again).
3. **Record.** Generate a new `run_token`, take the lease, set `drafting` (T3), and write a new prompt file with the trusted header (`draft_id`, the new `run_token`, and for a resume a sentence that this is a continuation: write a new `draft.json` that echoes this `run_token` and report `done`). A resumed conversation holds the old token in its history; the artifact must echo the token of the newest prompt.
4. **Kind check and call.** Resume: run `agent-spawn resume AGENT_ID --dry-run --json` with the flags below (without `--result-file`) and require exit 0, `would_refuse` null and empty `warnings`, then run the same call with `--result-file`. New session: the Draft order's kind check and launch.
5. **Outcome and store.** As Draft order steps 7 to 9, with T5 or T6.

Resume flags (all passed every time so safety does not depend on the carried values): `--mode interactive --placement here`, or `--mode headless --no-implicit-done --wait` for a headless continuation; `--prompt-file`; `--result-file`; `--json`; `--permission-profile`; `--sandbox`; `--setting-sources`; `--env-allowlist-only` (the same names as launch); one `--plugin-dir` per directory; `--no-promote`; `--actor`; `--decision-id` as given; `--model` and `--timeout` when configured or given. It passes no `--repo`, worktree flag, `--kind`, `--skill`, `--label` (the agent keeps its labels) or `--ephemeral`. The `origin` of the draft after the run is the mode of this run.

## Create order

`create` is human-only as a whole. `--actor auto:...` exits 2 `human_only` (except with `--dry-run`), the actor passed to ticketctl on every call is `human:<user>`, and the audit records the same actor.

1. **Gate.** Without a human present (`/dev/tty` does not open) exit 13 `needs_human` before anything else; `--dry-run` skips this check. Take the per-draft lease (exit 12 `unavailable`, `reason: locked`; `--dry-run` takes none). Apply the Gate table. A sprint-only run goes to the Sprint step.
2. **Overrides.** Apply the flags to a copy of the draft in memory. Check the freeze rule. Resolve `--sprint` (see Sprint step). Nothing is saved yet.
3. **Plan.** Create a fresh work directory under `tmp/` (mode 0700, random name) and write the title, the body and every field value into it, one file each, mode 0600, `O_EXCL`, and remember the sha256 of each file. Run `ticketctl create --dry-run --json --actor human:<user> --site=S --project=P --type=T --title-file=F --body-file=B` with `--parent=`, `--assign=`, `--label=`, `--set=NAME=@FILE` and `--dedupe-key=KEY`, each only when set and each model-derived value only when it matches its closed grammar. Derive the screen state from the answer (Dedupe key and attempts). Everything the screen shows comes from the plan (`changes[]` and `steps`) where the plan carries it, and from the draft otherwise.
4. **Duplicates.** Run the near-duplicate search on the final title on the first round and again only when the title changed; hits are warnings.
5. **Review.** Save the overrides given by this call (a human is present), show the screen, read one answer. `e` edits (Editor), saves the new text to the draft and returns to step 3 with new files. A `blocked` state offers only `e`. A `ready` `y` goes to step 6; any other answer ends as the state table says.
6. **Write.** Check that the sha256 of every file in the work directory still equals the remembered value (a difference exits 1 with `reason: input_changed` and sends nothing). In one transaction insert the attempt (outcome `unknown`), set the generation and set the status `unknown`. Write the audit `attempted` record. Run `ticketctl create --yes --expect-plan HASH --dedupe-key KEY --actor human:<user>` with the same arguments and files as step 3, then apply the outcome rule and the Lower tool exits table. `stale_ticket` returns to step 3 (the human answers again). `escalated` runs the escalation question. The audit outcome record is written when the call ends.
7. **Sprint.** If a sprint was chosen and the ticket exists, run the Sprint step; a failure sets `sprint_failed`, keeps the key and exits 12 with the key in the result.
8. **Finish.** Write the audit outcome records, run `post-create`, print the key and URL, delete the work directory and release the lease.

**What the plan hash covers.** ticketctl's hash covers the action, site, project, assign target, field names with value hashes, body hashes, labels, sprint id and dedupe key. It does not state that it covers the title, the type or the parent. ticket-create does not rely on it for those: it renders them from the dry-run plan and it checks the file digests of step 6.

**Escalation.** When ticketctl exits 13 `escalated` after the human's yes, the attempt is `failed` (nothing was written). Show the hook names and the sanitized reason (as a model-free note: ticketctl's hook text is untrusted), keep the draft and ask a second question on the same prompt rules: `A hook asked for review (NAMES). Override and create? [y/N]`. On yes run the same call again with `--override-escalation=ticketctl` and one `--decision-id` per id ticketctl returned, as a new attempt with the same key; the audit records `override_escalation: ticketctl`. Any other answer ends with exit 13 `escalated` and the draft unchanged. The same question covers `pre-sprint`.

**Edits that are not a dry-run problem.** A body that fails ticketctl's validation is `blocked`; `e` edits it and the loop returns to step 3. Fields that fail are fixed with `--set NAME=VALUE` (or `--set NAME=` to drop a model-written field from the draft) on the next `create`.

## Prefetch

ticket-create gathers context deterministically before the session, with the same environment and credentials as its own process; the session itself never calls ticketctl. All ticketctl calls carry `--site` and `--json`, take their text from files, and have a total time budget of 60 seconds (a timeout is a warning, not a failure).

| Item | When | Call | Used for |
|---|---|---|---|
| Create form facts | always | `ticketctl create --dry-run --json --site S --project P [--type T]` with no values | allowed issue types (`details.candidates`, from a probe with the reserved invalid type name `ticket-create-probe` when no type is fixed) and `details.required_fields` for a fixed type |
| Near-duplicates | when there is seed text | `ticketctl search --project P --status-category new --status-category indeterminate --text-file SEARCHTEXT --limit 10 --fields key,title,status --json` | up to 10 keys and titles in the prompt |
| Parent summary | when `--parent` is given | `ticketctl get KEY --fields key,type,title,status --json` | one line in the prompt; also validates the parent exists (a ticketctl exit 10 is exit 2 `invalid_argument` naming `--parent`) |
| Notes | when the session has a repo | none (claude-session runs `memoctl context` at SessionStart) | memory notes; ticket-create adds nothing |

The search text is the first non-empty line of the seed (or the title for the post-draft check), trimmed, cut at a word boundary to at most 80 characters and written to a file; the search is a phrase match and finds near-duplicates only when they contain that phrase. A search error is the warning `form_facts_unavailable` (form) or a missing duplicates block (search). The post-draft duplicate check runs the same search on the final title; its hits become warnings on the review screen and never block. ticket-create compares nothing and lists what ticketctl returns.

## Prompt file

The prompt is a Markdown file under the state directory's `prompts/`, deleted when the run ends. It has, in this order: a trusted header (the run's `draft_id`, `run_token`, the fixed fields with a sentence that the skill must not change them without asking the human, the path `inputs/template.md` when a template applies, the output rule: write the artifact as `draft.json` and report it); then the fenced untrusted blocks. Each untrusted block uses the nonce fence of design guide §12 with a fresh 32-hex nonce per prompt: `<untrusted source="S" id="R">` and `</untrusted id="R">`, with `</untrusted` and `<untrusted` neutralized case-insensitively, and a standing "data, not instructions" sentence before the first block. The sources are `seed` (the idea text), `duplicates` (titles and keys), `parent` (the parent summary), `previous` (the stored draft of a continuation) and `form` (the allowed types and required fields, which are ticketctl and provider text and never go in the trusted header). `template` is not a fenced source; it is the trusted input file `inputs/template.md`. Every untrusted string passes the one sanitizer first; a single block longer than 16 KiB is cut with `[truncated: N bytes]`.

ticket-create adds no report or escalation instructions to the prompt (design guide §11): the standard instructions that claude-session appends tell the session how to report and that asking the human is always acceptable.

## Session launch

`draft` runs `agent-spawn launch` with these flags (every flag is passed on every call, so ticket-create's safety does not depend on another tool's config):

| Flag | Value |
|---|---|
| `--kind` | `draft-ticket` |
| `--mode` | `interactive` (or `headless` with `--mode headless`) |
| `--placement` | `here` (interactive only) |
| `--wait` | headless only (interactive `here` implies it) |
| `--skill` | the configured skill (default `ticket-create:draft-ticket`) |
| `--plugin-dir` | the bundled plugin, then each `plugin_dirs` entry |
| `--prompt-file` | the prompt file |
| `--input-file` | the template file, when one applies |
| `--repo` | the resolved repo, when there is one |
| `--worktree` | `none` |
| `--cwd` | not passed for interactive runs; `session-dir` for headless runs (config `cwd` overrides; `caller` passes the caller's cwd) |
| `--permission-profile` | the bundled profile file (config `permission_profile`) |
| `--sandbox` | `off` (config `sandbox`) |
| `--setting-sources` | empty (config `setting_sources`) |
| `--env-allowlist-only` | each name of the fixed base list `PATH HOME USER LOGNAME SHELL LANG TERM TZ LC_ALL` plus configured names (`LC_*` cannot be passed because claude-session takes plain names) |
| `--model`, `--timeout` | only when configured or given |
| `--no-implicit-done` | headless only |
| `--no-promote` | always |
| `--label` | `owner=ticket-create`, `draft=<draft_id>`, `repo=<id>` when there is a repo |
| `--actor` | `human:<user>` interactive, `auto:ticket-create` headless |
| `--result-file` | a fresh path under `results/` (none on the kind check) |
| `--json` | always |

ticket-create never passes `--ephemeral`, `--wait-timeout` (a headless launch is bounded by `--timeout` or the kind's headless limit, so exit 22 cannot occur), `--yes`, `--force` or `--override-escalation` to agent-spawn. The bundled permission profile sets `defaultMode` to `default`, allows `claude-session report` and writes under `$CSESSION_DIR/artifacts`, and denies `ticketctl`, `agent-spawn`, `ticket-create`, `memoctl add`, `memoctl delete`, `memoctl cron`, `mcp__*`, network tools, writes to every awkit config and state directory, and, as defense in depth, Edit and Write to the user's shell startup files and to ticket-create's config and state. The profile is guidance for a cooperative session, not a boundary; a user who wants connectors in draft sessions replaces the profile in config. The bundled skill is a single `SKILL.md` with no other files (a second file the skill must read prompts the human for permission).

A headless launch passes `--wait` (and no `--wait-timeout`: `--timeout` or the kind's headless limit bounds the run, and exit 22 cannot occur). `agent-spawn launch ... --mode headless --no-implicit-done --wait --result-file R --json` exits with the run's outcome code. Read R (a result object with `run.outcome` and `run.report`; a missing file, unreadable JSON or a file with `dry_run` or `run: null` is unusable), and if it is unusable find the agent with `agent-spawn list --label draft=<draft_id> --label owner=ticket-create --json` and read the last run of `agent-spawn show AGENT_ID --json`. The kind check (`launch --dry-run`) passes no `--result-file`.

## Session outcomes

| Outcome (exit) | Draft status | Agent | ticket-create exit | Message |
|---|---|---|---|---|
| done (0), report `done`, `report.implicit` false, valid artifact | `ready` | torn down | 0 | the draft id and next step |
| done (0), artifact missing or invalid | `failed` (`failure.code: artifact_missing` or `artifact_invalid`) | parked | 22 | names the cause (wrong skill name, malformed file) and `draft --continue` |
| done (0) with `report.implicit` true (headless without a report) | `failed` (`implicit_done`) | parked | 22 | the session did not report |
| exited-unreported (14), interactive, `first_working_at` set | `abandoned` | parked | 14 | none beyond one line: no draft was made, `--continue` resumes |
| exited-unreported (14), interactive, `first_working_at` null | `abandoned` | parked | 14 | the session never started working; names the skill and suggests checking it |
| exited-unreported (14), headless | `failed` (`no_report`) | parked | 14 | the session did not report; `--continue` |
| blocked or needs-input (13) | `failed` (`needs_human`) | parked | 13 | the report reason, sanitized |
| killed (15), timeout (16), lost (17), failed (18) | `failed` (`outcome`) | parked | the same code | the end reason |
| vetoed (11) or not started (12) | `failed` (`vetoed` or `not_started`) | parked when the launch failed after the agent record was made, none after a `pre-launch` veto | the same code | the lower tool's reason |

For the last row read `agent_id` from the error envelope and keep it. agent-spawn exit 10 is mapped to exit 12 `lower_failed` with `lower_error`. The outcome names are agent-spawn's. A parked agent keeps its attention flag in agent-spawn until the draft is discarded or pruned; this is intended, ticket-create never acknowledges it. A run that ends in any row other than `done` after a stored artifact exists leaves the draft `ready` with `failure` set (T6).

In an interactive run a `needs-input` report does not end the session; the human keeps typing and a later `done` report supersedes it. Only a session that ends while still in `needs-input` produces exit 13.

## Slot contract

The `draft-ticket` slot (architecture §5.5) is filled by a skill that receives the prompt file and optionally an input file, and writes one JSON artifact. The skill's name is the config key `skill` (any name Claude Code accepts). A replacement skill must follow this contract.

Inputs: the prompt file (the first line the session sees gives the session directory) and, when a template applies, the template as `inputs/template.md` under the session directory. The template is looked up by the fixed type (flag or default type), else by `default`.

Output: the skill writes `draft.json` directly inside `$CSESSION_DIR/artifacts/` and reports it as the artifact of its `done` report. The artifact is one JSON object with exactly these keys (unknown keys are refused):

| Key | Type | Rule |
|---|---|---|
| `schema_version` | integer | `1` |
| `draft_id` | string | equal to the prompt's `draft_id` |
| `run_token` | string | equal to the newest prompt's `run_token` |
| `type` | string | `^[A-Za-z0-9][A-Za-z0-9 _.-]{0,63}$` |
| `title` | string | 1 to `limits.title_chars` characters after trimming; one line (no line breaks) |
| `body_md` | string | Markdown restricted to what ticketctl converts (no tables, images or raw HTML); at most 29000 characters |
| `parent` | string or null | `^[A-Z][A-Z0-9_]{0,31}-[0-9]{1,12}$` |
| `labels` | array of string | at most 20; each `^[A-Za-z0-9][A-Za-z0-9._:-]{0,63}$` |
| `assignee` | string or null | `me`, `none` or `^[A-Za-z0-9][A-Za-z0-9:_.-]{0,127}$` |
| `fields` | object | at most 20 entries; name `^[A-Za-z][A-Za-z0-9 _.-]{0,99}$` and not (case-insensitively) `title`, `summary`, `description`, `labels`, `assignee`, `parent`, `sprint`, `resolution` or `reporter` (`artifact_invalid`, detail `field`); value a string of at most 8192 characters, a finite number whose decimal form matches `^-?[0-9]+(\.[0-9]+)?$` (booleans, exponent forms and non-finite values are `artifact_invalid`), or an array of at most 50 strings |
| `notes` | string | at most 4096 characters; shown to the human |

`parent`, `labels`, `assignee`, `fields` and `notes` may be omitted. An assignee that is an account id (not `me` or `none`) gets the warning `assignee_account` on the review screen. Every model-derived value is passed in the attached form: `--type=T`, `--parent=KEY`, `--assign=VALUE`, `--label=L`, `--set=NAME=@FILE`; no model-derived value begins with `-`. Title, body and field values only ever travel in files. Field files hold a string verbatim, a number as its canonical decimal and an array as a JSON array of strings.

A skill's `done` report must attach this artifact. A missing, wrong or unreadable artifact is exit 22; the draft's text is never trusted beyond these checks, and the human reviews every field before anything is sent.

## Artifact reader

One reader opens every untrusted file (the frozen artifact, the template, a `--idea-file`, the editor's file, a `--set NAME=@FILE` file). It opens with `O_NOFOLLOW|O_NONBLOCK`, requires a regular file by `fstat`, checks the size against the cap before reading (`limits.artifact_bytes` for an artifact, 1 MiB for an idea file, 256 KiB for a template, `4 * 29257 + 1` bytes for the editor file), decodes strict UTF-8 without a byte order mark and, for JSON, rejects duplicate keys, `NaN` and `Infinity`, lone surrogates and a nesting depth over 8. Numbers are parsed with an integer or decimal rule that rejects exponents, non-finite values and more than 400 digits. In title, field names and field values it rejects every code point the sanitizer would replace (body and notes keep newline and tab). For the frozen artifact it also checks that the file's `size` and `sha256` equal `run.report.artifact`, that `report.status` is `done` and `report.implicit` is false. Any failure is a validation failure with a `code` (`artifact_missing`, `artifact_invalid` with a `detail`: `frozen_file`, `json`, `schema`, `draft_id`, `run_token`, `field`), never an internal error; the validator catches every exception.

## Review screen

`create` shows the screen on `/dev/tty` (and stderr when that is the terminal). Its height is read from `TIOCGWINSZ`; when it is unknown the screen assumes 24 rows and 80 columns.

1. **Model text, in a gutter.** Every line begins with a box-drawing vertical bar and a space, and folds at the terminal width in display cells (wide characters count 2), with the gutter on every fold row. In this order: the body as ticketctl will send it (the `description` of the dry-run plan's `changes[]`, which includes ticketctl's attribution paragraph); the draft's `notes` under the label `notes`; each field under `field NAME` with its value; the near-duplicate hits (`KEY title`) under `possible duplicates`. One shared budget applies: at most 200 lines and 20000 characters, and at most the terminal rows minus the facts block and the prompt. Every item that is cut ends with `[... N more characters; sha256 HEX]`. A cut body can be read in full with `e`, and any field with `show DRAFT_ID --field NAME`.
2. **The facts block, last, directly above the prompt.** Single-line items only, drawn from closed grammars or quoted: site, project, repo, type, parent, assignee, the ticket labels (each quoted), the field names (a count and the names), the sprint (quoted name and id), the `overridden` marks, the draft's origin, the warning codes (with counts) and the plan hash, and the title, quoted, in full (at most 255 characters, folded). Every model-derived value is quoted and has its line breaks and control characters replaced by markers; a value longer than 120 characters is cut with the same marker. Values come from the dry-run plan where it carries them (title, description, assignee, labels, fields, project, type) and a difference between a value the human asked for and the plan's value is the warning `plan_differs`.
3. The prompt line (Dedupe key and attempts, screen state).

If the terminal has fewer rows than the facts block plus the prompt plus 4, the screen prints `terminal too small: N rows needed` and offers only cancel and `e`. Nothing in the gutter zone can appear below the facts block, and no pager is used.

## Prompt and terminal rules

- **Presence.** A human is present exactly when `os.open("/dev/tty", O_RDWR | O_NOCTTY)` succeeds. The state of stdin, stdout and stderr is never tested for this (spike evidence: it is wrong in both directions).
- **Prompt.** The confirmation reads one line from `/dev/tty` after forcing canonical mode, echo and signal generation, and flushing pending input with `tcflush(fd, TCIFLUSH)` before and again after the read. Only `y` or `yes` (trimmed, case-insensitive) accepts. EOF, an over-long line (over 64 characters), pending extra input, a partial line and a bracketed paste decline; Ctrl-C ends the call with exit 130 and changes nothing. The same prompt serves every yes-or-no question in the tool.
- **Restore.** Before launching the session, the editor or any child that uses the terminal, ticket-create takes a termios snapshot of `/dev/tty`. In a `finally` it restores it with `tcsetattr(TCSAFLUSH, snapshot)` and writes the reset sequence that turns off bracketed paste, mouse tracking and the alternate screen and shows the cursor. This happens on every exit, including a signal-killed child.
- **Cooked output.** Everything untrusted that is printed passes the Output sanitizing rule first.

## Editor

`e` writes `title`, a blank line and `body_md` to a file named at random with mode 0600 created `O_EXCL` inside a mode 0700 directory under the state directory's `tmp/`, and runs the editor command from config `editor` (default `$VISUAL`, then `$EDITOR`, then `vi`) with the file path as its last argument (the command string is split with `shlex`, never run through a shell; it comes from the user's own config or environment, never from a draft). The editor inherits the terminal and the tool's environment minus the names in `credential_env`; the snapshot and restore rules apply. After it exits, the file is re-read with the Artifact reader (a symlink or FIFO swap, a size over the cap, an empty result, a non-zero exit or a death by signal aborts the edit and keeps the draft unchanged), the first line becomes the title (at most `limits.title_chars`, no control characters) and the rest the body. The new text is saved to the draft, listed in `overrides` and sent through a new dry-run.

## Dedupe key and attempts

A draft has an integer `generation` (0 at first). The dedupe key of generation g is `ticket-create:<draft_id>` for g = 0 and `ticket-create:<draft_id>:<g>` for g > 0; it matches ticketctl's key grammar and never contains title or type text. `draft_id` comes from ticket-create's own record, never from the artifact. `create --again` is the only way to raise the generation: it is accepted for `created`, `sprint_failed` and `unknown` drafts and means "I checked the ticket system; make another ticket".

ticketctl labels the ticket it creates with a value derived from the site, the project and the key, and looks it up inside the project, and it binds a key to the issue type and the exact title. Therefore, once an attempt of the current generation has outcome `unknown`, `created` or `reused`, the draft's site and project are frozen: `create --site` or `create --project` with a different value exits 20 `attempt_recorded` and changes nothing. Type and title edits are allowed (ticketctl answers `reused`, or `already_exists` for a changed type or title, which selects the `already_created` state below). `--again` starts a new generation and lifts the freeze for it.

An attempt is one `ticketctl create --yes` call: `n` (1, 2, ...), `generation`, `key`, `site`, `project`, `plan_sha256`, `body_sha256` (sha256 of the draft's `body_md`), `started_at`, `outcome` (`unknown`, `created`, `reused`, `failed`), `attempt_id` (ticket-create's own, used to pair audit records), `ticketctl_attempt_id`, `ticket_key`. Outcome rule for the answer of a `ticketctl create --yes` call: exit 0 gives `created`, or `reused` when ticketctl says `reused`; exit 12 `outcome_unknown`, exit 1, a death by signal and an unreadable result keep `unknown`; every other answer is definite and gives `failed`. The sprint step is not an attempt; it is recorded in the draft's `sprint` (`selector`, `id`, `name`, `board`, `step`: `none`, `done` or `failed`).

### Screen state

Every `create` round runs `ticketctl create --dry-run` with the key of the current generation and derives one state from the answer. Attempt history never selects a state; it only adds warnings.

| ticketctl answer | State |
|---|---|
| exit 0, plan with steps | `ready` |
| exit 0, plan with empty `steps` and a `dedupe_match` | `already_created` (key not yet known) |
| exit 20 `already_exists` | `already_created` (key is `details.existing_key`) |
| exit 22 or 10 (`missing_required_fields`, `invalid_value`, `body_too_large`, `empty_body`, `node_not_allowed`, `secret_in_body`, `unknown_field`, parent or board not found, ...) | `blocked` with ticketctl's reason |
| exit 21 | not a screen: exit 21 with the candidates and the flag to pass |
| any other exit | not a screen: the Lower tool exits table |

Warnings added by history: `previous_attempt_unknown` when the state is `ready` and an attempt with this key has outcome `unknown` (ticketctl found no ticket, so it will create one with the same key; it also guards the label index lag on its side); `body_differs` when the state is `already_created` and the stored attempt's `body_sha256` differs from the current body (a body edit is not applied to an existing ticket).

| State | Prompt line | Accepted answers | Any other answer | Exit |
|---|---|---|---|---|
| `ready` | `Create this ticket? [y/N]` | `y`, `yes` write; `e` edit | decline | 20 `declined` |
| `already_created` | `A ticket for this draft already exists (KEY). Type accept to record it, anything else cancels:` | `accept` (`y` is not accepted) | cancel | 20 `declined` |
| `blocked` | `Cannot create: REASON. e edits, anything else stops:` | `e` | stop | 22 with ticketctl's reason |

Ctrl-C at any prompt ends the call with exit 130 and changes nothing. `accept` records the draft as `created`: with a known key it reads `ticketctl get KEY --site S --fields key,url --json`; with only a `dedupe_match` it runs the same `ticketctl create --yes --expect-plan HASH --dedupe-key KEY` call as the write, which ticketctl answers `reused` with the key and url without prompting, hooks or audit (a changed ticket-system state makes the hash differ and ends in `stale_ticket`, never a write). It records an attempt `reused`, writes one audit record with `result.outcome: accepted`, and runs `post-create` with result `reused`.

## Sprint step

`--sprint open|next|ID` (id grammar `^[0-9]{1,18}$`) and `--board ID|NAME` are accepted on `create`; the selector, the resolved id and name, and the board are stored in the draft's `sprint`. Resolution uses ticketctl's rules, with `--site` and `--project` (and the stored board): `open` is the single `active` sprint of `ticketctl sprints list --state active` (none: exit 20 `no_sprint`; several: exit 21 with the candidates); `next` is the `future` sprint with the earliest non-null `start`, else the lowest id (none: exit 20 `no_sprint`); an id is read with `ticketctl sprints show ID` and a `closed` sprint exits 20 `sprint_closed`. `--board` takes an id or a name.

**After a create.** Once the ticket exists the step runs `ticketctl sprint add KEY --sprint ID --site S [--board B] --actor human:<user> --dry-run`, then with `--yes --expect-plan HASH2`; the human's yes on the create screen covers it because the sprint was on the screen. `unchanged` is success. `stale_ticket` is re-planned once. Any other answer (veto, escalation unanswered, rejected, closed sprint, unknown outcome) sets `sprint_failed`, keeps `created_key`, and exits 12 with the key and ticketctl's reason in `details`; an escalation asks the escalation question first.

**Sprint-only run.** `create DRAFT_ID --sprint S` on a `created` draft, or `create DRAFT_ID` on a `sprint_failed` draft (the stored sprint is used unless `--sprint` is given), runs only this step and needs a human. It resolves the sprint, runs the `sprint add` dry-run, and shows a short screen: the ticket key and URL, the sprint name and id, the board, ticketctl's warnings (for example `sprint_moved`) and the plan hash, with the prompt `Add KEY to sprint NAME? [y/N]`. Only `y` or `yes` runs `--yes --expect-plan`. Declining leaves the status unchanged (exit 20 `declined`). The sprint step is recorded in `sprint.step`, never as an attempt.

## Hooks

Hooks are configured in the tool's config (design guide §8). Each receives one JSON document on stdin and the environment `TICKET_CREATE_HOOK`, `TICKET_CREATE_ID` (the draft id) and `TICKET_CREATE_OBJECT_TYPE` (`draft`); they inherit the tool's environment minus the names in `credential_env`. Default timeout 30 seconds. `validate` warns when `credential_env` is empty.

Exit 13 escalates and any other non-zero exit vetoes (exit 11) for `pre-draft`; a timeout is a veto; `post-` hooks cannot undo. `async` is valid on `post-` hooks only. `event_id` is a fresh id (`ev-` plus 16 base32 characters) per hook invocation.

| Hook | Runs | Veto or escalation | Payload (beyond `tool`, `hook`, `object_type`, `object_id`) |
|---|---|---|---|
| `pre-draft` | before the session launches (Draft order step 2) | exit 11 on a veto, 13 on an escalation (a new draft's row is removed; a continued draft is left as it was) | `origin`, `site`, `project`, `type`, `mode`, `skill`, `has_seed`, `seed_bytes`, `continue` |
| `post-draft` | after every run that ends in `ready`, `failed` or `abandoned`, a failed `--continue` run included | none | `origin`, `status`, `failure`, `outcome`, `site`, `project`, `type`, `sizes` (`title`, `body`), `artifact_sha256`, `event_id` |
| `post-create` | after the create and sprint steps | none | `result` (`created`, `reused`, `partial`, `failed`, `unknown`), `site`, `project`, `key`, `sprint_step` (`none`, `done`, `failed`), `attempt` (`n`, `plan_sha256`; null for a sprint-only run), `event_id`, `dedupe_key` (the ticketctl key of the attempt, or null for a sprint-only run) |

Payloads carry `reads_third_party: false` and `tainted: false`, ids, enums, counts, sizes and hashes, and never the title, body, notes or any model text. There is no `repo` or `labels` key (the draft has a repo; it is not part of the payload), so a gate on these hooks is scoped `*`. There is no `pre-create`: ticketctl's own `pre-create` hook runs after the human's confirmation and before the provider write, and a gate belongs there.

## Exit codes

| Exit | `code` | `reason` | Caller action |
|---|---|---|---|
| 0 | none | none | done |
| 1 | `internal` | `input_changed` (a stored input file changed before the write; nothing was sent) or null | report |
| 2 | `usage` | `stdin_conflict`, `invalid_combination`, `missing_seed`, `human_only`, `invalid_argument`, `missing_option`, `no_terminal` | fix the invocation |
| 2 | `invalid_config` | `invalid_config` | fix the config (`details.config_key`) |
| 10 | `draft_not_found` | null | no such draft |
| 11 | `vetoed` | null | a `pre-draft` hook or a lower tool vetoed (`details.hook`) |
| 12 | `tool_missing`, `tool_too_old` | null | install or upgrade (`details.tool`, `minimum`, `found`, `install`) |
| 12 | `skill_not_found`, `not_started` | null | check the skill name or the session |
| 12 | `unavailable` | `locked` for a held lease, else null | retry later |
| 12 | `lower_failed`, `rejected`, `outcome_unknown`, `partial`, `schema_newer` | null | `outcome_unknown`: run `create` again (same key); `partial`: the ticket exists, only the sprint step failed (`details.key`, `step`); `lower_failed`: `details.tool`, `lower_error` |
| 13 | `needs_human` | null | `create` or `discard` without a human, or the session asked for one |
| 13 | `escalated` | null | a ticketctl hook asked for review and the human declined the override |
| 14 to 18 | `session_exited_unreported`, `session_killed`, `session_timeout`, `session_lost`, `session_failed` | null | the draft session's outcome |
| 20 | `refused` | `declined`, `draft_not_ready`, `draft_not_resumable`, `already_created`, `live`, `sprint_closed`, `no_sprint`, `already_exists`, `attempt_recorded` | by reason |
| 21 | `ambiguous_draft`, `ambiguous` | null | a prefix (`details.candidates`), or a site, board or sprint |
| 22 | `artifact_missing`, `artifact_invalid`, `invalid_ticket_input`, `missing_required_fields` | the same word | the draft's content is missing or invalid; `invalid_ticket_input` is ticketctl's own validation on a dry-run (`details.lower_error`) |
| 130, 143 | `usage` or none | null | 130: Ctrl-C at a prompt or a waiter (nothing changed); 143: SIGTERM before a session started |

`details` keys are closed: `candidates`, `required_fields`, `existing_key`, `plan`, `attempt`, `lower_error`, `hook`, `hook_stderr`, `key`, `step`, `tool`, `minimum`, `found`, `install`, `config_key`. Warning codes are closed: `duplicate_candidates`, `body_near_limit`, `body_differs`, `previous_attempt_unknown`, `plan_differs`, `fixed_field_changed`, `assignee_account`, `form_facts_unavailable`, `credential_env_empty`, `mcp_state_unknown` and `skill_unchecked` (both from `validate` and the preflight), plus ticketctl's plan warning codes passed through; `body_near_limit` is ticketctl's (above 90 percent of its limit). A `SIGTERM` after the session started exits 15 and leaves the agent parked. `SIGINT` while a session runs goes to the session's own terminal handling, as in agent-spawn.

## Lower tool exits

| Call | Exit | ticket-create does | Attempt and status |
|---|---|---|---|
| `ticketctl create --yes` | 0 | records the key, URL and `reused` | `created` or `reused`; status `created` |
| | 11 | shows the vetoing hook's reason (sanitized) and exits 11 | `failed`; `rest_status` |
| | 12 `outcome_unknown` | exits 12 with the attempt in `details` | stays `unknown` |
| | 12 other (`auth_required`, `bad_credentials`, `forbidden`, `rejected`, `unavailable`, `timeout`, `rate_limited`, `provider_changed`) | shows the code and message (sanitized), exits 12 `lower_failed` (`rejected` on a `labels` error says the site refuses the dedupe label and the ticket can only be made through ticketctl directly) | `failed`; `rest_status` |
| | 13 `escalated` | the escalation question | `failed` (a second attempt follows on yes) |
| | 13 `confirmation_required`, 2, 20 `declined` | cannot happen with `--yes` and a correct call: exit 1 `internal` | `unknown` for exit 1, `failed` for exit 2 |
| | 20 `stale_ticket` | re-plans and asks again | `failed` |
| | 20 `already_exists` | the `already_created` state | `failed` |
| | 21 | exits 21 with the candidates | `failed` |
| | 22, 10 | the `blocked` state | `failed` |
| | 1, signal, unreadable output | exits 1 | stays `unknown` |
| `ticketctl create --dry-run`, `search`, `get`, `sprints list/show`, `sites` | 0 | used | none |
| | 20 `already_exists` (dry-run) | the `already_created` state | none |
| | 22, 10 on `create --dry-run` | `blocked` | none |
| | 21 | exits 21 | none |
| | 12 on a prefetch read or on `search` | a warning, the prefetch continues; on `create --dry-run` exit 12 `lower_failed` | none |
| | 10 on `get` of the parent in `draft` | exit 2 `invalid_argument` naming `--parent` | none |
| `ticketctl sprint add` | 0 (`unchanged` too) | step done | `sprint.step: done` |
| | 11, 12, 13 (unanswered), 20 `sprint_closed`, 21 | step failed: `sprint_failed` in a full create (exit 12); in a sprint-only run the exit passes through | `sprint.step: failed` |
| | 20 `stale_ticket` | re-plan once, then as above | none |
| `agent-spawn launch`, `resume` | 0, 13 to 18 | the Session outcomes table | none |
| | 11, 12, 20, 21 | passed through with `lower_error`; exit 12 records `agent_id` from the envelope | none |
| | 10 | exit 12 `lower_failed` | none |
| | 22 | cannot happen (no `--wait-timeout`) | none |

Every lower tool's JSON is read tolerantly (unknown keys are ignored, a missing required key is a failure). Provider messages, hook stderr and `lower_error` text are untrusted and go through the sanitizer before printing.

## Output by exit code and error envelope

With `--json`, exit 0 prints the command's result object; an exit that adopted a run or built a plan prints that one object with the error as its `error` member; every other failure prints the envelope. The envelope is `{"error": {"code", "message", "reason", "retryable", "outcome_unknown", "draft_id", "details"}}`. `reason` is non-null for exits 2, 20 and 22. `details` carries only the closed keys listed under Exit codes; `message`, `lower_error`, `hook_stderr` and provider text are untrusted.

`draft` and `create` results carry the draft (id, status, origin, site, project, type, key and URL when created, the attempts) and, for a session, `agent_id`, `phase`, `outcome`, `end_reason` and `first_working_at`.

## Output sanitizing

Every untrusted string that leaves the process (human output, `--json` values, hook payload strings, logs) passes one sanitizer. It replaces C0 controls except newline and tab, DEL, C1 controls, bidirectional and isolate controls, zero-width and other format characters, the tag-character block, surrogates, private-use and unassigned code points, U+2028 and U+2029, non-space separators and variation selectors with `<U+XXXX>` markers, and also U+034F, U+115F, U+1160, U+16FE4, U+17B4, U+17B5, U+180B to U+180F, U+2800, U+3164, U+FFA0 and U+FFFC (so shown text equals the text ticketctl sends). It caps runs of combining marks at 8 and, for non-screen output (hook and log text), truncates lines over 200 characters with `<...>`. Terminal output of model text follows the Review screen's layout rule instead of the truncation. The sanitizer is applied once, at the point of output.

## Objects

Draft: the stored fields of Draft record and statuses (without `run_token`), as printed by `show --json`, with `generation` and `sprint` (`selector`, `id`, `name`, `board`, `step`). Attempt: `n`, `generation`, `key`, `site`, `project`, `plan_sha256`, `body_sha256`, `started_at`, `outcome`, `attempt_id`, `ticketctl_attempt_id`, `ticket_key`. Both are defined in `schemas/defs.json`.

## Configuration

Config is YAML at `~/.config/ticket-create/config.yaml` with drop-ins in `config.d/*.yaml` (design guide §5; `XDG_CONFIG_HOME` is respected). Precedence: flags, then `repos.<id>`, then environment (the keys listed under Environment variables only), then global config, then defaults. Durations: `timeout` is a number of seconds or a number with `ms`, `s` or `m`; `retention.*` also accepts `h` and `d`. `setting_sources` entries are `user`, `project` and `local`.

| Key | Default | Meaning |
|---|---|---|
| `me.name` | OS user | actor name for `human:<user>` |
| `ticketctl.config` | none | a ticketctl config file passed as `--config` |
| `ticketctl.site` | none | the default `--site` |
| `skill` | `ticket-create:draft-ticket` | the draft skill |
| `mode` | `interactive` | the default mode |
| `model`, `timeout` | none | passed to agent-spawn |
| `cwd` | none | `caller` or `session-dir`; null: the caller's directory (interactive), session-dir (headless) |
| `sandbox` | `off` | `--sandbox` |
| `setting_sources` | `[]` | `--setting-sources` |
| `permission_profile` | bundled file | `--permission-profile` |
| `env_allowlist` | `[]` | names added to the fixed base list |
| `credential_env` | `[]` | environment variable names removed from the environment of hooks and the editor (list the ticketctl token variable); `validate` warns while it is empty |
| `plugin_dirs` | `[]` | extra plugin directories |
| `templates` | none | map of type name or `default` to a template file; looked up by the fixed type (flag or default type), else by `default`; `repos.<id>.templates` entries replace same-key global entries |
| `projects.<site>/<KEY>.repo` | none | the repo id used when `--repo` is absent |
| `projects.<site>/<KEY>.default_type` | none | the type used when none is given |
| `duplicates.enabled`, `duplicates.limit` | `true`, `10` | the near-duplicate searches (limit 1 to 25) |
| `editor` | `$VISUAL`, `$EDITOR`, `vi` | the edit command |
| `retention.open`, `retention.done` | `30d`, `90d` | draft lifetimes from `updated_at`: `open` covers `drafting`, `ready`, `failed` and `abandoned`; `done` covers `created`, `sprint_failed` and `unknown` |
| `limits.artifact_bytes`, `limits.title_chars` | `262144`, `255` | reader caps (`artifact_bytes` 4096 to 1048576, `title_chars` 1 to 255) |
| `hooks` | none | the hooks listed under Hooks |

`projects` keys are `<site>/<KEY>` (a site name contains no `/`). `repos.<id>` may override `skill`, `templates`, `sandbox`, `setting_sources`, `env_allowlist`, `plugin_dirs`, `permission_profile`, `cwd` and `model`; it may not set `hooks`, `ticketctl.*`, `editor` or `credential_env`, because a repository-level setting must never select a program to run or a place to send credentials. A template path or plugin directory must be absolute and owned by the user. The template is delivered as `inputs/template.md` and the prompt header names that path.

## Environment variables

Config keys take `TICKET_CREATE_<KEY>` (nested keys joined with `__`) for `mode`, `model`, `timeout`, `duplicates.enabled`, `duplicates.limit` and `retention.*` only; other keys are not read from the environment because the environment is often set by repository tooling and a repository-level value must never select a program or a place to send credentials. The tool reads `VISUAL`, `EDITOR`, `TERM`, `PATH`, `HOME` and `XDG_CONFIG_HOME` and `XDG_STATE_HOME`. It passes no credential of its own: ticketctl reads its own token variable, which must be listed in claude-session's `secret_env` so a draft session never receives it, and in `credential_env` so hooks and the editor do not inherit it.

## Labels

Agents get `owner=ticket-create`, `draft=<draft_id>` and, when there is a repo, `repo=<id>`. These are the keys ticket-create uses to find its agents again (orphan recovery, `--continue`); `agent-spawn list --label owner=ticket-create --json` is the lookup. Ticket labels (Jira labels) are separate: `--ticket-label` and the artifact's `labels`. They are never agent labels and never contain whitespace.

## Files

All under `~/.local/state/ticket-create/` (`XDG_STATE_HOME` respected), directories mode 0700, files 0600: `state.db` (private SQLite, WAL), `log.jsonl` and `audit.jsonl` (public contracts), `prompts/`, `results/`, `tmp/` (random names, deleted when the call ends, stale files pruned after a day). The state directory, `state.db`, `log.jsonl` and `audit.jsonl` are opened `O_NOFOLLOW` and checked to be owned by the caller.

## Audit and log

`audit.jsonl` has one `attempted` record just before each `ticketctl create --yes` and `sprint add --yes` call and an outcome record after it, paired by `attempt_id` (ticket-create's own), plus one record per `draft` run. Fields: `ts`, `actor`, `decision_id` (list), `phase` (`attempted`, `outcome`), `attempt_id`, `action` (`create`, `sprint_add`, `draft`), `target` (`<site>/<project>` or the draft id), `inputs` (`draft_id`, `dedupe_key`, `plan_sha256`, `body_sha256`, `origin`, `overrides` (the closed list of Draft record and statuses), `override_count`, `override_escalation` (null or `ticketctl`), `agent_id`), `result` (`key`, `reused`, `ticketctl_attempt_id`, `outcome`, `code`), `seq`. `result.outcome` is one of `attempted`, `created`, `reused`, `accepted`, `failed`, `unknown`, `done` (sprint), `ready`, `abandoned`. `log.jsonl` records each command with counts and hashes; its events are `command`, `hook_result`, `lower_call`, `session`, `draft_state`, `lease` and `warning`, each written when its thing happens (a command ends, a hook returns, a lower tool call ends, a session outcome is read, a status changes, a lease is acquired, released, taken over, lost or stalled, a warning is raised). Neither file holds the title, body, notes or any model text, and neither holds secrets.

## Concurrency

A per-draft lease (holder PID plus process start time plus expiry) is taken by `draft` (new and `--continue`), `create`, `discard` and `prune`. Its time to live is 60 seconds and a background thread of the holder renews it every 20 seconds, including while the process waits for a session or a prompt. A lease is taken over when its holder (PID plus start time) is dead or its expiry passed by more than one time to live; a failed renewal is logged as `stalled`. A contended lease is exit 12 `unavailable` (`locked`), except that `prune` skips the draft. Two `draft` runs for different drafts do not interact. `list` and `show` take no lease.

## Dependencies

| Tool | Binding | Minimum | Used for |
|---|---|---|---|
| ticketctl | A | `0.1.0` (its first release, which has `create`, `sprint add`, `sprints`, `search --text-file`) | the only writer: `create`, `sprint add`, and reads (`search`, `get`, `sprints`, `sites`, `validate`) |
| agent-spawn | A | 0.34.0 | `launch`, `show`, `list`, `teardown`, `resume` |
| claude-session | none (through agent-spawn) | 0.45.0, the version agent-spawn requires | its `secret_env` must name ticketctl's token variable; `notes_context` runs `memoctl context` |
| wtctl | none (agent-spawn requires it even with no worktree) | the version agent-spawn requires | installed for agent-spawn |

A missing sibling exits 12 `tool_missing` naming its install command; one below its pinned minimum exits 12 `tool_too_old`.

## Upgrades

The state database has a version and only additive migrations; a database newer than the tool exits 12 `schema_newer`. Config keys are added, never repurposed.

## Residual risks

- The draft session is a trusted, unsandboxed run (the `draft-ticket` kind is not third-party-reading). Text it is shown (seed, duplicate titles, a parent summary) is fenced but can steer its output and its tool use; its permission profile denies the dangerous tools and the human watches it, and a replaced profile or sandbox setting is the user's own choice. Text from a third party given as a headless seed can steer the session without a human watching.
- The `draft-ticket` kind inherits MCP servers and claude-session has no per-call switch, so a connector that writes tickets bypasses the screen unless the kind is configured `mcp: none` or the profile denies `mcp__*` (the bundled profile does).
- A shell profile that exports the ticketctl token gives every shell command of the session the token; keep the token in a file sourced only by ticketctl.
- A hijacked session with one approved write can plant a shim on `PATH` or edit a template or plugin file.
- Nothing stops a process that can write to the terminal's input queue after the prompt is shown; the flush and the fixed layout cover input that arrived earlier.
- A process with the same user id can read `state.db`, the prompts and the drafts, and can edit the config.
- ticketctl's confirmation is not a boundary against a caller that can allocate a pty; ticket-create's boundary is that the draft session is not given the credential and is told not to run ticketctl.
- claude-session keeps a session's frozen artifact for 30 days after `forget`, so a draft's text outlives the draft there; `discard` prints `claude-session forget --purge ID` to remove it early.
- `--idea TEXT` and inline `--set` values are visible in `ps`; use `--idea-file` for anything sensitive.
- An unattended headless draft spends the user's model tokens and its cost is not recorded (interactive runs report no `cost_usd`).

## Platform notes

Linux and WSL are tested; macOS is a local checklist (`/dev/tty`, termios restore, the editor). The interactive draft needs no tmux. Python 3.12 or later.
