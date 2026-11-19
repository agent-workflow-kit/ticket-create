# ticket-create: interface

Status: 0.1.0-design (2026-11-19). This document is the single normative source for ticket-create's rules. `design.md`, the README and the architecture link to its headings and never restate its tables. Section references (architecture §N, design guide §N) point to `../awkit-docs/architecture.md` and `../awkit-docs/design-guide.md`.

ticket-create drafts a ticket with an interactive Claude session and creates it after a human confirmation. `draft` launches a session of kind `draft-ticket` in the current terminal through agent-spawn, validates the artifact the session reports and stores it as a draft. `create` shows the draft on a review screen, lets the human edit it, and after one explicit yes writes it through ticketctl. It is a skill wrapper in layer 2 (architecture §5.2) and works without the harness.

## Overview

- **Draft.** One stored record per ticket under preparation, with a `draft_id` that ticket-create generates before the session starts. It holds the validated artifact, the human's overrides, the session's agent id and the history of create attempts. Everything a retry needs is in the draft.
- **Seed.** Optional text (`--idea`, `--idea-file`) and fixed fields (`--project`, `--type`, `--parent`, `--repo`, `--site`, `--ticket-label`) given to `draft`. Seed text is untrusted and fenced in the prompt. Fixed fields are trusted facts. The stored draft keeps the fixed value: when the artifact's `type` or `parent` differs, or an artifact omits a fixed ticket label, the artifact's value is dropped, the warning `fixed_field_changed` is stored with the dropped value, and the screen shows it as a suggestion. The human adopts it with `create --type`, `--parent` or `--ticket-label`. Ticket labels: the stored list is the fixed list plus the artifact's other labels.
- **Run token.** A fresh random 128-bit value (32 lowercase hex characters) generated for every session run, including a continuation, and stated in the prompt outside the fence. The artifact echoes the token of the newest prompt. It binds an artifact to the run that was launched for it.
- **Origin.** `headless` when any run of the draft was headless, else `interactive`. It is set at T1 to the mode of the first run, can change from `interactive` to `headless` when a later run is headless, and never changes back. The review screen always shows it.
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

Flag grammars: `--project` `^[A-Z][A-Z0-9_]*$`; `--repo` a claude-session repo id (`host/<full project path>`: lowercase host, a non-default port only, then the path); `--type` the type rule of the Slot contract; `--site` `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`; `--assign` `me`, `none` or `^[A-Za-z0-9][A-Za-z0-9:_.-]{0,127}$`; `--sprint` `open`, `next` or `^[0-9]{1,18}$`; `--board` an id or a name of 1 to 128 characters with no control characters; `--ticket-label` the artifact label grammar, and `--ticket-label ''` clears the list. A value that does not match exits 2 `invalid_argument`.

### Global flags

`--json`, `--config PATH`, `--verbose`, `--quiet`, `--version`, `--actor STRING` (grammar `^(human|auto):[A-Za-z0-9._@-]{1,64}$`; the default is `human:<user>` when a human is present, else `auto:unknown`, where `<user>` is `me.name` normalized to the actor grammar: disallowed characters become `_` and the name is cut to 64 characters) and `--decision-id ID` (repeatable, recorded in the audit and forwarded to every ticketctl write and every agent-spawn launch and resume). `--yes` exists only on `discard` (see below). There is no `--yes` on `create`: nothing makes ticket creation skip the human's answer. `create` exits 2 `human_only` with an `auto:` actor (not with `--dry-run`, which builds its plan as `human:<user>`).

With `--json`, exit 0 prints the result object; an exit that adopted a run or built a plan prints that one document, the result object with the error as its `error` member; other failures print the envelope of Output by exit code and error envelope. `--result-file PATH` (on `draft` and `create`) holds the same document, written atomically with mode 0600, whatever the exit code, replacing an existing file; a write failure is a warning on stderr and never changes the exit code. It exists for callers whose terminal the interactive session occupies. The member names of the result objects are those of `schemas/cli/`.

### draft

Runs the Draft order. `--idea TEXT` is a convenience for a short trusted note by a human; its value is treated as untrusted like the file form. `--idea-file F` reads a regular file; `-` reads stdin and is accepted only with `--mode headless` (an interactive session needs stdin for the terminal, so `-` with interactive exits 2 `stdin_conflict`). It is read in Prepare, before the row exists: stdin that is a terminal exits 2 `invalid_argument`; at most 1 MiB plus one byte are read (more exits 2 `invalid_argument`), with a read timeout of 30 seconds (exit 2 `invalid_argument`); the bytes then pass the reader's UTF-8 rules.

An interactive draft needs a terminal on stdin and stdout (agent-spawn's `--placement here` rule). `--json` on stdout would follow the session's screen, so an interactive caller that wants machine output passes `--result-file`. A headless draft needs a seed (`--idea` or `--idea-file`) and runs with no terminal.

`--continue DRAFT_ID` follows the Continue order. It is accepted for drafts in `drafting` (an orphan), `ready`, `failed` and `abandoned` (see the Gate table).

`--no-review` stops after the draft is stored. Without it, an interactive-mode run that reached `ready` on a terminal, when neither `--json` nor `--result-file` is given, runs `post-draft`, releases its lease, and then asks `Review and create now? [y/N]` (the same `/dev/tty` rules as the confirmation prompt); on yes it runs the `create` flow as a new call for that draft and the process exits with that call's code. A declined offer exits 0. A headless-mode run never asks, whatever the terminal.

`--dry-run` writes no row and takes no lease, runs no hook, needs no terminal, runs the read-only prefetch, writes the prompt file under `prompts/` and deletes it, and runs the kind check; it prints the plan (site, project, repo, skill, kind check result, prefetch counts, prompt size, session settings) with an in-memory `draft_id` and launches nothing.

### create

Runs the Create order. It is the only command that writes to the ticket system. Flags override the stored draft: they are applied in memory and saved to the draft (see Draft record and statuses) only by a call that reaches the prompt with a human present; they are shown on the review screen as `overridden`, and `--dry-run` saves nothing. `--ticket-label` given on `create` replaces the draft's label list; `--type`, `--parent` and `--ticket-label` are also how the human adopts a suggestion shown on the screen. `--set NAME=VALUE|@FILE` takes a field name that follows the artifact `fields` grammar and reserved-name list, matched case-insensitively against the stored field names and ticketctl's logical names; a name that violates them exits 2 `invalid_argument`. An inline value is a string. A `@FILE` value is read by the reader as UTF-8 text (cap `limits.artifact_bytes`); when the stored field is an array, or the file holds a JSON array of at most 50 strings and the field is new, it is an array of strings, else a string. `--set NAME=` removes the field from the draft. `--again` starts a new generation (see Dedupe key and attempts).

`--dry-run` runs everything up to the human prompt with no terminal needed: it prints the screen's facts (the resolved plan with the plan hash, the screen state, warnings, required fields) and exits 0 for the `ready` and `already_created` states (the state is in the result) and 22 for `blocked`.

### list and show

`list` prints drafts newest first. By default it hides `created` drafts; `--all` shows them. `--status` filters (repeatable; `orphan` selects the `drafting` rows that `list` shows as `orphan`). Rows carry `failure`, `updated_at` and `agent` (`live`, `parked`, `none` or `unknown`, found as in Gate; null with `--no-agents`, which makes the command offline). A `drafting` row that no process holds the lease of and that has no live agent is shown as `orphan`. There is no `discarded` status (`discard` deletes). `--limit` is 1 to 500 (default 100).

`show` prints one draft: status, origin, site, project, type, repo, title, the attempts and, with `--body`, the body through the sanitizer; it prints `agent_id` and, for a draft with a parked agent, the line `agent-spawn show AGENT_ID shows the session's last report`; it never prints the `run_token`. Without `--body` only the sizes and hashes are printed. `--field NAME` prints one field's value through the sanitizer as text (it cannot be combined with `--json`).

### discard

Removes drafts and tears down any parked agent of each. On a terminal it asks once per call (`Discard N draft(s)? [y/N]`, `/dev/tty` rules; for an `unknown` draft the prompt says a ticket may exist); `--yes` skips the question and is accepted from any actor because the action is local and does not publish (design guide §4.4). Without a human and without `--yes` it exits 13. A draft that is live (its lease is held by a running process or its agent runs, see Gate) is refused (exit 20, `reason: live`, the one reason for in use). The agent it tears down is the one found as Labels says. A `created`, `unknown` or `sprint_failed` draft is discarded without touching the ticket. With any refused item the call exits 20 and prints every item's result; a refused or `teardown_failed` item keeps its row. `discard` records the agent's `csession_id` in the output and prints `claude-session forget --purge ID` as the way to delete the frozen artifact early.

### prune

Deletes drafts past their retention (see `retention.open` and `retention.done`) and tears down their parked agents. It skips a draft whose lease is held or whose agent is live (result `skipped`, `reason: locked` or `live`) and exits 0. It also removes entries older than a day in `prompts/`, `results/` and `tmp/` (files and directories, by `lstat`, never following a symlink); their `object_type` is `prompt`, `result` or `tmp`. `--dry-run` lists what it would remove. There is no cron block; run it by hand or from a harness rule.

### validate

Offline: loads the config (exit 2 `invalid_config` naming the key), checks that the sibling binaries resolve on `PATH` (exit 12 `tool_missing` when one is missing), runs the skill preflight for the configured skill and checks the template files. `--probe` additionally runs `ticketctl validate --probe --json`, `agent-spawn --version` and `ticketctl --version` against the pinned minimums (exit 12 `tool_too_old`) and `agent-spawn launch --dry-run` with a probe prompt (the kind check, no model call). The checks are named `config`, `siblings`, `skill`, `templates`, `ticketctl`, `versions` and `kind_check`. `validate` also lists the resolved permission profile's denies (`profile_denies`) and warns `credential_env_empty` when `credential_env` is empty and `mcp_inherited` when the dry-run profile's `mcp` is `inherit`. It prints `{valid, probe, checks[], profile_denies[], warnings[]}` whatever the exit code (with exit 2 or 12 the failure is its `error` member); any other failed check is data with exit 0.

### export

`export --audit` and `export --log` print the public JSON lines of `audit.jsonl` or `log.jsonl` after `--after-seq N` (a per-file sequence number carried in each record). `export --json` prints JSON lines too; each line is one record.

## Draft record and statuses

The `drafts` table row is the draft. The artifact copy, the overrides and the attempts are part of it.

| Status | Meaning |
|---|---|
| `drafting` | a session run is in progress for a draft that stores no artifact, or a crash left one unfinished (`list` shows `orphan` when no process holds the lease and no agent is live) |
| `ready` | a validated artifact is stored and no write to the ticket system is outstanding; a non-null `failure` means the last `draft --continue` run failed and the stored artifact is still the last good one; a continuation run never changes the status |
| `failed` | a run ended without a valid artifact and none is stored; `failure` names the cause |
| `abandoned` | a human left an interactive session without reporting and no artifact is stored |
| `created` | a ticket exists for the draft (`created_key`) and the sprint step is done or was not requested |
| `sprint_failed` | the ticket exists and the requested sprint step failed or did not finish |
| `unknown` | a write to the ticket system was started and its outcome is not known |

`orphan` is not a stored status: it applies to `drafting` rows only and is how `list` presents a row that no process holds the lease of and that has no live agent. It is resolved by `draft --continue` or `discard`.

`rest_status(draft)` is the status of a draft when no write is in flight:

```text
rest_status(d):
    if any attempt of d.generation has outcome "unknown":            return "unknown"
    if d.created_key and d.sprint.step in ("failed", "pending"):      return "sprint_failed"
    if d.created_key:                                                 return "created"
    return "ready"
```

Stored fields: `draft_id`, `status`, `origin`, `created_at`, `updated_at`, `site`, `project`, `repo`, `type`, `title`, `body_md`, `parent`, `assignee`, `ticket_labels`, `fields`, `notes`, `overrides` (the closed list `title`, `body`, `type`, `project`, `site`, `parent`, `assignee`, `labels`, `sprint`, `fields` of what the human changed), `seed`, `warnings`, `agent_id`, `run_token` of the last run (never printed), `artifact` (`sha256`, `size`), `failure` (`code`, `outcome`, `detail`), `generation`, `created_key`, `created_url`, `sprint` (`selector`, `id`, `name`, `board`, `step`) and the `attempts` list. Text fields hold your own draft text; the database is private (mode 0600 in a 0700 directory).

`seed` holds what `draft` was given: `type`, `parent` and `ticket_labels` are the fixed values (null when not given; a `--ticket-label ''` on `draft` counts as not given), `has_seed` and `seed_bytes` describe the idea text (the text itself is never stored). It is written with the row (T1) and never changed, not by `--continue` either. `warnings` is a list of `{code, field, value}` (draft-time warnings only, today only `fixed_field_changed`; `field` is `type`, `parent` or `ticket_labels`; `value` is the artifact's dropped value, null when the artifact omitted it, model-written and shown only in the gutter zone); it is replaced at every T5 and is empty when the artifact agreed. Site, project and repo are stored as before and are never changed by an artifact.

`title`, `body_md`, `type`, `parent`, `assignee`, `notes`, `artifact`, `failure`, `repo`, `created_key`, `created_url` and `agent_id` are null until set; `origin`, `site`, `project`, `seed`, `warnings`, `generation` and `sprint` are never null (`sprint.selector`, `id`, `name` and `board` are null until a sprint is requested or resolved).

### Transitions

| # | From | Event | To | Attempts and notes |
|---|---|---|---|---|
| T1 | none | `draft` passes its checks and the `pre-draft` hook | `drafting` | row with `origin` (the mode of this run), `seed`, `run_token`; no attempts |
| T2 | none | `pre-draft` veto (11) or escalation (13), or an error before any run produced a session | none | the row is deleted |
| T3 | `drafting` (orphan), `failed`, `abandoned`; `ready` | `draft --continue` passes the gate | `drafting` for the first three; `ready` stays `ready` | new `run_token`; the lease marks the run; a stored artifact and the attempts are kept |
| T4 | the states of T3 | `--continue` vetoed, escalated, refused or failing before a run | unchanged | nothing is written |
| T5 | `drafting`, `ready` | the run ends `done` with a valid artifact (or an adopted finished run does) | `ready` | content replaced, seed values applied, overrides cleared, `warnings` set, `failure` cleared, origin per Overview; agent torn down (a failure is a warning) |
| T6 | `drafting`, `ready` | any other run outcome (Session outcomes table) | `ready` with `failure` set if an artifact is stored, else `failed` or `abandoned` as the table says | the agent stays parked |
| T7 | `drafting`, `ready` | the process dies (kill, reboot) | unchanged | `list` shows `orphan` for `drafting` while no process holds the lease and no agent is live |
| T8 | `created`, `sprint_failed`, `unknown` | `draft --continue` | unchanged | exit 20: `already_created` for the first two, `draft_not_ready` for `unknown` |
| T9 | `ready`, `unknown`, `created`, `sprint_failed` | `create` ends before any write (decline, Ctrl-C, blocked screen, an error before the write) | unchanged | overrides given by this call are saved once the prompt was reached; no attempt; a declined `--again` leaves `generation` unchanged |
| T10 | `ready`, `unknown`, `created`, `sprint_failed` | the write starts (`ticketctl create --yes` is about to run) | `unknown` | one transaction on the lease token: `generation` set (under `--again`: plus 1, `created_key`, `created_url` cleared, `sprint` reset to the call's selector with step `none`), attempt inserted with outcome `unknown`, status set |
| T11 | `unknown` (from T10) | ticketctl exit 0 (`created` or `reused`) | `created`, or `sprint_failed` when a sprint was requested | attempt `created` or `reused`; `created_key`, `created_url` set; `sprint.step` is `pending` when a sprint was requested |
| T12 | `unknown` (from T10) | ticketctl exit 20 `already_exists` | `rest_status` | attempt `failed`; the screen shows the `already_created` state |
| T13 | `unknown` (from T10) | every other definite ticketctl answer: exit 2, 10, 11, 13, 20 (any other reason), 21, 22, or exit 12 with any code except `outcome_unknown` | `rest_status` | attempt `failed` |
| T14 | `unknown` (from T10) | ticketctl exit 12 `outcome_unknown`, ticketctl's own exit 1, death by signal, unreadable output, or a crash of ticket-create | `unknown` | attempt stays `unknown` |
| T15 | `ready`, `unknown` | the human answers `accept` on the `already_created` state | `rest_status` | no attempt row; the ticket is read (see Screen state); `created_key`, `created_url` set; every `unknown` attempt of the current generation becomes `reused` |
| T16 | `created`, `sprint_failed` | the sprint step fails or does not finish | `sprint_failed` | key kept; exit 12 `partial` |
| T17 | `created`, `sprint_failed` | a sprint-only run succeeds or fails | `created` or `sprint_failed` | no attempt; `sprint.step` records it |
| T18 | any except a live `drafting` draft | `discard` | row deleted | the parked agent is torn down; refused with `reason: live` otherwise |
| T19 | any not locked or live | `prune` after retention | row deleted | the parked agent is torn down |

`stale_ticket` is not a transition for `create` (the plan hash of `create` carries no ticket-system facts, so a changed ticket system cannot make it differ). If ticketctl ever returns it for `create`, it falls under T13 and exit 12 `lower_failed`. It stays a re-plan case in the Sprint step only.

Attempts are append-only. They are never changed after the call ends, `--continue` never touches them, and they are deleted only with the row. The attempt `outcome` starts as `unknown` and is overwritten once ticketctl's answer is known, so a crash needs no extra state. `generation` starts at 0, `--again` adds 1, and the dedupe key of generation g is `ticket-create:<draft_id>` for g = 0 and `ticket-create:<draft_id>:<g>` otherwise.

### Gate

`create`, `discard` and `draft --continue` enter through one function, and the Gate table decides the rest. Locked (exit 12) is for contention and lost races only.

```text
def enter(draft_id, command):                 # create, discard, draft --continue
    row = read_row(draft_id)                  # no lease; exit 10 when missing
    seen = row.status
    if row.status == "drafting" and is_live(row):
        exit_refused(20, "draft_not_ready" if command == "create" else "live")
    lease = take_lease(draft_id)              # exit 12 unavailable (locked) on contention; --dry-run takes none
    row = read_row(draft_id)                  # re-read under the lease
    if row.status != seen:                    # a race: apply the gate once more, never loop
        if row.status == "drafting" and is_live(row): release(lease); exit_refused(20, same)
    return row, lease                         # then the Gate table decides the rest

def is_live(row):                             # row.status == "drafting"
    if lease_held_by_live_process(row): return True
    agent = find_agent(row)                   # see below
    return agent is not None and agent.status in ("launching", "live", "tearing_down")

def find_agent(row):
    if row.agent_id: return agent_spawn_show(row.agent_id)       # stored id first
    hits = agent_spawn_list(labels=["draft="+row.draft_id, "owner=ticket-create"])
    hits = [h for h in hits if h.kind == "draft-ticket" and h.status != "torn_down"]
    return hits[0] if len(hits) == 1 else None                   # none or ambiguous: no agent
```

| Status | `create` | `create --again` | `create --sprint S` | `draft --continue` | `discard` |
|---|---|---|---|---|---|
| `drafting` | exit 20 `draft_not_ready` (live or orphan) | same | same | allowed for an orphan (lease free and no live agent), exit 20 `live` otherwise | exit 20 `live` while the draft is live (lease held or agent running) |
| `ready` | full create | exit 2 `invalid_combination` | an override of the full create | allowed | allowed |
| `failed`, `abandoned` | exit 20 `draft_not_ready` | same | same | allowed | allowed |
| `unknown` | full create with the same key | new generation, full create | an override of the full create | exit 20 `draft_not_ready` | allowed, its prompt says a ticket may exist |
| `created` | exit 20 `already_created` | new generation, full create | sprint-only run | exit 20 `already_created` | allowed |
| `sprint_failed` | sprint-only run (the stored sprint) | new generation, full create | sprint-only run | exit 20 `already_created` | allowed |

## Draft order

`draft` performs these steps in order. `draft --continue` follows the Continue order. An error before any run produced a session (steps 3 to 6: a prompt-file error, a failed kind check, or agent-spawn ending the launch with no run: exit 10, 11, 12, 13, 20 or 21) deletes the row of a new draft and tears down an agent record the launch made (read `agent_id` from the error envelope; a failed teardown is a warning), and leaves a continued draft as it was. Prefetch failures are warnings and never fail the draft.

1. **Prepare.** Load config. Site: `--site`, else `ticketctl.site`. Project: `--project`; else, when the site is known and `projects` names exactly one `<site>/<KEY>` for it, that key; else, when the site is unknown and `projects` names exactly one entry, that entry (and its site); else exit 2 `missing_option` naming `--project`. A site still unknown after the project is known is resolved with `ticketctl sites resolve <PROJECT>-1 --json` (offline; it applies ticketctl's own rules including the only-site rule; the number is a placeholder); several candidates exit 21 `ambiguous` with the site names. Type: `--type`, else `projects.<site>/<KEY>.default_type`, else none. Repo: `--repo`, then `projects.<site>/<KEY>.repo`, then the cwd's `origin` remote, then none. The repo is a claude-session repo id: lowercase host, a non-default port only, then the path (`host/<full project path>`), derived from the `origin` URL by stripping the scheme, `user@`, a default port and a trailing `.git` (the scp form `git@host:path` becomes `host/path`); a URL that does not give a valid id means no repo. `--repo` and the config values must already be an id. Check the siblings against the pinned minimums (exit 12 `tool_missing` or `tool_too_old`, naming the install command), run the skill preflight, and refuse a headless draft with no seed (exit 2 `missing_seed`). An interactive draft with no terminal on stdin and stdout exits 2 `no_terminal`.

   **Skill preflight.** For `plugin:skill` look for `skills/<skill>/SKILL.md` in each plugin directory passed (the bundled one and `plugin_dirs`; the plugin name is the manifest's name, not the directory's) and exit 12 `skill_not_found` naming the directories searched. A bare skill name cannot be checked, so a warning is printed and the launch goes ahead; a user or project skill also needs `setting_sources` to include its source. The check applies to `--skill NAME`. Template and plugin files must have a base name of 1 to 128 characters of `[A-Za-z0-9._-]`. `validate` runs the same check.
2. **Record.** Generate `draft_id` and `run_token`, take the per-draft lease and write the `drafting` row (T1: `origin` is the mode of this run, `seed` is what `draft` was given). The `pre-draft` hook runs here; a veto (exit 11) or escalation (exit 13) of a new draft deletes its row again.
3. **Prefetch** (see Prefetch). Failures here are warnings (the session simply gets less context).
4. **Prompt file.** Write the prompt file (see Prompt file) with mode 0600 under the state directory's `prompts/`, created `O_CREAT|O_EXCL|O_NOFOLLOW` with a random name, and, when a template applies, the template as a trusted input: it is copied to a private file named `template.md` in a mode 0700 directory under `tmp/` (read through the reader, cap 256 KiB) and that copy is passed as `--input-file`; the directory is deleted when the run ends. A resume passes no input; the earlier run's copy stays readable at its old path.
5. **Kind check.** Run `agent-spawn launch --dry-run --json` with the final flags and no `--result-file`, and require exit 0, `agent_settings.kind == "draft-ticket"`, `reads_third_party == false`, `would_refuse == null`, an empty top-level `warnings` and an empty `session_plan.warnings` and `session_plan.profile.fallback` not true (a misspelled kind exits 0 with only those signals). `session_plan.profile.mcp` is read; a value `inherit` is the warning `mcp_inherited` (not a failure; the bundled profile denies `mcp__*`).
6. **Launch.** Snapshot the terminal (termios) and run `agent-spawn launch` (see Session launch) with a fresh `--result-file` path under `results/`. Restore the terminal in a `finally` (see Prompt and terminal rules).
7. **Outcome.** Read the result (see Session launch for the headless reading). The result is unusable when the file is missing, is not JSON, has `dry_run` or has `run: null`; then find the agent with `find_agent` (Gate) and read `agent-spawn show <agent_id> --json` instead. Map the outcome with the Session outcomes table. On `done`, read the frozen artifact with the Artifact reader and validate it against the Slot contract.
8. **Store.** On a valid artifact apply T5 and tear the agent down with `agent-spawn teardown <agent_id>`; a failed teardown after a valid artifact is stored is a warning, the draft stays `ready`, and `prune` and `discard` retry it. On any other outcome apply the table (T6; the agent stays parked). Delete the prompt file and the result file.
9. **Finish.** Run the post-draft duplicate check (see Prefetch), the `post-draft` hook and the audit record, print the result and offer the review as the `draft` text says.

## Continue order

`draft --continue DRAFT_ID [--mode M]` follows these steps.

```text
draft --continue DRAFT_ID [--mode M]:
 1. Prepare: row, lease = enter(draft_id, "draft --continue")
    apply the Gate table; the fixed-field flags and --idea/--idea-file are exit 2 invalid_combination.
    The seed rule (missing_seed) does not apply: the stored draft is the seed.
    pre-draft hook payload: has_seed = row.seed.has_seed, seed_bytes = row.seed.seed_bytes, continue = true.
    A veto or escalation changes nothing (exit 11 or 13).
 2. Adopt (only a `drafting` orphan whose agent is parked):
    run = last run of agent_spawn_show(agent_id)
    if run.outcome == "done" and run.report and not run.report.implicit:
        run Draft order steps 7 to 9 on this run (artifact reader checks run_token == row.run_token);
        if the artifact reads valid: T5, done, no new run.
        if it fails with run_token mismatch: not ours; continue with step 3.
        any other failure: T6 as for a live outcome; stop.
 3. Pick the way:
    agent = find_agent(row)
    if row has no artifact or row.failure is set:
        if agent is parked, not torn down, and its last run has first_working_at: resume it.
        elif row has an artifact: new session from the stored artifact (tear down any parked agent first; a failed teardown is a warning).
        else: exit 20 draft_not_resumable.
    else:                                   # ready, failure null: the last run ended done
        new session from the stored artifact (same teardown rule).
    If the resume dry-run or the resume call fails before a run starts (exit 10, 11, 12, 13, 20, 21) and an artifact is stored:
        tear down the agent (warning on failure) and take the new-session path once. Otherwise map the failure as the Draft order says.
 4. Record: new run_token; lease held; a row without an artifact becomes `drafting` (T3); a row with an artifact stays `ready`.
    Prompt header (trusted): draft_id, run_token, the fixed fields from `seed`, the template file name if any, and for a resume one closed
    sentence: "this is a continuation: write a new draft.json that echoes this run_token". When the row has a `failure` from a refused
    artifact, add the closed sentence "the last draft.json was refused: <code>/<detail>" (the two closed values of `failure`, never a field name).
    A new session from a stored artifact carries it as the fenced untrusted block `previous`.
 5. Kind check and call, outcome and store: as Draft order steps 5 to 9, with T5 or T6.
```

A draft whose agent is live exits 20 `live`. The `post-draft` hook and the audit record run for an adopted run as for any run. A resumed conversation holds the old token in its history; the artifact must echo the token of the newest prompt. In a resume, the kind check runs `agent-spawn resume AGENT_ID --dry-run --json` with the flags below (without `--result-file`) and requires exit 0, `would_refuse` null and empty `warnings`; then the same call runs with `--result-file`.

Resume flags (all passed every time so safety does not depend on the carried values): `--mode interactive --placement here`, or `--mode headless --no-implicit-done --wait` for a headless continuation; `--prompt-file`; `--result-file`; `--json`; `--permission-profile`; `--sandbox`; `--setting-sources`; `--env-allowlist-only` (the same names as launch); one `--plugin-dir` per directory; `--no-promote`; `--actor`; `--decision-id` as given; `--model` and `--timeout` when configured or given. It passes no `--repo`, worktree flag, `--kind`, `--skill`, `--label` (the agent keeps its labels) or `--ephemeral`.

## Create order

`create` is human-only as a whole. `--actor auto:...` exits 2 `human_only` (except with `--dry-run`), the actor passed to ticketctl on every call is `human:<user>`, and the audit records the same actor.

```text
def create(draft_id, flags):
    if not human_present():                      # /dev/tty opens; skipped by --dry-run
        exit(13, "needs_human")
    if actor is auto and not dry_run: exit(2, "human_only")
    row, lease = enter(draft_id, "create")        # Gate; --dry-run takes no lease
    ctx = Context(attempts_this_call=[], sprint_ran=False, accepted=False, workdir=None, exit=None)
    try:
        run_steps(row, flags, ctx)                # steps below; sets ctx.exit, may return early
    except ExitRequest as e:
        ctx.exit = e                              # every `exit(...)` inside run_steps raises this
    except BaseException as e:
        ctx.exit = internal_error(e)
    finally:
        finish(row, lease, ctx)                   # the single place for every exit
    sys.exit(ctx.exit.code)

def finish(row, lease, ctx):                      # never raises; each part is independent
    delete_dir(ctx.workdir)                       # also deleted when step 3 builds a new one
    for a in ctx.attempts_this_call:
        if not a.outcome_audited: audit_outcome(a)          # pairs every `attempted` record
    if ctx.attempts_this_call or ctx.sprint_ran or ctx.accepted:
        run_hook("post-create", result=post_create_result(ctx))   # exactly once; failure of the hook is logged only
    release_lease(lease)                          # compare-and-set on the lease token; last
    write_result_file_and_output(ctx)             # --json / --result-file, whatever the exit code

def post_create_result(ctx):
    last = ctx.attempts_this_call[-1] if ctx.attempts_this_call else None
    sprint_failed = ctx.sprint_ran and row.sprint.step == "failed"
    if last is None:                              # sprint-only run or accept without an attempt
        return "partial" if sprint_failed else ("reused" if ctx.accepted else "created")
    if last.outcome in ("created", "reused"):
        return "partial" if sprint_failed else last.outcome
    return last.outcome                           # "failed" or "unknown"

def run_steps(row, flags, ctx):
    # 1. Gate: apply the Gate table to `row`; a sprint-only run goes to step 7.
    # 2. Overrides: apply flags to a copy in memory; freeze rule; resolve --sprint; the dedupe key is the key of
    #    generation g, or g+1 under --again (kept in memory until step 6). Nothing is saved yet.
    # 3. Plan: new workdir under tmp/ (the previous one is deleted), files, `ticketctl create --dry-run`, state.
    # 4. Duplicates: search on the first round and when the title changed.
    # 5. Review: save the overrides given by this call (the prompt was reached, so a decline saves them too); draw the screen;
    #    read one answer. `e` edits and returns to 3; `v` shows the cut items; ready+y (no item cut) goes to 6;
    #    `accept` (state already_created) goes to 6a; any other answer exits 20 declined.
    # 6. Write: check the file digests; one transaction (compare-and-set on the lease token): generation, attempt (outcome unknown),
    #    status unknown, and under --again clear created_key, created_url and reset sprint; audit `attempted`; call ticketctl create --yes;
    #    outcome rule and Lower tool exits; on escalated, the escalation question.
    # 6a. Accept: the `accept` function of Screen state; it ends in step 7.
    # 7. Sprint: when a sprint is chosen and the ticket exists (after 6, 6a or a sprint-only run): the Sprint step.
    # 8. Return; `finish` prints the key and URL.
```

`post-create` does not run for a declined, blocked, `needs_human`, dry-run or error-before-the-write call (see Hooks). The steps in detail:

1. **Gate.** Without a human present (`/dev/tty` does not open) exit 13 `needs_human` before anything else; `--dry-run` skips this check. Enter through the gate function (contention is exit 12 `unavailable`, `reason: locked`; `--dry-run` takes no lease). Apply the Gate table. A sprint-only run goes to the Sprint step.
2. **Overrides.** Apply the flags to a copy of the draft in memory. Check the freeze rule. Resolve `--sprint` (see Sprint step). Nothing is saved yet.
3. **Plan.** Create a fresh work directory under `tmp/` (mode 0700, random name) and write the title, the body and every field value into it, one file each, mode 0600, `O_EXCL`, and remember the sha256 of each file. Run `ticketctl create --dry-run --json --actor human:<user> --site=S --project=P --type=T --title-file=F --body-file=B` with `--parent=`, `--assign=`, `--label=`, `--set=NAME=@FILE` and `--dedupe-key=KEY`, each only when set. Values were checked when stored or parsed; a stored value that fails its grammar at this point is exit 1 `internal`. Derive the screen state from the answer (Dedupe key and attempts). Everything the screen shows comes from the plan (`changes[]` and `steps`) where the plan carries it, and from the draft otherwise. The body shown is the `to` member of the plan's `changes[]` entry named `description`. It must be a string (ticketctl's Markdown rendering of the final text, attribution paragraph included). If it is not a string, or the entry is missing, `create` exits 12 `lower_failed` (the screen cannot show what would be sent) and nothing is written.
4. **Duplicates.** Run the near-duplicate search on the final title on the first round and again only when the title changed; hits are warnings.
5. **Review.** Save the overrides given by this call, show the screen, read one answer. `e` edits (Editor), saves the new text to the draft and returns to step 3 with new files. `v` shows the cut items (Review screen). A `blocked` state offers only `e`. A `ready` `y` goes to step 6 (it is refused while any item is cut); `accept` on the `already_created` state goes to step 6a; any other answer ends as the state table says.
6. **Write.** Check that the sha256 of every file in the work directory still equals the remembered value (a difference exits 1 with `reason: input_changed` and sends nothing). In one transaction on the lease token (T10) insert the attempt (outcome `unknown`), set the generation, set the status `unknown` and, under `--again`, clear `created_key` and `created_url` and reset `sprint`. Write the audit `attempted` record. Run `ticketctl create --yes --expect-plan HASH --dedupe-key KEY --actor human:<user>` with the same arguments and files as step 3, then apply the outcome rule and the Lower tool exits table. `escalated` runs the escalation question. The audit outcome record is written by the finish step.

   6a. **Accept.** Run the `accept` function of Screen state (T15); it ends in step 7.
7. **Sprint.** If a sprint was chosen and the ticket exists, run the Sprint step; a failure sets `sprint_failed`, keeps the key and exits 12 `partial` with the key in the result.
8. **Return.** `finish` (above) writes the audit outcome records, runs `post-create`, deletes the work directory, releases the lease and prints the key and URL.

**What the plan hash covers.** ticketctl's hash covers the action, site, project, assign target, field names with value hashes, body hashes, labels, sprint id and dedupe key. It does not state that it covers the title, the type or the parent. ticket-create does not rely on it for those: it renders them from the dry-run plan and it checks the file digests of step 6, which narrows the window between the check and ticketctl's read.

**Escalation.** When ticketctl exits 13 `escalated` after the human's yes, the attempt is `failed` (nothing was written). Show the hook names and the sanitized reason (as a model-free note: ticketctl's hook text is untrusted), keep the draft and ask a second question on the same prompt rules: `A hook asked for review (NAMES). Override and create? [y/N]`. On yes run the same call again with `--override-escalation=ticketctl` and one `--decision-id` per id ticketctl returned, as a new attempt with the same key; the audit records `override_escalation: ticketctl`. Any other answer ends with exit 13 `escalated` and the draft unchanged. The same question covers `pre-sprint`.

**Edits that are not a dry-run problem.** A body that fails ticketctl's validation is `blocked`; `e` edits it and the loop returns to step 3. Fields that fail are fixed with `--set NAME=VALUE` (or `--set NAME=` to drop a model-written field from the draft) on the next `create`.

## Prefetch

ticket-create gathers context deterministically before the session, with the same environment and credentials as its own process; the session itself never calls ticketctl. All ticketctl calls except `sites resolve` carry `--site`, and all carry `--json`, take their text from files, and have a total time budget of 60 seconds (a timeout is a warning, not a failure).

| Item | When | Call | Used for |
|---|---|---|---|
| Create form facts | always | `ticketctl create --dry-run --json --site S --project P [--type T]` with no values | allowed issue types (`details.candidates`, from a probe with the reserved invalid type name `ticket-create-probe` when no type is fixed) and `details.required_fields` for a fixed type |
| Near-duplicates | when there is seed text | `ticketctl search --project P --status-category new --status-category indeterminate --text-file SEARCHTEXT --limit 10 --fields key,title,status --json` | up to 10 keys and titles in the prompt |
| Parent summary | when `--parent` is given | `ticketctl get KEY --fields key,type,title,status --json` | one line in the prompt; also validates the parent exists (a ticketctl exit 10 is exit 2 `invalid_argument` naming `--parent`) |
| Notes | when the session has a repo | none (claude-session runs `memoctl context` at SessionStart) | memory notes; ticket-create adds nothing |

The search text is the first non-empty line of the seed (or the title for the post-draft check), trimmed, cut at a word boundary to at most 80 characters and written to a file; the search is a phrase match and finds near-duplicates only when they contain that phrase. A search error is the warning `form_facts_unavailable` (form) or a missing duplicates block (search). The post-draft duplicate check runs the same search on the final title; its hits become warnings on the review screen and never block. ticket-create compares nothing and lists what ticketctl returns.

## Prompt file

The prompt is a Markdown file under the state directory's `prompts/`, deleted when the run ends. It has, in this order: a trusted header (the run's `draft_id`, `run_token`, the fixed fields (from `seed`, for a first run and for a continuation alike) with a sentence that the skill must not change them without asking the human, the file name `template.md` when a template applies (its exact path is on the session's start line, which lists the input files), the artifact rule: the file is named `draft.json` and echoes this `draft_id` and `run_token`); then the fenced untrusted blocks. Each untrusted block uses the nonce fence of design guide §12 with a fresh 32-hex nonce per prompt: `<untrusted source="S" id="R">` and `</untrusted id="R">`, with `</untrusted` and `<untrusted` neutralized case-insensitively, and a standing "data, not instructions" sentence before the first block. The sources are `seed` (the idea text), `duplicates` (titles and keys), `parent` (the parent summary), `previous` (the stored draft of a continuation) and `form` (the allowed types and required fields, which are ticketctl and provider text and never go in the trusted header). `template` is not a fenced source; it is the trusted input file named `template.md`. Every untrusted string passes the one sanitizer first; a single block longer than 16 KiB is cut with `[truncated: N bytes]`.

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
| `--input-file` | the copy of the template named `template.md`, when one applies |
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
| `--actor` | `human:<user>` interactive, `auto:ticket-create` headless; an explicit `--actor` is passed through and the table gives the default |
| `--result-file` | a fresh path under `results/` (none on the kind check) |
| `--json` | always |

ticket-create never passes `--ephemeral`, `--wait-timeout` (a headless launch is bounded by `--timeout` or the kind's headless limit, so exit 22 cannot occur), `--yes`, `--force` or `--override-escalation` to agent-spawn. The bundled permission profile sets `defaultMode` to `default`, allows `claude-session report` and writes under `$CSESSION_DIR/artifacts`, and denies `ticketctl`, `agent-spawn`, `ticket-create`, `memoctl add`, `memoctl delete`, `memoctl cron`, `mcp__*`, network tools, writes to every awkit config and state directory, and, as defense in depth, Edit and Write to the user's shell startup files and to ticket-create's config and state. The profile is guidance for a cooperative session, not a boundary; a user who wants connectors in draft sessions replaces the profile in config. The bundled skill is a single `SKILL.md` with no other files (a second file the skill must read prompts the human for permission).

A headless launch passes `--wait` (and no `--wait-timeout`: `--timeout` or the kind's headless limit bounds the run, and exit 22 cannot occur). `agent-spawn launch ... --mode headless --no-implicit-done --wait --result-file R --json` exits with the run's outcome code. Read R (a result object with `run.outcome` and `run.report`; a missing file, unreadable JSON or a file with `dry_run` or `run: null` is unusable), and if it is unusable find the agent with `find_agent` (Gate) and read the last run of `agent-spawn show AGENT_ID --json`. The kind check (`launch --dry-run`) passes no `--result-file`.

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

Ending with exit 11, 12 or 13 and no run is the pre-run error rule of the Draft order. The outcome names are agent-spawn's. A parked agent keeps its attention flag in agent-spawn until the draft is discarded or pruned; this is intended, ticket-create never acknowledges it. A run that ends in any row other than `done` after a stored artifact exists leaves the draft `ready` with `failure` set (T6).

In an interactive run a `needs-input` report does not end the session; the human keeps typing and a later `done` report supersedes it. Only a session that ends while still in `needs-input` produces exit 13.

## Slot contract

The `draft-ticket` slot (architecture §5.5) is filled by a skill that receives the prompt file and optionally an input file, and writes one JSON artifact. Every grammar in this document is matched with a full match; a value with a trailing newline never matches. The skill's name is the config key `skill` (any name Claude Code accepts). A replacement skill must follow this contract.

Inputs: the prompt file (the first line the session sees gives the session directory) and, when a template applies, the template as an input file named `template.md` (the session's start line gives its path). The template is looked up by the fixed type (flag or default type), else by `default`.

Output: the skill writes `draft.json` directly inside `$CSESSION_DIR/artifacts/` and reports it as the artifact of its `done` report. The artifact is one JSON object with exactly these keys (unknown keys are refused):

| Key | Type | Rule |
|---|---|---|
| `schema_version` | integer | `1` |
| `draft_id` | string | equal to the prompt's `draft_id` |
| `run_token` | string | equal to the newest prompt's `run_token` |
| `type` | string | the first character a letter or digit (any script), then at most 63 letters, digits, spaces and the characters `_ . - ( ) & / +`, no trailing space; `--type` and the `type_name` definition use the same rule |
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

One reader opens every untrusted file (the frozen artifact, the template, a `--idea-file`, the editor's file, a `--set NAME=@FILE` file). It opens with `O_NOFOLLOW|O_NONBLOCK`, requires a regular file by `fstat`, checks the size against the cap before reading (`limits.artifact_bytes` for an artifact, 1 MiB for an idea file, 256 KiB for a template, `4 * 29257 + 1` bytes for the editor file, `limits.artifact_bytes` for a `--set NAME=@FILE` file), decodes strict UTF-8 without a byte order mark and, for JSON, rejects duplicate keys, `NaN` and `Infinity`, lone surrogates and a nesting depth over 8. Numbers are parsed with an integer or decimal rule that rejects exponents, non-finite values and more than 400 digits. In title, field names and field values it rejects every code point the sanitizer would replace (body and notes keep newline and tab); a tab in `title` and in field names is rejected (ticketctl turns it into a space, so the stored draft would differ from the plan). For the frozen artifact it also checks that the file's `size` and `sha256` equal `run.report.artifact`, that `report.status` is `done`, `report.implicit` is false and the reported artifact's name is `draft.json`. Any failure is a validation failure with a `code` (`artifact_missing`, `artifact_invalid` with a `detail`: `frozen_file`, `json`, `schema`, `draft_id`, `run_token`, `field`), never an internal error; the validator catches every exception.

## Review screen

`create` shows the screen on `/dev/tty`. No pager is used.

```text
Screen = items (the gutter zone) + facts (one row each) + prompt. R rows and C columns come from TIOCGWINSZ; unknown or non-positive values give 24 and 80.
If C < 40 the screen prints "terminal too narrow: 40 columns needed" and offers only cancel and e.

def render(state, plan, draft, R, C):
    W = C - 4                                   # "│ " gutter plus 2 cells of slack for width-table differences
    items = []                                  # model- or provider-written text only, fixed order
    items += [Item("title", draft.title)]
    items += [Item("body", plan.change("description").to)]      # as ticketctl will send it, attribution paragraph included
    if draft.notes: items += [Item("notes", draft.notes)]
    for name, value in draft.fields:  items += [Item("field " + name, text(value))]
    if draft.ticket_labels: items += [Item("labels", ", ".join(draft.ticket_labels))]
    if draft.assignee not in (None, "me", "none"): items += [Item("assignee", draft.assignee)]
    for w in draft.warnings: items += [Item("suggested " + w.field, w.value)]
    if sprint: items += [Item("sprint", sprint.name)]
    if state.existing_ticket: items += [Item("existing ticket", type + ": " + title)]
    if state.provider_text: items += [Item("reason", provider_text + hint_for(state.reason))]   # blocked and escalation screens
    if duplicates: items += [Item("possible duplicates", "\n".join(duplicates))]
    facts = build_facts(...)                    # one row per fact, each clipped to C-1 cells with "<...>"; never folded
    F = len(facts)
    B = R - F - 1 - 1                           # gutter rows: minus the facts, the prompt row and one spare row
    for it in items:
        it.rows = fold(it.text_with_label, W)   # tab -> one space; newline starts a row; width: East Asian W/F = 2, combining = 0, else 1;
                                                # a code point is never split; every row carries the "│ " prefix
    full  = [len(it.rows) for it in items]
    floor = [min(f, 2) for f in full]           # a cut item shows one text row and one marker row
    if sum(floor) > B:
        refuse("terminal too small: %d rows needed" % (sum(floor) + F + 2))     # offers only e and cancel; y is not offered
    alloc, spare = floor[:], B - sum(floor)
    while spare > 0 and any(a < f for a, f in zip(alloc, full)):                # one row at a time, in item order
        for i in range(len(items)):
            if spare > 0 and alloc[i] < full[i]: alloc[i] += 1; spare -= 1
    write("ESC[?7l")                            # autowrap off for the whole draw; restored with ESC[?7h in the terminal finally
    for it, a, f in zip(items, alloc, full):
        if a == f: draw(it.rows)
        else:      draw(it.rows[:a-1]); draw_marker("[... %d more characters; sha256 %s]" % (chars_hidden(it, a-1), sha12(it.text)))
    draw(facts); draw(prompt_line(state))
```

Rules that follow:

- Facts rows (closed): `site`, `project`, `repo`, `type` (quoted, `"` and `\` escaped), `parent`, `assignee` (`me`, `none` or `account id (see above)`), `labels: N`, `fields: N`, `sprint: id N board "B"` (quoted), `origin`, `overridden`, `warnings` (codes with counts), `plan` (hash), `title: "..."` (quoted and clipped to one row, `"` and `\` escaped, with the length in characters) and, when `generation` is above 0, `earlier attempts: N (KEYS)` (the ticket keys of earlier generations and the count of earlier unknown attempts). The full title is the first gutter item. F is the number of rows present (at most 15).
- A cut item always ends with the marker; the marker's 12-hex digest is the start of the item's sha256; a cut body is read in full with `e`, any field with `show DRAFT_ID --field NAME`, and every cut item with `v`.
- `gutter(text)` is the one function that prints provider, hook and model text on every screen (create, blocked, escalation, sprint-only, `already_created`): such text is never printed outside the zone, and every prompt line holds closed text only: `Create this ticket? [y/N]`, `K item(s) are cut. v shows them in full, e edits, anything else cancels:`, `Cannot create (see above). e edits the title and body; anything else stops:`, `A hook asked for review (NAMES). Override and create? [y/N]` (NAMES from the user's own hook config, quoted), the `already_created` lines of Screen state, `Add KEY to sprint? [y/N]`.
- Blocked hints (closed text appended to the `reason` item): `missing_required_fields`: "pass --set NAME=VALUE for each required field"; `invalid_value`, unknown type: "pass --type with one of the allowed types"; parent not found: "pass --parent"; board not found: "pass --board"; `unknown_field`, `invalid_combination`, `ambiguous_field`: "remove or rename the field with --set NAME="; `body_too_large`, `empty_body`, `node_not_allowed`, `secret_in_body`: "press e to edit the body". `e` is always offered.
- The facts rows and the prompt are drawn last, so nothing in the gutter zone can appear below them.
- While any item is cut, `y` is refused. The prompt is `K item(s) are cut. v shows them in full, e edits, anything else cancels:`. `v` prints each cut item in full through the same fold, in pages of R - 2 rows with `Enter` for the next page and `q` to stop; after the last page of every cut item the screen is drawn again with the normal prompt and `y` is accepted until the next edit or override. `q` stops the viewer and draws the screen again with the same cut prompt (items not shown stay unseen).

## Prompt and terminal rules

- **Presence.** A human is present exactly when `os.open("/dev/tty", O_RDWR | O_NOCTTY)` succeeds. The state of stdin, stdout and stderr is never tested for this (spike evidence: it is wrong in both directions).
- **Prompt.** The confirmation reads one line from `/dev/tty` after forcing canonical mode, echo and signal generation, and flushing pending input with `tcflush(fd, TCIFLUSH)` before and again after the read. Only `y` or `yes` (trimmed, case-insensitive) accepts. EOF, an over-long line (over 64 characters), pending extra input, a partial line and a bracketed paste decline; Ctrl-C ends the call with exit 130 and changes nothing. The same prompt serves every yes-or-no question in the tool.
- **Restore.** Before launching the session, the editor or any child that uses the terminal, ticket-create takes a termios snapshot of `/dev/tty`. In a `finally` it restores it with `tcsetattr(TCSAFLUSH, snapshot)` and writes the reset sequence that turns off bracketed paste, mouse tracking and the alternate screen, shows the cursor and turns autowrap back on (`ESC [ ? 7 h`). This happens on every exit, including a signal-killed child.
- **Cooked output.** Everything untrusted that is printed passes the Output sanitizing rule first.
- **Signals.** Before the launch call starts, SIGINT, SIGTERM or SIGHUP end the call with 130, 143 or 129 and delete a new draft's row. While `launch` or `resume` runs, ticket-create ignores SIGINT (the session owns the terminal) and forwards SIGTERM and SIGHUP to the agent-spawn child, waits for it and maps its exit; the draft is stored by the outcome rules, or stays `drafting` (an orphan) if ticket-create itself dies. During `ticketctl create --yes` and `sprint add --yes` it ignores all three signals until the child ends, then runs the single finish of the Create order and exits 130, 129 or 143.

## Editor

`e` writes `title`, a blank line and `body_md` to a file named at random with mode 0600 created `O_EXCL` inside a mode 0700 directory under the state directory's `tmp/`, and runs the editor command from config `editor` (default `$VISUAL`, then `$EDITOR`, then `vi`) with the file path as its last argument (the command string is split with `shlex`, never run through a shell; it comes from the user's own config or environment, never from a draft). The editor inherits the terminal and the tool's environment minus the names in `credential_env`; the snapshot and restore rules apply. After it exits, the file is re-read with the Artifact reader (a symlink or FIFO swap, a size over the cap, an empty result, a non-zero exit or a death by signal aborts the edit and keeps the draft unchanged), the first line becomes the title (at most `limits.title_chars`, no control characters) and the rest the body. The new text is saved to the draft, listed in `overrides` and sent through a new dry-run.

## Dedupe key and attempts

A draft has an integer `generation` (0 at first). The dedupe key of generation g is `ticket-create:<draft_id>` for g = 0 and `ticket-create:<draft_id>:<g>` for g > 0; it matches ticketctl's key grammar and never contains title or type text. `draft_id` comes from ticket-create's own record, never from the artifact. `create --again` is the only way to raise the generation: it is accepted for `created`, `sprint_failed` and `unknown` drafts and means "I checked the ticket system; make another ticket". `--again` computes the key of generation g+1 in memory for the plan; T10 writes it (and clears `created_key`, `created_url`, resets `sprint`). `--again --dry-run` prints the g+1 plan and writes nothing; a declined `--again` changes nothing. `--again --sprint S` resolves S for the new generation.

ticketctl labels the ticket it creates with a value derived from the site, the project and the key, and looks it up inside the project, and it binds a key to the issue type and the exact title. Therefore, once an attempt of the current generation has outcome `unknown`, `created` or `reused`, the draft's site and project are frozen: `create --site` or `create --project` with a different value exits 20 `attempt_recorded` and changes nothing. Type and title edits are allowed (ticketctl answers `reused`, or `already_exists` for a changed type or title, which selects the `already_created` state below). `--again` starts a new generation and lifts the freeze for it.

An attempt is one `ticketctl create --yes` call: `n` (1, 2, ...), `generation`, `key`, `site`, `project`, `plan_sha256`, `body_sha256` (sha256 of the draft's `body_md`), `started_at`, `outcome` (`unknown`, `created`, `reused`, `failed`), `attempt_id` (ticket-create's own, used to pair audit records), `ticketctl_attempt_id`, `ticket_key`. Outcome rule for the answer of a `ticketctl create --yes` call: exit 0 gives `created`, or `reused` when ticketctl says `reused`; ticketctl's own exit 1, a death by signal, an unreadable result and exit 12 `outcome_unknown` keep `unknown`; every other answer, including exit 2, 13 and 20 reasons, is definite and gives `failed`. The sprint step is not an attempt; it is recorded in the draft's `sprint` (`selector`, `id`, `name`, `board`, `step`: `none`, `pending`, `done` or `failed`).

### Screen state

Every `create` round runs `ticketctl create --dry-run` with the key of the current generation and derives one state from the answer. Attempt history never selects a state; it only adds warnings.

| ticketctl answer | State |
|---|---|
| exit 0, plan with steps | `ready` |
| exit 0, plan with empty `steps` and a `dedupe_match` | `already_created` (key from the `dedupe_match` when it is a ticket key, else not known) |
| exit 20 `already_exists` | `already_created` (key is `details.existing_key`) |
| exit 22 or 10 (`missing_required_fields`, `invalid_value`, `body_too_large`, `empty_body`, `node_not_allowed`, `secret_in_body`, `unknown_field`, parent or board not found, ...), exit 2 `invalid_combination`, exit 21 `ambiguous_field` | `blocked` with ticketctl's reason |
| exit 21 `ambiguous_site`, `ambiguous_board`, `ambiguous_sprint` | not a screen: exit 21 with the candidates and the flag to pass |
| any other exit | not a screen: the Lower tool exits table |

Warnings added by history: `previous_attempt_unknown` when the state is `ready` and an attempt with this key has outcome `unknown` (ticketctl found no ticket, so it will create one with the same key; it also guards the label index lag on its side); `body_differs` when the state is `already_created` and the stored attempt's `body_sha256` differs from the current body (a body edit is not applied to an existing ticket); `existing_differs` when the state came from exit 20 `already_exists` (the existing ticket has another title or type than the draft).

| State | Prompt line | Accepted answers | Any other answer | Exit |
|---|---|---|---|---|
| `ready`, nothing cut | `Create this ticket? [y/N]` | `y`, `yes` write; `e` edit | decline | 20 `declined` |
| `ready`, text cut | `K item(s) are cut. v shows them in full, e edits, anything else cancels:` | `v` view; `e` edit (`y` is refused) | cancel | 20 `declined` |
| `already_created`, key known | `A ticket for this draft already exists (KEY). Edits to title, type, body, parent, labels, assignee and fields are not applied to it. Type accept to record it, anything else cancels:` | `accept` (`y` is not accepted) | cancel | 20 `declined` |
| `already_created`, key not known | `A ticket for this draft already exists but its key is not known. Anything else than e cancels:` | `e` edit (`accept` is not offered) | cancel | 20 `declined` |
| `blocked` | `Cannot create (see above). e edits the title and body; anything else stops:` | `e` | stop | 22 with ticketctl's reason |

In both `already_created` forms the screen shows, in the gutter zone, the existing ticket's `type` and `title`, read with `ticketctl get KEY --fields key,type,title,url` (a read that runs only in this state and only when the key is known). Ctrl-C at any prompt ends the call with exit 130 and changes nothing.

```text
def accept(row, state, ctx):                      # only `accept` in full (never y); the state is already_created
    key = state.existing_key or state.dedupe_match   # exit 20 details.existing_key, or the plan's dedupe_match
    if key does not match the ticket-key grammar:
        # the key is not known: the prompt did not offer accept (see table); this branch is never reached
        return
    t = ticketctl_get(key, site=S, fields="key,type,title,url")      # a read; exit 10 -> exit 12 lower_failed, nothing recorded
    one transaction on the lease token:
        created_key, created_url = t.key, t.url
        for a in row.attempts where a.generation == row.generation and a.outcome == "unknown": a.outcome = "reused"; a.ticket_key = t.key
        (under --again: generation, created_key, created_url and sprint are set as in T10 first)
        sprint.step = "pending" if a sprint was requested else sprint.step
        status = rest_status(row)
    write one audit record: phase outcome, action create, result.outcome "accepted", result.key = t.key (no `attempted` record: there was no write call)
    ctx.accepted = True                            # then step 7 runs when a sprint was chosen; post-create result is "reused"
```

`accept` is a read: it never calls a create. With several `unknown` attempts it resolves all of them in the same transaction.

## Sprint step

`--sprint open|next|ID` (id grammar `^[0-9]{1,18}$`) and `--board ID|NAME` are accepted on `create`; the selector, the resolved id and name, and the board are stored in the draft's `sprint`. Resolution uses ticketctl's rules, with `--site` and `--project` (and the stored board): `open` is the single `active` sprint of `ticketctl sprints list --state active` (none: exit 20 `no_sprint`; several: exit 21 with the candidates); `next` is the `future` sprint with the earliest non-null `start`, else the lowest id (none: exit 20 `no_sprint`); `next` needs every `future` sprint, so the `sprints list` call passes `--limit 500`; an id is read with `ticketctl sprints show ID` and a `closed` sprint exits 20 `sprint_closed`. `--board` takes an id or a name.

**After a create.** Once the ticket exists the step runs `ticketctl sprint add KEY --sprint ID --site S [--board B] --actor human:<user> --dry-run`, then with `--yes --expect-plan HASH2`; the human's yes on the create screen covers it because the sprint was on the screen. `unchanged` is success. `stale_ticket` is re-planned once. Any other answer (veto, escalation unanswered, rejected, closed sprint, unknown outcome) sets `sprint_failed`, keeps `created_key`, and exits 12 `partial` with the key and ticketctl's reason in `details`; an escalation asks the escalation question first. The `sprint.step` values are `none`, `pending`, `done` and `failed`. T11 sets `pending` in the same transaction as `created_key` when a sprint was requested; the step ends `done` or `failed`. A process that dies after T11 leaves `pending`, which `rest_status` reads as `sprint_failed`, and `create DRAFT_ID` then runs the sprint-only step.

**Sprint-only run.** `create DRAFT_ID --sprint S` on a `created` draft, or `create DRAFT_ID` on a `sprint_failed` draft (the stored sprint is used unless `--sprint` is given), runs only this step and needs a human. It accepts only `--sprint`, `--board`, `--site`, `--project` (equal to the stored value), `--dry-run` and the global flags; any other `create` flag exits 2 `invalid_combination`; `--board` without a sprint uses the stored sprint. It resolves the sprint, runs the `sprint add` dry-run, and shows a short screen drawn like the review screen: the facts hold the ticket key, the sprint id and the plan hash, and the gutter zone holds the URL, the sprint name, the board and ticketctl's warnings (for example `sprint_moved`), with the prompt `Add KEY to sprint? [y/N]`. A plan with empty `steps` (the ticket is already in the sprint) records `done` with no prompt and exits 0. Only `y` or `yes` runs `--yes --expect-plan`. Declining leaves the status unchanged (exit 20 `declined`). The sprint step is recorded in `sprint.step`, never as an attempt.

## Hooks

Hooks are configured in the tool's config (design guide §8). Each receives one JSON document on stdin and the environment `TICKET_CREATE_HOOK`, `TICKET_CREATE_ID` (the draft id) and `TICKET_CREATE_OBJECT_TYPE` (`draft`); they inherit the tool's environment minus the names in `credential_env`. Default timeout 30 seconds. `validate` warns when `credential_env` is empty.

Exit 13 escalates and any other non-zero exit vetoes (exit 11) for `pre-draft`; a timeout is a veto; `post-` hooks cannot undo. `async` is valid on `post-` hooks only. `event_id` is a fresh id (`ev-` plus 16 base32 characters) per hook invocation.

| Hook | Runs | Veto or escalation | Payload (beyond `tool`, `hook`, `object_type`, `object_id`) |
|---|---|---|---|
| `pre-draft` | before the session launches (Draft order step 2) | exit 11 on a veto, 13 on an escalation (a new draft's row is removed; a continued draft is left as it was) | `origin`, `site`, `project`, `type`, `mode`, `skill`, `has_seed`, `seed_bytes`, `continue` |
| `post-draft` | after every run that ends in `ready`, `failed` or `abandoned`, a failed `--continue` run included, and after an adopted run | none | `origin`, `status`, `failure`, `outcome`, `site`, `project`, `type`, `sizes` (`title`, `body`), `artifact_sha256`, `event_id` |
| `post-create` | after the create and sprint steps | none | `result` (`created`, `reused`, `partial`, `failed`, `unknown`), `site`, `project`, `key`, `sprint_step` (`none`, `done`, `failed`), `attempt` (`n`, `plan_sha256`; null for a sprint-only run and for `accept`), `event_id`, `dedupe_key` (the ticketctl key of the attempt, or null likewise) |

`post-create` does not run for a declined, blocked, `needs_human`, dry-run or error-before-the-write call. Payloads carry `reads_third_party: false` and `tainted: false`, ids, enums, counts, sizes and hashes, and never the title, body, notes or any model text. There is no `repo` or `labels` key (the draft has a repo; it is not part of the payload), so a gate on these hooks is scoped `*`. There is no `pre-create`: ticketctl's own `pre-create` hook runs after the human's confirmation and before the provider write, and a gate belongs there.

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
| 12 | `skill_not_found` | null | check the skill name |
| 12 | `unavailable` | `locked` for a held or lost lease, else null | retry later (`details.key` when a write call had returned: the ticket exists) |
| 12 | `lower_failed`, `outcome_unknown`, `partial`, `schema_newer` | null | `outcome_unknown`: run `create` again (same key); `partial`: the ticket exists, only the sprint step failed (`details.key`, `step`); `lower_failed`: `details.tool`, `lower_error` |
| 13 | `needs_human` | null | `create` or `discard` without a human, or the session asked for one |
| 13 | `escalated` | null | a hook asked for review: a ticketctl hook and the human declined the override, or a `pre-draft` or launch hook (nothing in ticket-create can answer those; `details.hook`) |
| 14 to 18 | `session_exited_unreported`, `session_killed`, `session_timeout`, `session_lost`, `session_failed` | null | the draft session's outcome |
| 20 | `refused` | `declined`, `draft_not_ready`, `draft_not_resumable`, `already_created`, `live`, `sprint_closed`, `no_sprint`, `attempt_recorded` | by reason |
| 21 | `ambiguous_draft`, `ambiguous` | null | a prefix (`details.candidates`), or a site, board or sprint |
| 22 | `artifact_missing`, `artifact_invalid`, `invalid_ticket_input`, `missing_required_fields` | the same word | the draft's content is missing or invalid; `invalid_ticket_input` is ticketctl's own validation on a dry-run (`details.lower_error`) |
| 129, 130, 143 | `usage` or none | null | SIGHUP, SIGINT or SIGTERM outside a session run (see Signals in Prompt and terminal rules); 130 is also Ctrl-C at a prompt (nothing changed) |

`details` keys are closed: `candidates`, `required_fields`, `existing_key`, `plan`, `attempt`, `lower_error`, `hook`, `hook_stderr`, `key`, `step`, `tool`, `minimum`, `found`, `install`, `config_key`. Warning codes are closed: `duplicate_candidates`, `body_near_limit`, `body_differs`, `existing_differs`, `previous_attempt_unknown`, `plan_differs`, `fixed_field_changed`, `assignee_account`, `form_facts_unavailable`, `credential_env_empty`, `mcp_inherited` and `skill_unchecked` (from `validate`, the kind check and the preflight), plus ticketctl's plan warning codes passed through; `body_near_limit` is ticketctl's (above 90 percent of its limit). Signals are described in Prompt and terminal rules.

## Lower tool exits

| Call | Exit | ticket-create does | Attempt and status |
|---|---|---|---|
| `ticketctl create --yes` | 0 | records the key, URL and `reused` | `created` or `reused`; status `created` |
| | 11 | shows the vetoing hook's reason (sanitized) and exits 11 | `failed`; `rest_status` |
| | 12 `outcome_unknown` | exits 12 with the attempt in `details` | stays `unknown` |
| | 12 other (`auth_required`, `bad_credentials`, `forbidden`, `rejected`, `unavailable`, `timeout`, `rate_limited`, `provider_changed`) | shows the code and message (sanitized), exits 12 `lower_failed` (`rejected` on a `labels` error says the site refuses the dedupe label and the ticket can only be made through ticketctl directly) | `failed`; `rest_status` |
| | 13 `escalated` | the escalation question | `failed` (a second attempt follows on yes) |
| | 13 `confirmation_required`, 2, 20 `declined` | cannot happen with `--yes` and a correct call: exits 1 `internal` (a ticket-create bug; nothing was written) | `failed` |
| | any other exit and code | shows code and message (sanitized), exits 12 `lower_failed` | `failed` |
| | 20 `already_exists` | the `already_created` state | `failed` |
| | 21 | exits 21 with the candidates | `failed` |
| | 22, 10 | the `blocked` state | `failed` |
| | 1, signal, unreadable output | exits 1 | stays `unknown` |
| `ticketctl create --dry-run`, `search`, `get`, `sprints list/show`, `sites` | 0 | used | none |
| | 20 `already_exists` (dry-run) | the `already_created` state | none |
| | 22, 10, 2 `invalid_combination`, 21 `ambiguous_field` on `create --dry-run` | the `blocked` state (the reason is the code) with the closed hint | none |
| | other 21 | exits 21 with the candidates | none |
| | 12 on a prefetch read or on `search` | a warning, the prefetch continues; on `create --dry-run` exit 12 `lower_failed` | none |
| | 10 on `get` of the parent in `draft` | exit 2 `invalid_argument` naming `--parent` | none |
| `ticketctl sprint add` | 0 (`unchanged` too) | step done | `sprint.step: done` |
| | 11, 12, 13 (unanswered), 20 `sprint_closed`, 2, 21 | step failed: `sprint_failed` in a full create (exit 12); in a sprint-only run the exit passes through | `sprint.step: failed` |
| | 20 `stale_ticket` | re-plan once, then as above | none |
| `agent-spawn launch`, `resume` | 0, 13 to 18 | the Session outcomes table (an exit 13 with `phase: launch` and no run is below) | none |
| | 22 | cannot happen (no `--wait-timeout`) | none |
| | any exit that produced no run | the pre-run error rule of the Draft order, mapped as the next table | none |

An error before any run produced a session (agent-spawn ending a launch or resume with no run, or the kind check) maps as follows.

| agent-spawn or kind-check result | ticket-create exit | `code` | `details` |
|---|---|---|---|
| exit 11 | 11 | `vetoed` | `hook`, `lower_error` |
| exit 13 with `phase: launch` or `escalated` | 13 | `escalated` | `hook`, `lower_error`; the message says the review is outside this tool (`agent-spawn show` names the owner) |
| exit 10, 12, 20, 21, any other | 12 | `lower_failed` | `tool: agent-spawn`, `lower_error` (carries agent-spawn's own reason text) |
| kind check exit 0 but a wrong kind, `reads_third_party`, `would_refuse`, warnings or `fallback` | 12 | `lower_failed` | `tool: agent-spawn`, `lower_error` naming the signal |

Every lower tool's JSON is read tolerantly (unknown keys are ignored, a missing required key is a failure). Provider messages, hook stderr and `lower_error` text are untrusted and go through the sanitizer before printing.

## Output by exit code and error envelope

With `--json`, exit 0 prints the command's result object; an exit that adopted a run or built a plan prints that one object with the error as its `error` member; every other failure prints the envelope. The envelope is `{"error": {"code", "message", "reason", "retryable", "outcome_unknown", "draft_id", "details"}}`. `reason` is non-null for exits 2, 20 and 22. `details` carries only the closed keys listed under Exit codes; `message`, `lower_error`, `hook_stderr` and provider text are untrusted.

`draft` and `create` results carry the draft (id, status, origin, site, project, type, key and URL when created, the attempts) and, for a session, `agent_id`, `phase`, `outcome`, `end_reason` and `first_working_at`.

## Output sanitizing

Every untrusted string that leaves the process (human output, `--json` values, hook payload strings, logs) passes one sanitizer. It replaces C0 controls except newline and tab, DEL, C1 controls, bidirectional and isolate controls, zero-width and other format characters, the tag-character block, surrogates, private-use and unassigned code points, U+2028 and U+2029, non-space separators and variation selectors with `<U+XXXX>` markers, and also U+034F, U+115F, U+1160, U+16FE4, U+17B4, U+17B5, U+180B to U+180F, U+2800, U+3164, U+FFA0 and U+FFFC (so shown text equals the text ticketctl sends). It caps runs of combining marks at 8 and, for non-screen output (hook and log text), truncates lines over 200 characters with `<...>`. Terminal output of model text follows the Review screen's layout rule instead of the truncation. The sanitizer is applied once, at the point of output.

## Objects

Draft: the stored fields of Draft record and statuses (without `run_token`), as printed by `show --json`, with `generation` and `sprint` (`selector`, `id`, `name`, `board`, `step`), `seed` (`type`, `parent`, `ticket_labels`, `has_seed`, `seed_bytes`) and `warnings` (`code`, `field`, `value`). Attempt: `n`, `generation`, `key`, `site`, `project`, `plan_sha256`, `body_sha256`, `started_at`, `outcome`, `attempt_id`, `ticketctl_attempt_id`, `ticket_key`. Both are defined in `schemas/defs.json`.

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

`projects` keys are `<site>/<KEY>` (a site name contains no `/`). `repos.<id>` may override `skill`, `templates`, `sandbox`, `setting_sources`, `env_allowlist`, `plugin_dirs`, `permission_profile`, `cwd` and `model`; it may not set `hooks`, `ticketctl.*`, `editor` or `credential_env`, because a repository-level setting must never select a program to run or a place to send credentials. A template path or plugin directory must be absolute and owned by the user. The template is delivered as an input file named `template.md`.

## Environment variables

Config keys take `TICKET_CREATE_<KEY>` (nested keys joined with `__`) for `mode`, `model`, `timeout`, `duplicates.enabled`, `duplicates.limit` and `retention.*` only; other keys are not read from the environment because the environment is often set by repository tooling and a repository-level value must never select a program or a place to send credentials. The tool reads `VISUAL`, `EDITOR`, `TERM`, `PATH`, `HOME` and `XDG_CONFIG_HOME` and `XDG_STATE_HOME`. It passes no credential of its own: ticketctl reads its own token variable, which must be listed in claude-session's `secret_env` so a draft session never receives it, and in `credential_env` so hooks and the editor do not inherit it.

## Labels

Agents get `owner=ticket-create`, `draft=<draft_id>` and, when there is a repo, `repo=<id>` (the repo id is claude-session's: `host/<path>`, see Draft order step 1). ticket-create finds an agent by its stored `agent_id`; labels are used only when none is stored (the process died before the launch result was read), and then only for exactly one agent of kind `draft-ticket` that is not torn down (`agent-spawn list --label owner=ticket-create --json`). `discard` and `prune` tear down only the agent found this way. Ticket labels (Jira labels) are separate: `--ticket-label` and the artifact's `labels`. They are never agent labels and never contain whitespace.

## Files

All under `~/.local/state/ticket-create/` (`XDG_STATE_HOME` respected), directories mode 0700, files 0600: `state.db` (private SQLite, WAL), `log.jsonl` and `audit.jsonl` (public contracts), `prompts/`, `results/`, `tmp/` (random names, deleted when the call ends; `prune` removes entries older than a day in all three). The state directory, `state.db`, `log.jsonl` and `audit.jsonl` are opened `O_NOFOLLOW` and checked to be owned by the caller.

## Audit and log

`audit.jsonl` has one `attempted` record just before each `ticketctl create --yes` and `sprint add --yes` call and an outcome record after it, paired by `attempt_id` (ticket-create's own), plus one `outcome` record per `draft` run and one per `accept` (no `attempted` record: there was no write call). Fields: `ts`, `actor`, `decision_id` (list), `phase` (`attempted`, `outcome`), `attempt_id`, `action` (`create`, `sprint_add`, `draft`), `target` (`<site>/<project>` or the draft id), `inputs` (`draft_id`, `dedupe_key`, `plan_sha256`, `body_sha256`, `origin`, `overrides` (the closed list of Draft record and statuses), `override_count`, `override_escalation` (null or `ticketctl`), `agent_id`), `result` (`key`, `reused`, `ticketctl_attempt_id`, `outcome`, `code`), `seq`. `result.outcome` is one of `attempted`, `created`, `reused`, `accepted`, `failed`, `unknown`, `done` (sprint), `ready`, `abandoned`. `log.jsonl` records each command with counts and hashes; its events are `command`, `hook_result`, `lower_call`, `session`, `draft_state`, `lease` and `warning`, each written when its thing happens (a command ends, a hook returns, a lower tool call ends, a session outcome is read, a status changes, a lease is acquired, released, taken over, lost or stalled, a warning is raised). Neither file holds the title, body, notes or any model text, and neither holds secrets.

## Concurrency

A per-draft lease is taken by `draft` (new and `--continue`), `create`, `discard` and `prune`, after the gate of Gate has run (a live `drafting` draft is refused with exit 20 before any lease is taken). A lease row holds the PID, the process start time, the expiry and a random `token` set at each acquisition. Its time to live is 60 seconds and a background thread of the holder renews it every 20 seconds, including while the process waits for a session or a prompt. Every transaction that changes a draft (T3, T5, T6, T9 to T18 of the Transitions table, `discard`, `prune`) is conditional on `token = mine`; a transaction that finds another token changes nothing and the call exits 12 `unavailable` (`locked`) before any ticketctl call. Renewal uses a monotonic clock: before the transaction of T10 and before a `sprint add --yes`, the holder refuses to continue (exit 12 `unavailable`) when its last successful renewal is older than the time to live. A lease is taken over when its holder (PID plus start time) is dead or its expiry passed by more than one time to live; a takeover logs `taken_over` and a failed renewal is logged as `stalled`. If the compare-and-set fails after a write call returned, ticket-create still writes the audit outcome record, prints the key and exits 12 `unavailable` naming the key in `details.key`. A contended lease is exit 12 `unavailable` (`locked`) for `create`, `--continue` and a draft in any status other than `drafting`; `discard` refuses a draft whose lease is held with exit 20 `live` (one reason for in use); `prune` skips it. Two `draft` runs for different drafts do not interact. `list` and `show` take no lease.

## Dependencies

| Tool | Binding | Minimum | Used for |
|---|---|---|---|
| ticketctl | A | `0.1.0` (its first release, which has `create`, `sprint add`, `sprints`, `search --text-file`) | the only writer: `create`, `sprint add`, and reads (`search`, `get`, `sprints`, `sites`, `validate`); its plan carries the final body as a Markdown string |
| agent-spawn | A | 0.34.0 (ticket-create's own pin: the first release with `launch --result-file`, `--no-implicit-done`, `--placement here` and `list --label`) | `launch`, `show`, `list`, `teardown`, `resume` |
| claude-session | none (through agent-spawn) | 0.45.0 (ticket-create's own pin: the version whose dry-run plan carries `tools` and which this interface was verified against; agent-spawn itself requires only the interface version 0.1.0, and 0.43.0 for `--implicit-done`) | its `secret_env` must name ticketctl's token variable; `notes_context` runs `memoctl context` |
| wtctl | none (agent-spawn requires it even with no worktree) | the version agent-spawn requires | installed for agent-spawn |

A missing sibling exits 12 `tool_missing` naming its install command; one below its pinned minimum exits 12 `tool_too_old`.

## Upgrades

The state database has a version and only additive migrations; a database newer than the tool exits 12 `schema_newer`. Config keys are added, never repurposed.

## Residual risks

- The draft session is a trusted, unsandboxed run (the `draft-ticket` kind is not third-party-reading). Text it is shown (seed, duplicate titles, a parent summary) is fenced but can steer its output and its tool use; its permission profile denies the dangerous tools and the human watches it, and a replaced profile or sandbox setting is the user's own choice. Text from a third party given as a headless seed can steer the session without a human watching.
- The `draft-ticket` kind inherits MCP servers and claude-session has no per-call switch, so a connector that writes tickets bypasses the screen unless the kind is configured `mcp: none` or the profile denies `mcp__*` (the bundled profile does); the kind check warns `mcp_inherited` while the kind's profile inherits them.
- A shell profile that exports the ticketctl token gives every shell command of the session the token; keep the token in a file sourced only by ticketctl.
- A hijacked session with one approved write can plant a shim on `PATH` or edit a template or plugin file.
- Nothing stops a process that can write to the terminal's input queue after the prompt is shown; the flush and the fixed layout cover input that arrived earlier.
- A process with the same user id can read `state.db`, the prompts and the drafts, and can edit the config.
- ticketctl's confirmation is not a boundary against a caller that can allocate a pty; ticket-create's boundary is that the draft session is not given the credential and is told not to run ticketctl.
- claude-session keeps the prompt as delivered, the staged inputs (seed text, template) and the frozen artifact for 30 days after the automatic teardown and after `forget`; `discard` prints `claude-session forget --purge ID`.
- `--idea TEXT` and inline `--set` values are visible in `ps`; use `--idea-file` for anything sensitive.
- An unattended headless draft spends the user's model tokens and its cost is not recorded (interactive runs report no `cost_usd`).
- A process of the same user can swap an input file between ticket-create's digest check and ticketctl's read; the digest check narrows that window, and ticketctl does not accept `/proc/self/fd` paths.

## Platform notes

Linux and WSL are tested; macOS is a local checklist (`/dev/tty`, termios restore, the editor). The interactive draft needs no tmux. Python 3.12 or later.
