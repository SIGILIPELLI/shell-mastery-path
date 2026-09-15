---
description: "Scheduling & Automation — Scripts are most useful when they run themselves. This module covers cron for recurring jobs and at for one-off scheduled jobs …"
---

# 07 · Scheduling & Automation

Scripts are most useful when they run themselves. This module covers
`cron` for recurring jobs and `at` for one-off scheduled jobs — the two
standard Unix ways to run a script "later" without a human triggering it.

## The crontab

```bash
crontab -l          # list the current user's cron jobs
crontab -e            # edit them (opens $EDITOR)
crontab -r             # remove ALL of the current user's cron jobs (careful!)
```

## Crontab syntax

```text
# minute hour day-of-month month day-of-week   command
#   0-59  0-23      1-31    1-12      0-6 (Sun=0)
```

```bash
# every day at 2:30 AM
30 2 * * * /home/user/scripts/backup.sh

# every 15 minutes
*/15 * * * * /home/user/scripts/healthcheck.sh

# every weekday (Mon-Fri) at 9am
0 9 * * 1-5 /home/user/scripts/report.sh

# at midnight on the 1st of every month
0 0 1 * * /home/user/scripts/monthly_cleanup.sh

# every Sunday at 3am
0 3 * * 0 /home/user/scripts/weekly_archive.sh
```

| Field | Range | Examples |
|-------|-------|----------|
| minute | 0-59 | `0`, `*/15`, `30` |
| hour | 0-23 | `2`, `9`, `*/6` |
| day of month | 1-31 | `1`, `15`, `*` |
| month | 1-12 | `6`, `*`, `1,7` |
| day of week | 0-6 (Sun=0) | `1-5`, `0`, `*` |

`*` means "every value," `*/N` means "every N units," and comma lists
(`1,15`) or ranges (`1-5`) can combine within a field.

## Writing cron-safe scripts

Cron jobs run with a minimal environment (no interactive `$PATH`, no
`.bashrc` loaded), so scripts that work fine in your terminal can fail
silently under cron. Defensive habits:

```bash
#!/usr/bin/env bash
set -euo pipefail

# don't rely on relative paths or an inherited PATH
export PATH="/usr/local/bin:/usr/bin:/bin"

cd "$(dirname "${BASH_SOURCE[0]}")"    # always work from the script's own directory

# use absolute paths everywhere
LOG_FILE="/var/log/myjob.log"
DATA_DIR="/home/user/data"
```

```bash
# always log — cron's own error output often gets emailed or silently dropped
0 2 * * * /home/user/scripts/backup.sh >> /home/user/logs/backup.log 2>&1
```

Redirecting both stdout and stderr to a log file (rather than leaving cron
to handle it) means you always have a record of what happened, and can
`tail -f` it while debugging a job that "isn't working."

## `at` — running a command once, at a specific time

```bash
at 10:00 PM <<< "/home/user/scripts/cleanup.sh"
at now + 30 minutes <<< "echo reminder: check the deploy"
at 2026-07-20 09:00 <<< "/home/user/scripts/monthly_report.sh"

atq              # list pending `at` jobs
atrm 3            # cancel job number 3 (from `atq`'s output)
```

`cron` is for **recurring** schedules; `at` is for a **single** future run —
useful for "run this once, tonight, after traffic dies down" tasks that
don't belong in the crontab permanently.

## Checking whether cron actually ran your job

```bash
grep CRON /var/log/syslog | tail -20        # Debian/Ubuntu
grep cron /var/log/system.log | tail -20     # macOS
crontab -l | grep backup.sh                    # confirm the job is actually installed
```

## A locking pattern to avoid overlapping runs

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/tmp/myjob.lock"

if [[ -e "$LOCK_FILE" ]]; then
    echo "already running (lock file exists) — exiting" >&2
    exit 1
fi

trap 'rm -f "$LOCK_FILE"' EXIT
touch "$LOCK_FILE"

# ... the actual job ...
echo "doing scheduled work..."
```

If a job sometimes takes longer than its own interval (a 5-minute cron job
that occasionally takes 6 minutes), this lock-file pattern prevents two
copies from running at once and stepping on each other.

## How It Actually Works

`cron` is a long-running **daemon** process, entirely separate from any
interactive shell — it wakes up roughly once a minute, re-reads crontab
files if their mtimes changed, and for any job whose schedule matches the
current time, it `fork()`s and `execve()`s a *new*, minimal, non-interactive
shell to run that one command line. That new shell inherits almost none of
your interactive environment: no `.bashrc`, a bare-bones `$PATH`, and no
`$DISPLAY` or other session variables — which is the actual root cause of
"works when I run it myself, fails under cron," not a cron bug.

A file lock pattern using `mkdir` (or `flock`) for "avoid overlapping runs"
works because `mkdir(2)` is atomic at the kernel/filesystem level — the
kernel guarantees that if two processes race to create the same directory
name simultaneously, exactly one `mkdir` call succeeds and the other gets
`EEXIST`, with no possibility of both seeing "doesn't exist yet." `flock(2)`
achieves the same guarantee more directly via an advisory lock the kernel
tracks per open file descriptor, released automatically if the holding
process dies — which is why `flock`-based locks self-heal after a crash
while a bare `mkdir` lock file can be left stale and needs manual or
timestamp-based cleanup.


## Cheat sheet

| Command | Purpose |
|---------|---------|
| `crontab -e` | edit the current user's crontab |
| `crontab -l` | list current cron jobs |
| `* * * * *` | minute hour day-of-month month day-of-week |
| `*/N` | every N units of that field |
| `at TIME` | schedule a one-off command |
| `atq` / `atrm N` | list / cancel pending `at` jobs |
| `>> log 2>&1` | always redirect cron job output to a log file |

## Exercise

Write a crontab line that runs a script called `disk_check.sh` every 6
hours, logging output to `/var/log/disk_check.log`. Then write
`disk_check.sh` itself using the lock-file pattern above, so it exits
immediately (with a clear message) if a previous run is still in progress.
