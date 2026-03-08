---
name: pi-tmux-manager
description: Manage a separate pi instance inside tmux (start, send prompts, capture output, stop) instead of spawning a subagent.
allowed-tools: Bash(tmux:*), Bash(pi:*), Bash(pgrep:*), Bash(pkill:*), Bash(printf:*), Bash(mkdir:*), Bash(date:*), Bash(tail:*)
---

# pi tmux manager

Use this skill when the user wants **no subagent spawning**, but still wants a separate controllable pi worker process.

## Rules

- **Do not spawn a subagent.**
- Run pi as a normal process inside a **tmux session**.
- Manage it only through tmux commands (`new-session`, `send-keys`, `capture-pane`, `list-sessions`, `kill-session`).
- Prefer detached sessions (`tmux new-session -d`) so the main agent remains responsive.

## Session conventions

Use predictable names:

- Session: `pi-worker`
- Window: `main`
- Pane: `0`

Reference target: `pi-worker:main.0`

If multiple workers are needed, suffix with a number, e.g. `pi-worker-2`.

## Start pi in tmux

```bash
# create session only if missing
if ! tmux has-session -t pi-worker 2>/dev/null; then
  tmux new-session -d -s pi-worker -n main
  tmux send-keys -t pi-worker:main.0 "pi" C-m
fi
```

## Send a prompt/command to the tmux pi instance

Use `send-keys` with Enter:

```bash
tmux send-keys -t pi-worker:main.0 "Summarize the TODOs in ./README.md" C-m
```

For multi-line prompts, send a here-doc safely:

```bash
tmux send-keys -t pi-worker:main.0 "cat <<'EOF'" C-m
tmux send-keys -t pi-worker:main.0 "Analyze src/app.ts and propose a refactor plan." C-m
tmux send-keys -t pi-worker:main.0 "EOF" C-m
```

## Read output from the tmux pi instance

Capture recent pane history:

```bash
tmux capture-pane -t pi-worker:main.0 -p -S -200
```

Increase history as needed (`-S -1000`, etc.).

## Health checks

```bash
# list sessions
tmux list-sessions

# verify target exists
tmux list-panes -t pi-worker:main -F '#{session_name}:#{window_name}.#{pane_index} #{pane_current_command}'
```

## Stop/restart

```bash
# graceful: send Ctrl-C then optional exit
tmux send-keys -t pi-worker:main.0 C-c

tmux send-keys -t pi-worker:main.0 "exit" C-m

# hard stop session
tmux kill-session -t pi-worker
```

Restart by running the start block again.

## Suggested workflow

1. Ensure `pi-worker` tmux session exists and pi is running.
2. Send one clearly scoped instruction.
3. Capture pane output and parse response.
4. Send follow-up prompts as needed.
5. Stop session when finished (or keep alive for iterative work).

## Safety & reliability

- Check `tmux has-session -t <name>` before creating/killing sessions.
- Avoid duplicate sessions with same name.
- Keep prompts short and explicit to reduce parsing ambiguity in captured output.
- If output is noisy, send a delimiter request, e.g. "Respond between BEGIN/END markers".

## Example end-to-end

```bash
if ! tmux has-session -t pi-worker 2>/dev/null; then
  tmux new-session -d -s pi-worker -n main
  tmux send-keys -t pi-worker:main.0 "pi" C-m
fi

tmux send-keys -t pi-worker:main.0 "Respond with only: READY" C-m
sleep 1
tmux capture-pane -t pi-worker:main.0 -p -S -80

tmux send-keys -t pi-worker:main.0 "List 3 improvements for src/server.js" C-m
sleep 2
tmux capture-pane -t pi-worker:main.0 -p -S -200
```

Use this pattern whenever the user asks for a managed secondary pi process without subagents.
