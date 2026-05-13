# Initial design: persistent logged task terminals

## Purpose

`terminals` provides a disciplined way for an AI assistant to do shell work in a way that a human collaborator can observe and audit.

The intended user story is:

1. A task begins.
2. The assistant starts a named persistent tmux session for that task.
3. All terminal commands for the task run inside that one session.
4. The user can attach to the session at any time and watch interactively.
5. The entire pane output is logged continuously.
6. When the task is complete, the final scrollback and metadata are written for review.

This improves over isolated one-shot terminal calls by giving the task a single visible shell timeline.

## Non-goals for the first version

- Replace Hermes tool calls entirely.
- Provide a full terminal multiplexer abstraction.
- Guarantee cryptographic non-repudiation of logs.
- Prevent a deliberately malicious process from evading logging.
- Solve secret redaction perfectly.

The first version should be a practical observability layer, not a security boundary.

## Why tmux, not dtach

`dtach` is attractive because it is tiny, but this project needs more than attach/detach:

| Requirement | tmux | dtach |
| --- | --- | --- |
| Named sessions | Yes | Socket path only |
| Live attach/detach | Yes | Yes |
| Scrollback capture | Yes | No native equivalent |
| Continuous pane logging | Yes, via `pipe-pane` | External wrapper required |
| Multiple panes/windows later | Yes | No |
| Introspection/listing | Good | Limited |

The default backend should therefore be tmux. A future dtach backend is possible if simplicity becomes more important than scrollback/introspection.

## Core concept

A **task terminal** is a tuple of:

- tmux session name;
- working directory;
- task slug/id;
- log directory;
- metadata file;
- continuously appended pane log;
- optional structured command/event log;
- final scrollback capture.

Example:

```text
~/.hermes/task-terminals/
  2026-05-13/
    20260513-081203-gmail-watcher-refactor/
      metadata.json
      session.log
      events.jsonl
      final-capture.txt
```

## Session naming

Session names should be deterministic enough for humans but unique enough to avoid collisions:

```text
hermes-task-YYYYMMDD-HHMMSS-<slug>
```

Example:

```text
hermes-task-20260513-081203-gmail-watcher-refactor
```

`latest` should resolve to the most recently started known task terminal.

## Directory layout

Default root:

```text
~/.hermes/task-terminals
```

Per-task directory:

```text
~/.hermes/task-terminals/<YYYY-MM-DD>/<task-id>/
```

Files:

- `metadata.json` — durable task/session metadata.
- `session.log` — raw appended terminal transcript from `tmux pipe-pane`.
- `events.jsonl` — structured lifecycle events and, if feasible, command events.
- `final-capture.txt` — final `tmux capture-pane` output when the task is finished.

## Metadata schema draft

```json
{
  "schema_version": 1,
  "task_id": "20260513-081203-gmail-watcher-refactor",
  "slug": "gmail-watcher-refactor",
  "description": "gmail watcher refactor",
  "tmux_session": "hermes-task-20260513-081203-gmail-watcher-refactor",
  "tmux_window": 0,
  "tmux_pane": 0,
  "cwd": "/Users/emrys/repo",
  "log_dir": "/Users/emrys/.hermes/task-terminals/2026-05-13/20260513-081203-gmail-watcher-refactor",
  "session_log": "session.log",
  "events_log": "events.jsonl",
  "final_capture": null,
  "started_at": "2026-05-13T08:12:03-04:00",
  "ended_at": null,
  "status": "running"
}
```

Statuses:

- `running`
- `finished`
- `abandoned`
- `failed`

## Prompt timestamping

The design should not inject explicit `printf` before and after each command. Instead, each managed shell should be configured so that the prompt itself records time and exit status.

For bash, the starter can create a temporary rcfile and launch:

```bash
bash --rcfile ~/.hermes/task-terminals/<...>/bashrc
```

That rcfile can define `PROMPT_COMMAND` and `PS1` so the log naturally contains timestamped prompt boundaries.

Example concept:

```bash
__terminals_prompt_command() {
  local status=$?
  printf '\n[terminals] %s exit=%s cwd=%q\n' "$(date -Is)" "$status" "$PWD"
}
PROMPT_COMMAND=__terminals_prompt_command
PS1='\u@\h:\w\$ '
```

This makes the transcript easier to read while keeping command execution natural and interactive.

For zsh/fish, equivalent hooks can be added later.

## Continuous logging

After creating the session, enable pane logging:

```bash
tmux pipe-pane -o -t "$SESSION" "tee -a '$LOG_DIR/session.log'"
```

Important details:

- Use `pipe-pane -o` to avoid accidentally double-piping.
- Quote paths safely.
- Prefer append-only logging.
- Record a `logging_started` event in `events.jsonl`.

## Running commands

The wrapper should send commands into the task pane:

```bash
terminals run <task> "git status"
```

`run` is synchronous: it returns only after the submitted command has completed and the managed shell is ready for the next command.

Implementation sketch:

```bash
tmux send-keys -t "$SESSION" -- "$COMMAND" Enter
tmux send-keys -t "$SESSION" -- "tmux wait-for -S $CHANNEL" Enter
tmux wait-for "$CHANNEL"
```

`run` should append a structured event before sending:

```json
{"type":"command_sent","at":"...","command":"git status"}
```

The terminal transcript remains the source of truth for output and eventual prompt/exit markers.

Because command events are written before the command is issued, they intentionally do not include exit codes. Exit status belongs in the prompt timestamp markers captured in `session.log`, not in `events.jsonl`.

## Attaching live

The user-facing attach command should be simple:

```bash
terminals attach latest
```

or directly:

```bash
tmux attach -t hermes-task-20260513-081203-gmail-watcher-refactor
```

The tool should print both forms after `start`.

Detaching remains standard tmux:

```text
Ctrl-b d
```

## Finishing a task

`finish` should:

1. Capture all available scrollback:

   ```bash
   tmux capture-pane -t "$SESSION" -p -S - > "$LOG_DIR/final-capture.txt"
   ```

2. Mark metadata with `ended_at`, `status`, and `final_capture`.
3. Kill the tmux session immediately.

Default behavior: completed tmux sessions are cleaned up immediately by `finish`. The durable artifacts are the logs and metadata, not a completed tmux process.

## Retention and cleanup

Initial behavior:

- logs persist indefinitely;
- finished tmux sessions are killed immediately by `finish`;
- `terminals list` shows running and finished known tasks;
- `terminals cleanup --older-than 7d --kill-finished` can be added later.

## CLI draft

```bash
terminals start DESCRIPTION [--cwd DIR] [--slug SLUG]
terminals run TASK COMMAND
terminals attach TASK
terminals finish TASK [--kill]
terminals list [--json]
terminals show TASK [--json]
terminals capture TASK
terminals cleanup [--older-than DURATION] [--kill-finished]
```

Where `TASK` can be:

- exact task id;
- tmux session name;
- slug if unique;
- `latest`.

## Configuration draft

Possible config file:

```text
~/.config/terminals/config.toml
```

Possible settings:

```toml
root = "~/.hermes/task-terminals"
history_limit = 200000
keep_finished_sessions = false
default_shell = "bash"
```

For the Hermes/Emrys use case, defaulting to `~/.hermes/task-terminals` is appropriate.

## Safety and privacy

This system increases observability, but it also records everything printed in the task terminal. Therefore:

- do not run commands that print secrets unless output is filtered;
- avoid `cat ~/.hermes/.env`, raw OAuth files, tokens, API keys, and cookies;
- consider future opt-in redaction filters for `session.log`;
- record in docs that logs are sensitive artifacts.

Secret redaction should not be trusted as a complete defense; command discipline remains necessary.

## Implementation approach

A small Python CLI is the first implementation:

- portable enough across macOS/Linux;
- robust JSON metadata handling;
- safe subprocess argument passing;
- easy testing;
- easy future packaging with Nix.

No daemon is needed for version 1. The system can be entirely command-driven around tmux.

## Milestones

### Milestone 1: design repository

- README.
- Initial design document.
- GPLv3 license.

### Milestone 2: minimal CLI

- `start` creates tmux session, metadata, log dir, and `pipe-pane` logging.
- `attach` attaches to session.
- `list` reads metadata.
- `finish` captures pane, updates metadata, and kills the tmux session.

### Milestone 3: command sending and prompt hooks

- `run` sends commands.
- bash rcfile with timestamp/exit prompt markers.
- events JSONL.

### Milestone 4: Hermes workflow integration

- Document when Emrys should use task terminals.
- Possibly create a Hermes skill for the workflow.
- Consider wrapping substantial terminal work by default.

## Resolved initial decisions

- Package/command name: `terminals`.
- Implementation language: Python.
- `run` behavior: synchronous.
- `run` wait mechanism: explicit tmux `wait-for` sentinel command after the user command.
- Completed tmux sessions: killed immediately by `finish` after final capture.
- JSONL command events: written before command issuance; include command text and timing, but no exit code.

## Open questions

- Should log redaction be opt-in, always-on for known secret patterns, or left to command discipline?
- What Nix packaging shape should be used first: flake app, package, Home Manager module, or all three?
