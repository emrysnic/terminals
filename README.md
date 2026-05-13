# terminals

`terminals` is a small tmux-based task terminal manager for Hermes/Emrys workflows.

The goal is to make terminal work observable and auditable:

- one persistent tmux session per substantial task;
- one continuous transcript per task;
- a stable command that Alex can use to attach and watch work live;
- final captured logs for after-action review;
- simple metadata so task terminal history can be searched and cleaned up later.

This project starts as a small Python CLI around tmux. It creates, logs, attaches to, runs commands in, and finalizes task-scoped terminal sessions.

## Why

Hermes tool calls are useful, but ordinary one-shot shell calls are hard to observe while they are happening and can scatter task evidence across separate invocations. For debugging, installs, deployments, benchmarks, and code work, Alex should be able to attach to the live terminal and later review the entire transcript.

## Design summary

For each substantial task, `terminals` creates a named tmux session and a log directory under:

```text
~/.hermes/task-terminals/<date>/<task-id>/
```

The tmux pane is continuously logged with `tmux pipe-pane`. The shell prompt is configured to include timestamps using `PROMPT_COMMAND`/prompt hooks rather than injecting explicit `printf` wrappers around every command. At task completion, the tool captures tmux scrollback and writes final metadata.

See [`INITIAL_DESIGN.md`](INITIAL_DESIGN.md) for the detailed design.

## Initial CLI sketch

```bash
terminals start "gmail watcher refactor" --cwd ~/repo
terminals run latest "git status"
terminals attach latest
terminals finish latest
terminals list
```

`run` is synchronous. `finish` captures final scrollback and kills the tmux session immediately; logs remain on disk.

The implementation should prefer boring, inspectable shell/Python over cleverness.

## License

GPL-3.0-or-later. See [`LICENSE`](LICENSE).
