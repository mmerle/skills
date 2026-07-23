---
name: tmux-operator
description: Operate and reuse tmux sessions, windows (tabs), panes, and terminal processes. Use when asked to inspect or capture tmux state, create or edit terminal workspaces, reuse or launch dev servers, run or monitor tests, watchers, or logs, interact with terminal applications, or launch visible workers such as OpenCode instances.
---

# tmux Operator

Use tmux as a shared execution and observation layer. Prefer visible, recoverable processes that the user can inspect or take over.

## Operating Contract

- Inspect before acting.
- Reuse a matching process in the project's existing tmux panes before creating another server, watcher, test process, log stream, or worker, unless the user explicitly requested a new pane, window, or process.
- Use the shortest safe path. When creating a pane or window for a known command, launch the command as part of creation instead of creating a shell and typing into it afterward.
- Keep user-visible terminal output clean. Do not print completion markers or agent bookkeeping into panes by default.
- Target stable tmux IDs such as `$1` for sessions, `@3` for windows, and `%48` for panes. Never rely on names or indexes because they may be duplicated or renumbered.
- Preserve the target pane's working directory unless the user provides another directory.
- Use executable names rather than interactive shell aliases.
- Never send keys to the current `$TMUX_PANE` while this turn is running.
- Verify the target command and cwd immediately before sending input.
- Do not capture unrelated panes speculatively. Pane output may contain sensitive data.
- Do not kill panes, windows, or sessions, respawn panes, send interrupts, detach clients, or replace running processes unless explicitly requested.
- Leave created tmux objects available for inspection when work completes.

Prefer an application's dedicated control CLI or API over simulated keystrokes. Use `tmux send-keys` when terminal input is the intended interface or no better control surface exists.

### Latency Rules

- For an explicit operation beside the current pane, query the current context once and act. Do not list panes first.
- Keep dependent shell capture, tmux mutations, and verification in one terminal tool call. Do not split them into separate agent turns.
- When all target IDs are already known, batch mutations and final metadata verification through one tmux command queue using `\;`.
- Set the new pane's percentage or exact size during `split-window` with `-p` or `-l`; do not create it at the default size and resize it afterward.
- Verify a known command launch with pane metadata. Capture terminal output only when readiness, failure text, interaction state, or command results matter.
- Do not poll a known CLI that has no requested readiness condition. One launch and one targeted verification are enough.

```bash
tmux resize-pane -t '%49' -R 5 \; swap-pane -d -s '%49' -t '%50' \; list-panes -t '@3' -F 'pane=#{pane_id} width=#{pane_width} height=#{pane_height}'
```

## Workspace Layout and Tabs

Tmux windows are tabs. For an explicit split, tab, resize, move, or swap request, inspect only the target window and mutate it directly; do not inventory the session or capture pane output unless the request requires process state. Preserve focus with `-d` by default. Change the visible pane or tab only when the user explicitly asks to focus it.

```bash
tmux list-panes -t '@3' -F 'window=#{window_id} pane=#{pane_id} active=#{pane_active} width=#{pane_width} height=#{pane_height} path=#{pane_current_path} command=#{pane_current_command}'
```

Create a tab or pane with its program atomically, retain the returned stable ID, and verify only the affected window's geometry:

```bash
window_and_pane=$(tmux new-window -d -P -F '#{window_id} #{pane_id}' -t '$1' -n 'monitor' -c '/absolute/cwd' btop)
pane_id=$(tmux split-window -h -d -p 40 -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' btop)
pane_id=$(tmux split-window -h -d -l 80 -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' btop)
tmux list-panes -t '@3' -F 'pane=#{pane_id} width=#{pane_width} height=#{pane_height}'
```

Use `-p` when the user describes a ratio and `-l` when a CLI needs an exact minimum width or height. With `-h`, `-l` is columns; with `-v`, it is rows.

Use the exact returned IDs for the requested mutation:

```bash
tmux rename-window -t '@3' 'monitor'
tmux resize-pane -t '%49' -x 80 -y 24
tmux swap-pane -d -s '%48' -t '%49'
tmux join-pane -d -h -s '%50' -t '%49'
tmux break-pane -d -P -F '#{window_id}' -s '%50'
tmux move-window -s '@4' -t '$1:2'
tmux select-window -t '@3'
tmux select-pane -t '%49'
tmux select-pane -T 'monitor' -t '%49'
tmux resize-pane -Z -t '%49'
tmux select-layout -t '@3' even-horizontal
tmux rotate-window -D -t '@3'
```

`join-pane`, `break-pane`, and `move-window` relocate existing work rather than restarting it. They may remove an empty source window. Do not use `kill-pane` or `kill-window` unless the user explicitly asks to close it.

## Adjacent Pane Fast Path

Use this path when the user explicitly asks to open a new split beside the current work and run or interact with a CLI there. The explicit request for a new split overrides process reuse. Do not inventory unrelated panes, inspect geometry, create an empty shell first, or search for duplicate processes unless the request itself requires one of those steps.

1. Inspect the current pane once using the standard context command below.
2. Preserve `pane_current_path`, keep focus on the current pane with `-d`, and default to a right split with `-h` unless the user requests another direction.
3. Launch the CLI directly as part of `split-window`. Apply a requested ratio or known minimum size in the same command. Pass the executable and its arguments separately; do not build shell source when direct argv is sufficient.
4. Record the returned pane ID, then verify only that pane's command and cwd. Include initial output only when the task depends on it.
5. Interact with the verified pane using literal text and a separate submit key.

For example, open OpenCode directly in a visible split without moving focus:

```bash
target=${TMUX_PANE:?not inside tmux}
cwd=$(tmux display-message -p -t "$target" '#{pane_current_path}')
pane_id=$(tmux split-window -h -d -P -F '#{pane_id}' -t "$target" -c "$cwd" opencode)
```

The same form works for any known executable and preserves argument boundaries:

```bash
pane_id=$(tmux split-window -h -d -P -F '#{pane_id}' -t "$target" -c "$cwd" lazygit)
pane_id=$(tmux split-window -h -d -P -F '#{pane_id}' -t "$target" -c "$cwd" opencode --prompt "$prompt")
pane_id=$(tmux split-window -h -d -l 80 -P -F '#{pane_id}' -t "$target" -c "$cwd" btop)
```

If a right split fails because the pane is too narrow, retry once with `-v`; do not precompute layout geometry. When a CLI's minimum size is already known, use `-l` during creation. If the CLI reports an unknown minimum after launch, inspect only the target window, resize the new pane to that minimum when its existing layout permits it, then recapture. For example, `btop` requires at least 80 columns and 24 rows with its default layout. Do not close, move, or replace another pane to satisfy a TUI's minimum size without an explicit request. Immediately verify the returned pane:

```bash
tmux display-message -p -t "$pane_id" 'pane=#{pane_id} command=#{pane_current_command} path=#{pane_current_path}' \; capture-pane -p -t "$pane_id" -S -40
```

For a known interactive application, wait only when the requested action depends on readiness and its ready state is not yet visible. Poll startup every 250 milliseconds for up to 10 seconds, stopping as soon as the expected command and application chrome appear. Do not add a fixed sleep when the first capture is already ready.

After readiness is verified, send literal input and its application-specific submit key in one tmux invocation, then take a best-effort snapshot:

```bash
tmux send-keys -t "$pane_id" -l -- "$text" \; send-keys -t "$pane_id" '<submit-key>' \; run-shell 'true' \; capture-pane -p -t "$pane_id" -S -80
```

The immediate snapshot may precede application output. Poll further only when the task has an explicit readiness or completion condition. Do not wait for an interactive application to exit merely to prove that input was accepted.

## Inspect tmux

For ordinary in-session operations, use one command to verify and record the current host, server socket, tmux IDs, application, and cwd:

```bash
tmux display-message -p -t "${TMUX_PANE:?not inside tmux}" 'host=#{host} socket=#{socket_path} session=#{session_id} name=#{session_name} window=#{window_id}:#{window_index} pane=#{pane_id}:#{pane_index} command=#{pane_current_command} path=#{pane_current_path}'
```

tmux IDs identify objects only within the reported server on the reported host. If this check fails, diagnose the environment without creating anything:

```bash
tmux -V
printenv TMUX
printenv TMUX_PANE
```

If `$TMUX_PANE` is absent, pane-local operations and client switching are unavailable. Continue only when the user explicitly requested server inventory, session creation, or attachment; otherwise stop with a clear error. Do not create a detached tmux server as an implicit workaround.

Expand only to the scope needed for the request. List one window's panes first; use session-wide or server-wide inventory only when the target cannot be resolved more narrowly:

```bash
tmux list-panes -t '@3' -F 'session=#{session_id} name=#{session_name} window=#{window_id}:#{window_index} pane=#{pane_id}:#{pane_index} active=#{pane_active} dead=#{pane_dead} command=#{pane_current_command} pid=#{pane_pid} path=#{pane_current_path} title=#{pane_title} tty=#{pane_tty}'
tmux list-panes -s -t '$1' -F 'session=#{session_id} name=#{session_name} window=#{window_id}:#{window_index} pane=#{pane_id}:#{pane_index} active=#{pane_active} dead=#{pane_dead} command=#{pane_current_command} pid=#{pane_pid} path=#{pane_current_path} title=#{pane_title} tty=#{pane_tty}'
tmux list-panes -a -F 'session=#{session_id} name=#{session_name} window=#{window_id}:#{window_index} pane=#{pane_id}:#{pane_index} active=#{pane_active} dead=#{pane_dead} command=#{pane_current_command} pid=#{pane_pid} path=#{pane_current_path} title=#{pane_title} tty=#{pane_tty}'
```

Inventory sessions and windows only for workspace-level operations:

```bash
tmux list-sessions -F 'session=#{session_id} name=#{session_name} attached=#{session_attached} windows=#{session_windows}'
tmux list-windows -a -F 'session=#{session_id} window=#{window_id}:#{window_index} name=#{window_name} active=#{window_active} panes=#{window_panes}'
```

List clients only when the request concerns attached clients or detaching:

```bash
tmux list-clients -F 'client=#{client_name} session=#{session_id} tty=#{client_tty}'
```

Use the exact stable ID returned by these commands for every later operation. Names are labels for the user, not control identifiers. If more than one session, window, pane, or client could match the request, ask the user to choose.

Treat `pane_current_command` as the primary application signal, but a shell value does not prove the pane is idle: a child TUI may still control the terminal beneath a wrapper shell. If the pane previously ran a TUI or its capture shows recognizable application chrome, inspect that pane's process tree using its tty before sending shell input. Never identify an application from the pane title alone.

## Capture Output

Capture bounded recent history from one exact pane:

```bash
tmux capture-pane -p -t '%48' -S -200
```

Increase the history range only when the requested output is missing. Do not default to the entire scrollback buffer.

Capture before interacting with an existing process when its current state matters or the new output would otherwise be ambiguous. Skip the pre-capture for a recently verified idle shell running a trivial command because the echoed command identifies the output boundary. A new pane that launches its command atomically also does not need an empty pre-capture. Summarize only content relevant to the request, preserving exact errors, paths, commands, and exit statuses when they matter.

## Create Windows and Panes

Resolve the source pane and cwd first. When the command is known, launch it atomically and pass its executable and arguments directly. This is the default for interactive applications and persistent processes:

```bash
tmux split-window -h -d -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' command_name arg1 arg2
```

Use `-h` for a pane to the right and `-v` for a pane below. tmux accepts `shell-command [argument ...]`; preserve argument boundaries instead of joining user input into shell source. An interactive or persistent process keeps the pane alive. A one-shot command exits with the pane unless the exact-status workflow below is used.

Capture the new pane ID from stdout, then take one immediate output snapshot. Monitor further only when the task has an explicit completion or readiness condition that is not yet visible.

Create an empty pane only when the user requested one or when later terminal interaction is itself the task:

```bash
tmux split-window -h -d -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd'
```

Reinspect an empty pane before sending input to it.

Use a separate window (tab) for independent or background work that does not need to remain visible beside the source pane:

```bash
tmux new-window -d -P -F '#{window_id} #{pane_id}' -t '$1' -n '<label>' -c '/absolute/cwd' command_name arg1 arg2
```

Capture both IDs from stdout, then reinspect the window and its initial pane. Use names only as human-readable labels.

For an explicit exact-status requirement, keep the finished pane as a dead pane and read `pane_dead_status` using the returned ID:

```bash
pane_id=$(tmux split-window -h -d -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' 'tmux set-option -p -t "$TMUX_PANE" remain-on-exit on; exec command_here')
tmux display-message -p -t "$pane_id" 'dead=#{pane_dead} status=#{pane_dead_status}'
```

Poll the metadata only until `dead=1`; status is blank while the command is still running.

## Manage Sessions and Clients

Create a session only when explicitly requested. Preserve the requested cwd and capture all returned IDs:

```bash
tmux new-session -d -P -F '#{session_id} #{window_id} #{pane_id}' -s '<name>' -c '/absolute/cwd'
```

Do not treat a detached session as an isolation boundary. It shares the host, filesystem, credentials, network, and other resources available to its processes.

When the user explicitly requests moving a client between sessions, use the operation appropriate to the current context:

```bash
# From inside tmux
tmux switch-client -t '$1'

# From a terminal outside tmux
tmux attach-session -t '$1'
```

Before detaching, identify the exact client with `list-clients`. Detach only that client and only when explicitly requested:

```bash
tmux detach-client -t '/dev/ttys001'
```

tmux preserves sessions, windows, panes, and running processes when a client disconnects. It does not preserve a process that exits, and it does not by itself restore state after the host or tmux server restarts.

## Send Input to Existing Panes

Reserve `send-keys` for an existing pane or an interactive application. Do not create an empty pane and use `send-keys` when the command can be supplied directly to `split-window` or `new-window`.

For a pane recently verified as an idle `zsh`, `bash`, `fish`, or `sh`, run a trivial command and take a best-effort snapshot in one tmux invocation. `run-shell 'true'` yields the command queue once without a fixed delay:

```bash
tmux send-keys -t '%57' -l -- 'pwd' \; send-keys -t '%57' Enter \; run-shell 'true' \; capture-pane -p -t '%57' -S -20
```

The snapshot is not completion detection. A returned prompt shows that the shell accepts input again, but background work may still be running; missing output is inconclusive. Reinspect first only when the pane was not recently verified, may have changed applications, or is not clearly idle. Quote paths and user-provided arguments defensively. Do not inject a command into a pane running an editor, TUI, server, test process, or unknown application.

## Monitor Processes

Before starting a persistent process, look for one already running in the target project unless the user explicitly requested a new pane, window, or separate process:

1. Search the current window, then the current session, expanding further only when the project may be elsewhere.
2. Match candidates by cwd and command or process tree; capture output only from likely candidates.
3. Reuse a matching healthy process as the observation and control surface, even when another user or agent created it.
4. Start another process only when no match exists, the existing process is unhealthy or configured differently, or the user explicitly requests a separate instance.

Do not infer identity from a listening port or pane title alone. Verify the pane cwd, process, and relevant readiness output.

Classify the process before waiting:

- **One-shot command:** Use an explicit completion condition, a dead pane with `pane_dead_status`, or an application-specific completion signal.
- **Server or watcher:** Wait for a specific readiness message, listening address, or healthy state; then leave it running.
- **Test watcher:** Capture the latest completed test cycle and distinguish it from the process continuing to watch.
- **Log stream:** Capture only the requested interval or event and leave the stream running.
- **Interactive application:** Use application-specific readiness and completion signals.

Capture immediately after launch. If the completion or readiness signal is absent, poll every 2-5 seconds unless the process has a naturally slower feedback cycle. Use a five-minute timeout unless the task implies another duration.

On timeout, report the current pane command and recent output. Do not interrupt the process automatically. If the pane exits, becomes dead, or changes cwd unexpectedly, stop and report the observed state.

## Interact With Terminal Applications

Before sending input to a TUI:

1. Verify the exact pane and application.
2. Capture its current state.
3. Determine how that application accepts text and submits actions.
4. Send literal text with `send-keys -l`.
5. Send control or submit keys separately.
6. Capture output to verify the action occurred.

Do not assume Enter submits across applications. Do not navigate or mutate an unfamiliar TUI by guessing keybindings.

Some TUIs coalesce repeated navigation keys that arrive within one terminal read. Prefer one key event followed by verification. When a short repeated sequence has already been shown to coalesce, keep it in one tmux command queue but separate key events with the smallest observed working interval, starting at 50 milliseconds. Do not add this pacing to ordinary text or single-key input.

When a dedicated skill or control CLI exists, use it instead:

- For Hunk, load `hunk-review` and use `hunk session *`; do not drive the Hunk TUI through tmux keystrokes.
- For other applications, prefer their documented session, RPC, socket, or remote-control interface when available.

## OpenCode Adapter

OpenCode may run beneath a wrapper shell, so `pane_current_command` can report `zsh` or another shell while the TUI remains active. Treat a previously known OpenCode pane or recognizable OpenCode TUI capture as OpenCode, then confirm with that pane's process tree when needed. Do not rely on the title alone.

Before launching another TUI, inventory existing panes unless the user explicitly requested a new split. Reuse an OpenCode pane only when its cwd matches the requested project and reusing its conversation is appropriate.

In a new pane or window, launch OpenCode directly as the creation command. When the first prompt is already known, pass it with `--prompt` so startup and submission are atomic:

```bash
tmux split-window -h -d -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' opencode
tmux split-window -h -d -P -F '#{pane_id}' -t '%48' -c '/absolute/cwd' opencode --prompt "$prompt"
```

Use `opencode run "$prompt"` instead when the task is intentionally one-shot and an ongoing TUI is unnecessary. Launch OpenCode in an existing pane only after a capture and process check establish that the pane is an idle shell with no child TUI:

```bash
tmux send-keys -t '%57' -l -- 'opencode'
tmux send-keys -t '%57' Enter
```

Poll pane metadata and output every 250 milliseconds for up to 10 seconds until either `pane_current_command` is `opencode`, or the pane's process tree and capture confirm the OpenCode TUI is ready beneath a shell wrapper. Stop on the first ready capture. If startup fails, report the pane output instead of retrying blindly.

OpenCode can be launched with explicit options when requested:

```bash
opencode --agent explore
opencode --session '<session-id>'
opencode --continue
```

Do not invent a session ID. Resolve sessions with `opencode session list --format json` when needed.

### Send a Prompt

This setup submits OpenCode input with `Ctrl+Y`. Return inserts a newline. Use the known submit binding directly; do not reread `tui.jsonc` on every prompt.

Send prompt text literally, then submit with a separate key command in the same tmux invocation:

```bash
tmux send-keys -t '%57' -l -- 'Prompt text' \; send-keys -t '%57' C-y
```

For multiline prompts, use OpenCode's live bracketed-paste mode so newline bytes are treated as pasted content rather than individual key events, then submit once with `C-y`:

```bash
payload=$(printf '\033[200~%s\033[201~' "$prompt")
tmux send-keys -t '%57' -l -- "$payload" \; send-keys -t '%57' C-y
```

Read the active `tui.json` or `tui.jsonc` only when `C-y` does not submit, the user says the binding changed, or the target instance is known to use different configuration. Prefer an explicit `OPENCODE_TUI_CONFIG`, then project TUI config, then global TUI config. Do not assume Enter submits.

### Wait for a Response

Capture the pane before submission. After submitting:

1. Confirm output changed or OpenCode entered its active state.
2. Continue polling while the TUI shows an active indicator such as `esc interrupt`.
3. If OpenCode requests approval, permission, or other user input, classify the worker as blocked and report the exact request. Do not approve it automatically.
4. Treat the response as complete when the active indicator disappears, the input area is ready, and output is stable across two captures.
5. On timeout, report that OpenCode remains active and include recent output without interrupting it.

## Visible Worker Delegation

Use a separate tmux pane or window when the user wants visibility, persistence across client disconnects, manual takeover, or an independent long-running process. Use a pane for tightly related side-by-side work and a window for an independent background worker or a separate layout. A worker may be an OpenCode instance, another terminal agent, a test runner, a server, or a purpose-specific CLI.

Before creating a worker, define its task, cwd and allowed scope, completion or readiness condition, verification, and expected handoff. Track its hostname, server socket, stable tmux IDs, application, purpose, and state. Check for duplicates and resource contention first.

Coding workers sharing the current checkout must be read-only or explicitly limited to non-overlapping files. Do not run simultaneous write-capable workers in the same files. This skill does not create Git worktrees.

Report blocked or stale workers without answering prompts or cleaning them up automatically. Do not allow recursive workers, commits, pushes, publishing, deployment, or other external actions unless explicitly requested.

When a worker finishes, capture its result, verify any claims in the parent context, and leave its tmux objects intact.

## Common Failures

- **Not inside tmux:** For pane-local operations, report the missing tmux context. Continue only for an explicitly requested server inventory, session creation, or attachment; do not silently create a server.
- **Unknown or wrong target:** Re-list tmux objects and stop when the stable ID, hostname, or server socket cannot be resolved exactly.
- **Target is the current pane:** Do not inject keys into the running process.
- **Wrong application:** Stop rather than sending shell input to a TUI or TUI input to a shell.
- **Operation did not complete:** Report the command, expected condition, process state, and recent output without interrupting or killing it.
