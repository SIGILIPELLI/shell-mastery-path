# 03 · Signal Handling

Level 1 introduced `trap ... EXIT` for cleanup. Long-running scripts —
servers, monitors, batch jobs — need to handle **signals** sent by the
operating system or another process: `Ctrl+C`, a `kill` command, a system
shutdown. This module covers catching and responding to those signals
properly.

## Common signals

| Signal | Number | Typical trigger | Default behavior |
|--------|--------|------------------|-------------------|
| `SIGINT` | 2 | `Ctrl+C` in the terminal | terminate |
| `SIGTERM` | 15 | `kill pid` (the default signal) | terminate |
| `SIGHUP` | 1 | terminal/session closed | terminate |
| `SIGKILL` | 9 | `kill -9 pid` | terminate, **cannot be caught or ignored** |
| `SIGQUIT` | 3 | `Ctrl+\` | terminate + core dump |
| `SIGSTOP`/`SIGCONT` | 19/18 | suspend/resume a process | pause / resume |

## Catching `SIGINT` for graceful shutdown

```bash
#!/usr/bin/env bash
set -euo pipefail

running=true

handle_sigint() {
    echo ""
    echo "Caught SIGINT — finishing current step, then exiting..."
    running=false
}
trap handle_sigint SIGINT

echo "Working... press Ctrl+C to stop gracefully."
count=0
while $running; do
    ((count++))
    echo "iteration $count"
    sleep 1
done

echo "Stopped cleanly after $count iterations."
```

Without a trap, `Ctrl+C` kills the script immediately, mid-operation —
possibly leaving a half-written file or an unreleased lock. Catching
`SIGINT` and flipping a flag that the main loop checks lets the current unit
of work finish before exiting.

## Catching multiple signals with one handler

```bash
#!/usr/bin/env bash

cleanup_and_exit() {
    local signal="$1"
    echo "Received $signal — cleaning up..."
    rm -f /tmp/myjob.lock
    exit 0
}

trap 'cleanup_and_exit SIGINT' SIGINT
trap 'cleanup_and_exit SIGTERM' SIGTERM

touch /tmp/myjob.lock
echo "Job running (PID $$). Send SIGINT or SIGTERM to stop."
while true; do sleep 1; done
```

## `trap` with multiple signal names on one line

```bash
trap 'echo "shutting down"; rm -f /tmp/myjob.lock; exit 0' SIGINT SIGTERM
```

A single `trap` call can list several signal names — the same handler
fires for whichever one is actually received.

## Ignoring a signal

```bash
trap '' SIGINT      # ignore Ctrl+C entirely — the script cannot be interrupted this way
```

Use this sparingly — for a critical section that truly must not be
interrupted (e.g. writing a database transaction file) — and restore normal
handling immediately afterward with `trap - SIGINT`.

## Resetting a trap

```bash
trap 'echo "ignoring Ctrl+C during critical section"' SIGINT
# ... critical section ...
trap - SIGINT      # restore default SIGINT behavior
```

## A robust long-running worker pattern

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/tmp/worker.lock"
shutdown_requested=false

on_shutdown_signal() {
    echo "Shutdown requested — will exit after this unit of work."
    shutdown_requested=true
}

on_exit() {
    echo "Removing lock file and exiting."
    rm -f "$LOCK_FILE"
}

trap on_shutdown_signal SIGINT SIGTERM
trap on_exit EXIT

if [[ -e "$LOCK_FILE" ]]; then
    echo "Already running." >&2
    exit 1
fi
touch "$LOCK_FILE"

while ! $shutdown_requested; do
    echo "processing a unit of work..."
    sleep 2
done

echo "graceful shutdown complete."
```

Combining a `SIGINT`/`SIGTERM` trap (to stop the work loop) with an `EXIT`
trap (to always release the lock, however the script ends — normal
completion, a caught signal, or an error) is the standard shape for a
production-quality long-running script.

## Testing signal handling

```bash
./worker.sh &
worker_pid=$!
sleep 3
kill -SIGTERM "$worker_pid"     # simulate an operator asking it to stop
wait "$worker_pid"
echo "worker exited with status $?"
```

## How It Actually Works

A Unix signal is an asynchronous notification the kernel delivers to a
process — not a message queued and read at the process's convenience, but
an interrupt that (for a running process) causes the kernel to pause normal
execution and invoke a registered handler, or apply a default action
(terminate, dump core, stop, ignore) if none is registered. `trap 'cleanup'
SIGINT` tells *bash itself* to register a handler; bash's internal signal
disposition table is what actually receives the interrupt from the kernel,
and bash then schedules your trap command to run — but only when bash is
between commands (or at points it explicitly checks), which is why a trap
can appear to be "delayed" until whatever foreground command bash is
currently blocked on (via `wait()`) returns or is itself interrupted.

`kill -l` enumerates signal numbers versus names because the kernel only
ever deals in small integers (`SIGINT` = 2, `SIGTERM` = 15, `SIGKILL` = 9 on
Linux/most Unix) — names are strictly a userspace convenience layer.

`trap ... EXIT` is special-cased inside bash: it isn't a real kernel signal
at all, it's a pseudo-signal bash fires from its own shutdown sequence
right before the process actually calls `_exit(2)`, guaranteeing it runs on
every exit path (normal fall-off, `exit`, or an actual fatal signal that
bash catches and translates into a shutdown), whereas `SIGKILL` cannot be
trapped by any process at all — the kernel deliberately never delivers it to
userspace signal-handling code, tearing the process down at the kernel
level instead.


## Cheat sheet

| Construct | Purpose |
|-----------|---------|
| `trap 'handler' SIGINT` | run `handler` when SIGINT (Ctrl+C) is received |
| `trap 'handler' SIGTERM` | run `handler` when SIGTERM (from `kill`) is received |
| `trap 'handler' SIGINT SIGTERM` | one handler for multiple signals |
| `trap '' SIGINT` | ignore a signal |
| `trap - SIGINT` | restore default handling |
| `trap 'handler' EXIT` | always run on exit, regardless of cause |
| `kill -SIGNAL pid` | send a named/numbered signal to a process |

## Exercise

Write `worker.sh` that runs an infinite loop printing a heartbeat message
every second, holds a `/tmp/worker.lock` file for its whole run, and
gracefully shuts down (removing the lock, printing a summary of how many
heartbeats it printed) on either `SIGINT` or `SIGTERM`. Start it in the
background, let it run for 5 seconds, then send it `SIGTERM` from another
terminal (or `kill`) and confirm it shuts down cleanly.
