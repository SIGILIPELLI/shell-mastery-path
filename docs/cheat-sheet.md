---
title: "Shell Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Shell cheat sheet: 33 quick-reference tables from every level of the Shell Mastery Path, each linked to its full lesson."
---

# Shell Cheat Sheet

Quick-reference tables from **33 lessons** of the free Shell Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; First Script](level-1/01-setup.md)

| Command | Purpose |
|---------|---------|
| `pwd` | print current directory |
| `ls -la` | list all files, long format |
| `cd path` | change directory |
| `chmod +x file` | make a file executable |
| `./script.sh` | run a script in the current directory |
| `bash script.sh` | run a script with bash explicitly |
| `read -r var` | read a line of input into `var` |

### [04 · Loops](level-1/04-loops.md)

| Loop | Runs while... |
|------|----------------|
| `for x in list; do ... done` | once per item in the list |
| `for (( i=0; i<n; i++ )); do ... done` | C-style numeric counting |
| `while [[ cond ]]; do ... done` | condition is true |
| `until [[ cond ]]; do ... done` | condition is false |
| `break` | — exits the loop immediately |
| `continue` | — skips to the next iteration |

### [05 · Functions &amp; Arguments](level-1/05-functions-arguments.md)

| Symbol | Meaning inside a function |
|--------|-----------------------------|
| `$1`, `$2`, ... | positional arguments passed to the function |
| `$#` | number of arguments passed |
| `"$@"` | all arguments, each preserved as a separate word |
| `"$*"` | all arguments joined into one string |
| `local var` | scope a variable to this function only |
| `return N` | set the function's exit status (0-255, not a value) |
| `echo value` + `$(func)` | the idiom for "returning" an actual value |

### [07 · Pipes &amp; Redirection](level-1/07-pipes-redirection.md)

| Syntax | Meaning |
|--------|---------|
| `cmd > file` | stdout to file (overwrite) |
| `cmd >> file` | stdout to file (append) |
| `cmd 2> file` | stderr to file |
| `cmd > file 2>&1` | stdout and stderr both to file |
| `cmd &> file` | bash shorthand for the line above |
| `cmd < file` | file as stdin |
| `cmd1 \| cmd2` | pipe cmd1's stdout into cmd2's stdin |
| `cmd \| tee file` | show output AND save it to file |
| `$(cmd)` | capture a command's stdout as a string |

### [08 · Basic Text Processing](level-1/08-text-processing.md)

| Tool | Job |
|------|-----|
| `grep` | find lines matching a pattern |
| `cut` | pull out specific columns by delimiter/position |
| `sort` | order lines (`-n` numeric, `-r` reverse, `-k` by field) |
| `uniq` | collapse adjacent duplicates (`-c` to count) |
| `sed` | stream-edit lines (substitute, delete, print ranges) |
| `awk` | field-aware processing, math, conditional printing |

### [09 · Exit Codes &amp; Basic Error Handling](level-1/09-exit-codes-error-handling.md)

| Construct | Purpose |
|-----------|---------|
| `$?` | exit code of the last command |
| `exit N` | end the script with exit code N |
| `set -e` | stop the script on the first failing command |
| `set -u` | error on unset variable references |
| `set -o pipefail` | a pipeline fails if any stage fails |
| `trap cmd EXIT` | run `cmd` when the script exits, always |
| `cmd1 \|\| cmd2` | run cmd2 only if cmd1 fails |

## Level 2 · Intermediate

### [01 · Arrays &amp; Associative Arrays](level-2/01-arrays.md)

| Syntax | Meaning |
|--------|---------|
| `arr=(a b c)` | create an indexed array |
| `declare -A arr` | declare an associative array (required before use) |
| `${arr[i]}` | element at index/key `i` |
| `${arr[@]}` | all elements |
| `${#arr[@]}` | number of elements |
| `${!arr[@]}` | all indices/keys |
| `arr+=(x)` | append `x` to an indexed array |
| `unset 'arr[i]'` | remove element at index/key `i` |
| `${arr[@]:start:len}` | slice |

### [02 · String Manipulation](level-2/02-string-manipulation.md)

| Expansion | Meaning |
|-----------|---------|
| `${#s}` | length of `s` |
| `${s:i:n}` | substring, `n` chars from index `i` |
| `${s/old/new}` | replace first match |
| `${s//old/new}` | replace all matches |
| `${s#pattern}` / `${s##pattern}` | trim shortest/longest match from the front |
| `${s%pattern}` / `${s%%pattern}` | trim shortest/longest match from the back |
| `${s^^}` / `${s,,}` | uppercase / lowercase all |
| `${s:-default}` | use default if unset/empty |
| `${s:=default}` | use AND assign default if unset/empty |

### [03 · Regular Expressions in Bash](level-2/03-regular-expressions.md)

| Pattern | Matches |
|---------|---------|
| `^` / `$` | start / end of line |
| `.` | any character |
| `*` / `+` / `?` | 0+, 1+, 0-or-1 of the preceding token |
| `{n,m}` | between n and m repetitions |
| `[...]` / `[^...]` | a character class / its negation |
| `(...)` | a capture group |
| `a\|b` | alternation: a OR b |
| `[[ $s =~ regex ]]` | match `$s` against `regex`, populate `BASH_REMATCH` |

### [04 · Process Management](level-2/04-process-management.md)

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

### [05 · Scripting Best Practices](level-2/05-best-practices.md)

| Practice | Why |
|----------|-----|
| `set -euo pipefail` | fail loudly instead of silently continuing |
| Quote all variable expansions | avoid word-splitting/glob bugs |
| Use `[[ ]]` not `[ ]` | safer, more expressive conditionals |
| Run `shellcheck` | catches bugs before they run in production |
| `UPPER_SNAKE_CASE` for constants | visually distinguishes config from local state |
| `local` inside every function | avoids leaking variables into global scope |
| `$( )` not backticks | nestable, readable command substitution |
| Validate inputs first, fail fast | keeps the rest of the logic simple |

### [06 · Debugging Scripts](level-2/06-debugging.md)

| Tool | Purpose |
|------|---------|
| `bash -n script.sh` | check syntax only, don't execute |
| `set -x` / `set +x` | trace commands as they run / stop tracing |
| `bash -x script.sh` | trace an entire script without editing it |
| `PS4` | customize the `set -x` trace prefix (add `$LINENO`, etc.) |
| `trap 'handler' ERR` | run a handler whenever a command fails |
| `declare -p var` | print a variable's exact type and value |
| `$LINENO` | current line number |

### [07 · Scheduling &amp; Automation](level-2/07-scheduling-automation.md)

| Command | Purpose |
|---------|---------|
| `crontab -e` | edit the current user's crontab |
| `crontab -l` | list current cron jobs |
| `* * * * *` | minute hour day-of-month month day-of-week |
| `*/N` | every N units of that field |
| `at TIME` | schedule a one-off command |
| `atq` / `atrm N` | list / cancel pending `at` jobs |
| `>> log 2>&1` | always redirect cron job output to a log file |

### [08 · Environment &amp; Configuration Management](level-2/08-environment-config.md)

| Concept | Detail |
|---------|--------|
| shell variable | local to the current shell only |
| `export VAR` | promotes it into the environment, inherited by child processes |
| `~/.bashrc` | loaded for interactive non-login shells |
| `~/.bash_profile` / `~/.profile` | loaded for login shells |
| `source file.sh` (or `. file.sh`) | run in the current shell — variables persist |
| `./file.sh` | run in a subshell — variables do NOT persist |
| `${!name}` | indirect expansion — look up variable by a name stored in another variable |

### [09 · Testing Shell Scripts](level-2/09-testing-shell-scripts.md)

| Construct | Purpose |
|-----------|---------|
| `@test "name" { ... }` | one test case |
| `run cmd` | execute `cmd`, capturing `$status`/`$output`/`$lines` |
| `setup()` / `teardown()` | run before/after every test in the file |
| `load "file.sh"` | source a script so its functions are testable |
| `[ "$status" -eq 0 ]` | assert a successful exit code |
| `[[ "$output" == *"text"* ]]` | assert output contains a substring |

## Level 3 · Advanced

### [01 · Advanced sed &amp; awk](level-3/01-advanced-sed-awk.md)

| Command | Purpose |
|---------|---------|
| `sed 's/x/y/' ` | substitute first match per line |
| `sed 's/x/y/g'` | substitute all matches per line |
| `sed -n 'N,Mp'` | print only lines N through M |
| `sed -i.bak 's/x/y/'` | in-place edit with a backup |
| `sed '/A/,/B/d'` | delete a block between two markers |
| `awk '{print $N}'` | print field N |
| `awk -F, ...` | set the field separator |
| `awk 'pattern {action}'` | run action only on matching lines |
| `awk 'BEGIN{...} {...} END{...}'` | setup / per-line / final summary |

### [02 · Process Substitution &amp; Here-Docs](level-3/02-process-substitution-heredocs.md)

| Syntax | Meaning |
|--------|---------|
| `<(cmd)` | run `cmd`, expose its stdout as a readable "file" |
| `>(cmd)` | expose a writable "file" whose content becomes `cmd`'s stdin |
| `cmd <<EOF ... EOF` | here-document: multi-line stdin, `$vars` expanded |
| `cmd <<'EOF' ... EOF` | here-document with NO expansion (literal) |
| `cmd <<-EOF ... EOF` | here-document, strips leading tabs |
| `cmd <<< "$string"` | here-string: a single value as stdin |

### [03 · Signal Handling](level-3/03-signal-handling.md)

| Construct | Purpose |
|-----------|---------|
| `trap 'handler' SIGINT` | run `handler` when SIGINT (Ctrl+C) is received |
| `trap 'handler' SIGTERM` | run `handler` when SIGTERM (from `kill`) is received |
| `trap 'handler' SIGINT SIGTERM` | one handler for multiple signals |
| `trap '' SIGINT` | ignore a signal |
| `trap - SIGINT` | restore default handling |
| `trap 'handler' EXIT` | always run on exit, regardless of cause |
| `kill -SIGNAL pid` | send a named/numbered signal to a process |

### [04 · Parallelism](level-3/04-parallelism.md)

| Command | Purpose |
|---------|---------|
| `xargs -P N` | run up to N processes concurrently |
| `xargs -0` | NUL-delimited input (safe with `find -print0`) |
| `parallel -j N` | GNU parallel, N concurrent jobs |
| `parallel --joblog file` | record exit status/runtime per job |
| `jobs -r -p \| wc -l` | count currently running background jobs |
| `wait -n` | wait for the next background job (not all) to finish |

### [05 · Writing Robust Production Scripts](level-3/05-robust-production-scripts.md)

| Practice | Why |
|----------|-----|
| Validate all inputs up front | fail fast, before anything irreversible |
| Make every step idempotent | safe to re-run after a partial failure |
| Support `--dry-run` | lets operators preview risky actions |
| Leveled logging (`LOG_LEVEL`) | quiet by default, verbose on demand |
| Retry with backoff | tolerate transient/flaky failures |
| Check post-conditions, not just exit codes | confirms the *actual* desired end state |

### [06 · Integrating with Git Hooks &amp; CI Scripts](level-3/06-git-hooks-ci.md)

| Hook/tool | Purpose |
|-----------|---------|
| `.git/hooks/pre-commit` | validate/lint before a commit is created |
| `.git/hooks/commit-msg` | validate the commit message text |
| `.git/hooks/pre-push` | validate before pushing to a remote |
| `core.hooksPath` | point git at a version-controlled hooks directory |
| `git diff --cached --name-only` | list staged files (for a pre-commit hook) |
| CI `run:` step | just a shell script — same conventions apply |

### [07 · Security Considerations](level-3/07-security-considerations.md)

| Risk | Mitigation |
|------|------------|
| `eval` on untrusted input | avoid `eval`; use arrays/namerefs instead |
| Filenames starting with `-` | always pass `--` before user-supplied args |
| Unquoted variables | quote everything; prevents word-splitting/globbing |
| Path traversal (`../`) | validate/reject `..` and enforce an allowed base path |
| World-writable config files | `umask`, `chmod`, check permissions before sourcing |
| Secrets in arguments/logs | pass via env vars or restricted files, never as CLI text; keep `set -x` off around them |

### [08 · Packaging &amp; Distributing Scripts](level-3/08-packaging-distributing.md)

| Task | Command |
|------|---------|
| Install a script to `$PATH` | `install -Dm755 bin/tool /usr/local/bin/tool` |
| Install a man page | `install -Dm644 man/tool.1 /usr/local/share/man/man1/tool.1` |
| Preview a man page before installing | `man ./man/tool.1` |
| Check where a command resolves | `which tool` / `type tool` |
| Read `VERSION` in a script | `VERSION=$(cat VERSION)` |

### [09 · Networking from the Shell](level-3/09-networking.md)

| Task | Command |
|------|---------|
| GET and print body | `curl https://host/path` |
| Fail script on HTTP error | `curl -fsS https://host/path` |
| POST JSON | `curl -X POST -H "Content-Type: application/json" -d '{...}' url` |
| Just the HTTP status code | `curl -w '%{http_code}' -o /dev/null -s url` |
| Download, resume-capable | `wget -c url` |
| Extract a JSON field | `curl -s url \| jq -r '.field'` |
| Check if a port is open | `(echo > /dev/tcp/host/port) 2>/dev/null` |

## Level 4 · Master

### [01 · Building Full CLI Tools](level-4/01-building-cli-tools.md)

| Pattern | Purpose |
|---------|---------|
| `while [[ $# -gt 0 ]]; do case "$1" in ...` | manual parsing for long-only flags |
| `getopts ":e:fvh" opt` | POSIX short-option parsing, `e:` = takes a value |
| `$OPTARG` | value attached to the current `getopts` flag |
| `$OPTIND` | index of the next unparsed argument |
| `shift $((OPTIND - 1))` | drop parsed flags, leaving positionals in `$@` |
| `--` | conventional "end of options" marker |
| `exit 2` | usage-error convention (vs. `1` for general failure) |

### [02 · Advanced Scripting Patterns](level-4/02-advanced-patterns.md)

| Pattern | Purpose |
|---------|---------|
| `case "$state" in ... esac` loop | core of a bash state machine |
| `save_state` / `load_state` to a file | make a state machine resumable |
| `declare -A NEXT_STATE=(...)` | table-driven state transitions |
| `source config.conf` | load trusted config as shell variables |
| `: "${VAR:?msg}"` | fail fast if a required config value is missing |
| `while IFS='=' read -r k v` | parse untrusted config as plain data |
| `declare -A ACTIONS=([cmd]=fn)` | dispatch table for config-driven commands |

### [03 · Performance Optimization](level-4/03-performance-optimization.md)

| Slow pattern | Fast alternative |
|---------------|-------------------|
| `$(external_cmd)` inside a per-line loop | do the work in one `awk`/`sed` pass |
| `expr 3 + 4` | `$((3 + 4))` |
| `` name=$(basename "$p") `` | `name="${p##*/}"` |
| `` dir=$(dirname "$p") `` | `dir="${p%/*}"` |
| `echo "$s" \| grep -q x` | `[[ "$s" == *x* ]]` |
| `cat file \| wc -l` (UUOC) | `wc -l < file` |
| `for f in *; do rm "$f"; done` | `rm -- *` |
| guessing about speed | `time cmd`, `TIMEFORMAT` |

### [04 · DevOps Automation](level-4/04-devops-automation.md)

| Pattern | Purpose |
|---------|---------|
| `exec "$@"` | hand off to the main process as PID 1 (correct signal handling) |
| `wait_for host port` | block until a dependency is reachable |
| `trap cleanup SIGTERM SIGINT` | run cleanup logic before a monitored child exits |
| `envsubst < tmpl > conf` | render config files from environment variables |
| `: "${VAR:?msg}"` | fail fast on missing required environment variables |
| `echo "::group::x"` | fold CI log output (GitHub Actions) |
| `echo "k=v" >> "$GITHUB_OUTPUT"` | pass values between CI pipeline stages |
| `run_step "name" cmd...` | name-and-fail-fast wrapper for pipeline steps |

### [05 · Cross-Shell Compatibility](level-4/05-cross-shell-compatibility.md)

| Feature | POSIX `sh` | bash | zsh (default opts) |
|---------|:---:|:---:|:---:|
| Arrays `arr=(a b)` | no | yes, 0-indexed | yes, 1-indexed |
| `[[ ]]` extended test | no | yes | yes |
| `${var^^}` / `${var,,}` | no | yes (bash 4+) | no |
| Process substitution `<(...)` | no | yes | yes |
| `set -o pipefail` | no | yes | yes |
| Unquoted var word-splitting | yes | yes | no (needs `SH_WORD_SPLIT`) |
| `local` in functions | not POSIX, widely supported | yes | yes |
| Safe cross-shell test | `case`/`[ ]` | anything | anything except relying on word-splitting |

### [06 · Advanced Text Processing Pipelines](level-4/06-text-processing-pipelines.md)

| Tool | Best for |
|------|----------|
| `grep` | filtering lines by pattern |
| `awk` | field extraction, math, reports |
| `sed` | line-based text substitution/reshaping |
| `jq` | querying/transforming JSON |
| `jq -r` | raw (unquoted) string output |
| `jq -s` | slurp multiple JSON lines/values into one array |
| `sort \| uniq -c` | count duplicate lines |
| `@csv` (in jq) | render an array as a CSV row |

### [07 · System Administration Scripting](level-4/07-sysadmin-scripting.md)

| Command | Purpose |
|---------|---------|
| `useradd -m -s /bin/bash user` | create a user with a home dir and shell |
| `usermod -aG group user` | add a user to a group (append, don't replace) |
| `userdel -r user` | delete a user and their home dir |
| `id user` / `getent passwd user` | inspect a user's identity |
| `systemctl start/stop/status svc` | control a systemd service |
| `systemctl enable svc` | start automatically on boot |
| `journalctl -u svc -f` | follow a service's logs |
| `*.timer` unit | systemd's cron-like scheduling primitive |

### [08 · Logging &amp; Observability for Scripts](level-4/08-logging-observability.md)

| Technique | Purpose |
|-----------|---------|
| JSON log lines | machine-parseable structured logging |
| log levels (`debug`/`info`/`warn`/`error`) | control verbosity |
| a `run_id`/correlation ID | trace one execution across log lines/systems |
| `logger -t tag` | send lines to syslog/journald |
| textfile-collector `.prom` file | expose metrics to Prometheus without a daemon |
| heartbeat file/ping | detect when a scheduled job silently stopped running |
| `trap ... ERR` + webhook | alert immediately on failure |

### [09 · Security Hardening for Production Scripts](level-4/09-security-hardening.md)

| Practice | Why |
|----------|-----|
| never hardcode secrets in source | leaks via version control, `ps`, logs |
| secret files, mode 600, owned by the service account | standard least-exposure pattern |
| secrets managers (Vault, cloud KMS) | short-lived, centrally audited credentials |
| `set +x` around secret handling | prevents `-x` tracing from leaking values |
| dedicated non-root service account | limits blast radius of a compromised script |
| `setpriv`/`sudo -u` to drop privileges | run as root only as long as strictly necessary |
| systemd sandboxing directives | defense in depth even if the script is compromised |
| a hardening-check script in CI | catches regressions automatically, not just at review time |

---

Go deeper with the full [Shell Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
