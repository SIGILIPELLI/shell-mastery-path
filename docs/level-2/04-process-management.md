# 04 · Process Management

Every command you run becomes a **process**. Longer-running scripts need to
know how to inspect running processes, run things in the background, bring
them back to the foreground, and stop them cleanly.

## Foreground vs background with `&`

```bash
sleep 30              # runs in the FOREGROUND — blocks your shell for 30s

sleep 30 &            # runs in the BACKGROUND — shell prompt returns immediately
echo "still going, PID: $!"    # $! is the PID of the last backgrounded command
```

```bash
long_running_task.sh &
task_pid=$!
echo "started task with PID $task_pid"

# ... do other things ...

wait "$task_pid"        # block until that specific background job finishes
echo "task finished"
```

## `jobs` — listing background jobs in the current shell

```bash
sleep 100 &
sleep 200 &

jobs
# [1]-  Running    sleep 100 &
# [2]+  Running    sleep 200 &
```

```bash
fg %1        # bring job 1 back to the foreground
bg %2        # resume job 2 in the background (if it was stopped)
```

`Ctrl+Z` suspends the current foreground job (sends `SIGTSTP`); `bg` then
resumes it in the background, `fg` resumes it in the foreground.

## `ps` — inspecting processes system-wide

```bash
ps aux                   # every process, for every user, in "BSD" style
ps aux | grep "nginx"      # find nginx processes
ps -ef                    # every process, "standard" style with parent PIDs
ps -o pid,ppid,cmd -p $$    # show pid/ppid/command for the CURRENT shell ($$)
```

| Column | Meaning |
|--------|---------|
| `PID` | process ID |
| `PPID` | parent process ID |
| `%CPU` / `%MEM` | resource usage |
| `STAT` | process state (R=running, S=sleeping, Z=zombie, T=stopped) |
| `CMD`/`COMMAND` | the command line that started it |

## `kill` — sending signals to processes

```bash
kill 12345           # sends SIGTERM (15) — ask the process to shut down gracefully
kill -9 12345          # sends SIGKILL (9) — force-kill, cannot be caught or ignored
kill -l                 # list all signal names/numbers

pkill -f "backup.sh"     # kill by matching the command line, no PID needed
killall node               # kill all processes literally named "node"
```

`SIGTERM` (the default) gives a process the chance to clean up (close files,
flush buffers) before exiting; `SIGKILL` is an unconditional, immediate stop
— reach for `-9` only when a process refuses to respond to a normal `kill`.
Level 3's "Signal Handling" module covers *catching* these signals inside
your own scripts with `trap`.

## Finding and managing processes by name

```bash
pgrep -f "python.*server.py"     # list PIDs matching a pattern
pgrep -f "python.*server.py" | xargs kill    # find and kill in one line

if pgrep -f "myscript.sh" > /dev/null; then
    echo "myscript.sh is already running — exiting"
    exit 1
fi
```

That last pattern — checking `pgrep` before starting — is a common way to
prevent the same script (a backup job, a monitor) from running twice at
once.

## Waiting on multiple background jobs

```bash
for url in "site1.com" "site2.com" "site3.com"; do
    curl -s -o "/tmp/$url.html" "https://$url" &
done

wait     # blocks until ALL background jobs started in this shell finish
echo "all downloads complete"
```

This pattern — fire off several jobs in parallel, then `wait` for all of
them — is the simplest form of shell-level concurrency (Level 3's
"Parallelism" module builds on this with job limits and `xargs -P`).

## `nohup` and `disown` — surviving shell exit

```bash
nohup long_task.sh > output.log 2>&1 &     # keeps running even after you log out
disown                                        # remove the job from THIS shell's job table
```

Without `nohup`, a background job normally receives `SIGHUP` and dies when
its parent terminal closes — `nohup` (and redirecting its I/O) is the
standard way to launch something that should outlive your session.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `cmd &` | run `cmd` in the background |
| `$!` | PID of the last backgrounded command |
| `jobs` | list background jobs of the current shell |
| `fg` / `bg` | resume a job in the foreground / background |
| `wait [pid]` | block until a background job (or all of them) finishes |
| `ps aux` | list all running processes |
| `kill [-9] pid` | send SIGTERM (or SIGKILL with `-9`) to a process |
| `pgrep -f pattern` | find PIDs by command-line pattern |
| `pkill -f pattern` | kill processes matching a pattern |
| `nohup cmd &` | run `cmd` immune to hangup when the shell exits |

## Exercise

Write `parallel_ping.sh` that takes a list of hostnames as arguments, pings
each one in the background (`ping -c 1 host &`), captures each PID, waits
for all of them with `wait`, and then prints a summary of which hosts
responded successfully and which didn't (using each background job's exit
status).
