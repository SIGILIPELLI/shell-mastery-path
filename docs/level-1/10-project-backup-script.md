# 10 · Project — Backup Script

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/DETp_6LuBlI" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


A small end-to-end project combining everything from Level 1: variables,
control flow, loops, functions, file tests, redirection, text processing,
and error handling.

## What you'll build

A script `backup.sh` that:

- Takes a source directory as an argument
- Archives it into a timestamped `.tar.gz` file
- Logs every step (with timestamps) to a log file, while also printing to
  the screen
- Validates its inputs and fails loudly with clear error messages
- Reports the final archive size and exits with a meaningful status code

## Project layout

```text
backup_project/
    backup.sh
    backups/        (created by the script)
    backup.log       (created by the script)
```

## backup.sh

```bash
#!/usr/bin/env bash
# backup.sh — archive a directory with logging and error handling
set -euo pipefail

# ---- configuration -------------------------------------------------------
BACKUP_DIR="./backups"
LOG_FILE="./backup.log"

# ---- helpers --------------------------------------------------------------
log() {
    # prints to the screen AND appends to the log file, with a timestamp
    local message="$1"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] $message" | tee -a "$LOG_FILE"
}

die() {
    log "ERROR: $1"
    exit "${2:-1}"
}

usage() {
    echo "Usage: $0 <source_directory>" >&2
    exit 1
}

cleanup() {
    log "backup script finished (exit code $?)"
}
trap cleanup EXIT

# ---- validate arguments ----------------------------------------------------
if [[ $# -ne 1 ]]; then
    usage
fi

source_dir="$1"

if [[ ! -d "$source_dir" ]]; then
    die "source directory '$source_dir' does not exist"
fi

# ---- prepare -----------------------------------------------------------------
mkdir -p "$BACKUP_DIR"

base_name=$(basename "$source_dir")
timestamp=$(date +%Y%m%d_%H%M%S)
archive_name="${base_name}_${timestamp}.tar.gz"
archive_path="$BACKUP_DIR/$archive_name"

log "starting backup of '$source_dir'"

# ---- do the backup --------------------------------------------------------
if tar -czf "$archive_path" -C "$(dirname "$source_dir")" "$base_name"; then
    log "archive created: $archive_path"
else
    die "tar failed while archiving '$source_dir'"
fi

# ---- report ------------------------------------------------------------------
if [[ -s "$archive_path" ]]; then
    size=$(du -h "$archive_path" | cut -f1)
    log "backup succeeded — size: $size"
else
    die "archive was created but is empty — something went wrong"
fi

# ---- retention: keep only the 5 most recent backups for this source ------------
log "cleaning up old backups (keeping the 5 most recent for '$base_name')"
# shellcheck disable=SC2012
old_backups=$(ls -1t "${BACKUP_DIR}/${base_name}"_*.tar.gz 2>/dev/null | tail -n +6)

if [[ -n "$old_backups" ]]; then
    while IFS= read -r old_file; do
        rm -f "$old_file"
        log "removed old backup: $old_file"
    done <<< "$old_backups"
else
    log "no old backups to remove"
fi

log "all done."
exit 0
```

## Running it

```bash
chmod +x backup.sh

./backup.sh ./my_project
# [2026-07-18 10:00:00] starting backup of './my_project'
# [2026-07-18 10:00:00] archive created: ./backups/my_project_20260718_100000.tar.gz
# [2026-07-18 10:00:00] backup succeeded — size: 128K
# [2026-07-18 10:00:00] cleaning up old backups (keeping the 5 most recent for 'my_project')
# [2026-07-18 10:00:00] no old backups to remove
# [2026-07-18 10:00:00] all done.
# [2026-07-18 10:00:00] backup script finished (exit code 0)
```

```bash
./backup.sh ./does_not_exist
# [2026-07-18 10:01:00] ERROR: source directory './does_not_exist' does not exist
# [2026-07-18 10:01:00] backup script finished (exit code 1)
```

## How each Level 1 module shows up here

| Module | Where it's used |
|--------|-------------------|
| 02 Variables | `BACKUP_DIR`, `LOG_FILE`, `archive_name`, quoting throughout |
| 03 Control flow | `if [[ ! -d ... ]]`, `if tar ...; then ... else ... fi` |
| 04 Loops | `while IFS= read -r old_file; do ... done` for cleanup |
| 05 Functions | `log`, `die`, `usage`, `cleanup` |
| 06 Files & directories | `mkdir -p`, `-d`, `-s` tests, `basename`/`dirname` |
| 07 Pipes & redirection | `tee -a`, `<<<`, `2>/dev/null` |
| 08 Text processing | `cut -f1`, `tail -n +6` |
| 09 Exit codes & errors | `set -euo pipefail`, `trap ... EXIT`, `die` |

## How It Actually Works

This script's execution is a chain of `fork()`/`execve()` calls stitched
together by the parent bash process: each external command (`tar`, `date`,
`find`, `mkdir`) becomes a separate short-lived process, while `wait()`
calls between them (implicit in sequential execution) make sure step *N*
fully finishes — including all its buffered writes reaching disk via the
kernel's file cache — before step *N+1* starts reading its output.

`tar` streams file data through its own internal buffer as it walks the
directory tree with `readdir(2)`/`stat(2)` calls; when combined with a
compressor like `gzip` via a pipe, the two run concurrently exactly as
described for pipelines elsewhere in this course — `tar` never waits for the
whole archive to be built before compression starts, it's compressed in a
continuous stream.

Timestamped filenames built from `$(date +%F)` rely on command substitution:
bash spawns `date` in a subshell, captures its stdout through an internal
pipe, strips the trailing newline, and splices the result into the string
being assigned — all of that happens once, at the moment the assignment
line executes, so every reference to that variable later in the script uses
the frozen value rather than re-invoking `date`.


## Stretch goals

- Add a `--dry-run` flag that logs what *would* happen without creating an
  archive or deleting anything.
- Compress with `zstd` instead of `gzip` if it's available, falling back to
  `gzip` otherwise (a `command -v zstd` check).
- Extend `log()` to also write JSON lines (`{"time": ..., "msg": ...}`) so the
  log can be parsed by other tools — you'll build a full log-monitoring
  script around exactly this idea in Level 2's project.

Completing this project means you're ready for **Level 2 · Intermediate**.
