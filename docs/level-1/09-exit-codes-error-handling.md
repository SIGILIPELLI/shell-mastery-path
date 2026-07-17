# 09 · Exit Codes & Basic Error Handling

## Every command returns an exit code

```bash
ls /tmp
echo "$?"        # 0 — success

ls /nonexistent
echo "$?"        # 2 (or another non-zero code) — failure
```

By convention, `0` means success and any non-zero value (1–255) means some
kind of failure — the specific non-zero number is often command-specific.

## Using exit codes in conditionals

```bash
if grep -q "error" app.log; then
    echo "found errors"
else
    echo "no errors found"
fi
```

`-q` makes `grep` "quiet" — no output, it just sets its exit code, which is
exactly what an `if` needs.

```bash
mkdir -p /tmp/build && cd /tmp/build && echo "ready to build"
# each step only runs if the previous one succeeded

cp important.txt backup/ || echo "backup copy failed!" >&2
# only runs the echo if cp failed
```

## `exit` — setting your own script's exit code

```bash
#!/usr/bin/env bash

if [[ ! -f "config.yml" ]]; then
    echo "Error: config.yml not found" >&2
    exit 1
fi

echo "config found, continuing..."
exit 0
```

A script's own exit code (from `exit N`, or implicitly the last command run)
is what the *caller* of your script sees in `$?` — important once your
scripts are called from other scripts, cron, or CI.

## `set -e` — stop on the first error

```bash
#!/usr/bin/env bash
set -e     # exit immediately if any command exits non-zero

mkdir -p /tmp/build
cd /tmp/build
cp /nonexistent/file .    # this fails...
echo "this line never runs"   # ...so set -e stops the script here
```

Without `set -e`, a script keeps running after a failed command, silently
building on top of a broken state — usually not what you want in anything
beyond a quick interactive check.

## set -u — catch unset variables

```bash
#!/usr/bin/env bash
set -u    # error out if you reference a variable that was never set

echo "$UNDEFINED_VAR"    # bash: UNDEFINED_VAR: unbound variable — script exits
```

`set -u` catches typos in variable names early instead of silently
substituting an empty string, which is what happens by default.

## set -euo pipefail — the standard "strict mode" trio

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e: stop on any error
# -u: error on unset variables
# -o pipefail: a pipeline fails if ANY command in it fails, not just the last

grep "error" nonexistent_file.txt | wc -l
# without pipefail, this would still "succeed" (wc always exits 0)
# with pipefail, the failed grep makes the whole pipeline fail
```

This trio is covered in more depth in Level 2's "Scripting Best Practices,"
but it's worth adopting from your very first real script.

## trap — running cleanup code on exit

```bash
#!/usr/bin/env bash

cleanup() {
    echo "cleaning up temp files..."
    rm -f /tmp/myscript.tmp
}

trap cleanup EXIT     # run cleanup no matter HOW the script ends

echo "working..." > /tmp/myscript.tmp
echo "doing work"
exit 0                 # cleanup still runs, because of the trap
```

`trap` registers a function or command to run when a given signal (or the
special pseudo-signal `EXIT`) occurs — this guarantees temp files get cleaned
up even if the script exits early or hits an error. (Level 3's "Signal
Handling" module covers `SIGINT`/`SIGTERM` traps for interrupting long-running
scripts gracefully.)

## A simple, defensive error-handling pattern

```bash
#!/usr/bin/env bash
set -euo pipefail

die() {
    echo "Error: $1" >&2
    exit "${2:-1}"
}

[[ -f "input.csv" ]] || die "input.csv not found"

line_count=$(wc -l < input.csv)
(( line_count > 0 )) || die "input.csv is empty"

echo "processing $line_count lines..."
```

A small `die` helper like this keeps error paths short and readable, and
sends the message to stderr where it belongs (not stdout, which might be
piped into something else).

## Cheat sheet

| Construct | Purpose |
|-----------|---------|
| `$?` | exit code of the last command |
| `exit N` | end the script with exit code N |
| `set -e` | stop the script on the first failing command |
| `set -u` | error on unset variable references |
| `set -o pipefail` | a pipeline fails if any stage fails |
| `trap cmd EXIT` | run `cmd` when the script exits, always |
| `cmd1 \|\| cmd2` | run cmd2 only if cmd1 fails |

## Exercise

Write `safe_copy.sh` with `set -euo pipefail` at the top that takes a source
and destination path as `$1`/`$2`, prints a usage message and exits 1 if
either is missing, uses a `die` helper to fail clearly if the source file
doesn't exist, and uses `trap` to print `"done"` on exit regardless of
success or failure.
