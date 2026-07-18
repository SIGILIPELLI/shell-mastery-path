# 10 · Project — CLI Tool with Git Hook Integration

The Level 3 capstone: a multi-subcommand CLI tool, `taskctl`, that manages a
plain-text task list — plus a git pre-commit hook that uses it to block
commits containing unfinished `TODO`-tagged tasks.

## What you'll build

- `taskctl` — a single script with subcommands (`add`, `list`, `done`,
  `rm`), proper argument parsing, and a `--help` for each subcommand
- Tasks stored one-per-line in a plain text file, so the tool composes
  cleanly with `grep`/`awk`/`sort`
- A `pre-commit` git hook that runs `taskctl list --pending` scoped to
  staged files and blocks the commit if any staged file contains an
  unresolved `TODO(taskctl)` marker without a matching open task

## Project layout

```text
taskctl_project/
    bin/taskctl
    .git/hooks/pre-commit        (installed by install-hook.sh)
    install-hook.sh
    tasks.txt                     (created on first `taskctl add`)
```

## bin/taskctl

```bash
#!/usr/bin/env bash
# taskctl — a tiny CLI task manager with subcommands
set -euo pipefail

TASKCTL_FILE="${TASKCTL_FILE:-./tasks.txt}"

usage() {
    cat <<'EOF'
Usage: taskctl <command> [args]

Commands:
  add <description>     add a new pending task
  list [--pending]        list all tasks (or only pending ones)
  done <id>                mark a task done
  rm <id>                    remove a task
  help                        show this help

Environment:
  TASKCTL_FILE   path to the task store (default: ./tasks.txt)
EOF
}

die() {
    echo "taskctl: error: $1" >&2
    exit 1
}

ensure_store() {
    [[ -f "$TASKCTL_FILE" ]] || : > "$TASKCTL_FILE"
}

next_id() {
    # id is just "1 + highest existing id"; 0 if the store is empty
    awk -F'|' 'BEGIN{max=0} {if ($1+0>max) max=$1+0} END{print max+1}' "$TASKCTL_FILE"
}

cmd_add() {
    local description="$*"
    [[ -n "$description" ]] || die "add requires a description"
    ensure_store
    local id
    id=$(next_id)
    # format: id|status|description
    echo "${id}|pending|${description}" >> "$TASKCTL_FILE"
    echo "added task #$id: $description"
}

cmd_list() {
    ensure_store
    local pending_only="${1:-}"
    while IFS='|' read -r id status description; do
        [[ -z "$id" ]] && continue
        if [[ "$pending_only" == "--pending" && "$status" != "pending" ]]; then
            continue
        fi
        printf '#%-4s [%-7s] %s\n' "$id" "$status" "$description"
    done < "$TASKCTL_FILE"
}

cmd_done() {
    local id="${1:-}"
    [[ -n "$id" ]] || die "done requires a task id"
    ensure_store
    grep -q "^${id}|" "$TASKCTL_FILE" || die "no task with id $id"
    local tmp
    tmp=$(mktemp)
    awk -F'|' -v id="$id" 'BEGIN{OFS="|"} { if ($1==id) $2="done"; print }' \
        "$TASKCTL_FILE" > "$tmp"
    mv "$tmp" "$TASKCTL_FILE"
    echo "marked #$id done"
}

cmd_rm() {
    local id="${1:-}"
    [[ -n "$id" ]] || die "rm requires a task id"
    ensure_store
    grep -q "^${id}|" "$TASKCTL_FILE" || die "no task with id $id"
    local tmp
    tmp=$(mktemp)
    grep -v "^${id}|" "$TASKCTL_FILE" > "$tmp" || true
    mv "$tmp" "$TASKCTL_FILE"
    echo "removed #$id"
}

main() {
    local command="${1:-help}"
    shift || true
    case "$command" in
        add)   cmd_add "$@" ;;
        list)  cmd_list "$@" ;;
        done)  cmd_done "$@" ;;
        rm)    cmd_rm "$@" ;;
        help|-h|--help) usage ;;
        *) die "unknown command '$command' (see 'taskctl help')" ;;
    esac
}

main "$@"
```

## Running it

```bash
chmod +x bin/taskctl
export PATH="$PWD/bin:$PATH"

taskctl add "write the networking module"
# added task #1: write the networking module

taskctl add "review PR #42"
# added task #2: review PR #42

taskctl list
# #1    [pending] write the networking module
# #2    [pending] review PR #42

taskctl done 1
# marked #1 done

taskctl list --pending
# #2    [pending] review PR #42

taskctl rm 2
# removed #2
```

## The git hook: blocking commits with open TODO markers

The idea: any staged file can carry a `TODO(taskctl):` comment referencing a
task description. The pre-commit hook scans staged files for these markers
and cross-checks them against `taskctl list --pending` — if a marker's text
doesn't match any pending task, it's either stale or was never tracked, and
the commit is blocked until it's resolved or properly logged.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit — block commits with untracked TODO(taskctl) markers
set -euo pipefail

repo_root=$(git rev-parse --show-toplevel)
export TASKCTL_FILE="$repo_root/tasks.txt"

staged_files=$(git diff --cached --name-only --diff-filter=ACM)
[[ -z "$staged_files" ]] && exit 0

pending_tasks=$("$repo_root/bin/taskctl" list --pending 2>/dev/null || true)

blocked=0
while IFS= read -r file; do
    [[ -f "$file" ]] || continue
    while IFS=: read -r lineno marker; do
        [[ -z "$marker" ]] && continue
        text=$(echo "$marker" | sed -E 's/.*TODO\(taskctl\):\s*//')
        if ! echo "$pending_tasks" | grep -qF "$text"; then
            echo "BLOCKED: $file:$lineno has an untracked TODO(taskctl): $text"
            echo "  -> run: taskctl add \"$text\""
            blocked=1
        fi
    done < <(grep -n 'TODO(taskctl):' "$file" || true)
done <<< "$staged_files"

if [[ "$blocked" -eq 1 ]]; then
    echo ""
    echo "commit blocked: resolve or register the TODO(taskctl) markers above."
    exit 1
fi

exit 0
```

## install-hook.sh

```bash
#!/usr/bin/env bash
# install-hook.sh — copies pre-commit into .git/hooks and makes it executable
set -euo pipefail

repo_root=$(git rev-parse --show-toplevel)
cp hooks/pre-commit "$repo_root/.git/hooks/pre-commit"
chmod +x "$repo_root/.git/hooks/pre-commit"
echo "pre-commit hook installed."
```

## Trying the full flow

```bash
./install-hook.sh

taskctl add "finish the auth module"

echo '# TODO(taskctl): finish the auth module' >> auth.sh
git add auth.sh
git commit -m "wip: auth module"
# passes — the marker matches a pending task

echo '# TODO(taskctl): add rate limiting' >> auth.sh
git add auth.sh
git commit -m "wip: more auth"
# BLOCKED: auth.sh:2 has an untracked TODO(taskctl): add rate limiting
#   -> run: taskctl add "add rate limiting"
```

## How earlier modules show up here

| Module | Where it's used |
|--------|-------------------|
| Subcommand dispatch | `case "$command" in ... esac` in `main()` |
| awk field rewriting | `cmd_done`'s in-place status update |
| Signal-safe temp files | `mktemp` + `mv` instead of editing in place |
| Git hooks (03/06) | `.git/hooks/pre-commit`, `git diff --cached` |
| Process substitution (02) | `< <(grep -n ... "$file")` |

## Stretch goals

- Add a `taskctl edit <id> <new description>` subcommand.
- Make the hook also run on `git commit --amend` and reject empty
  descriptions.
- Package `taskctl` using Module 08's `install.sh` + man page pattern so it
  can be installed once and reused across every repo on the machine.
- Add a `taskctl.json` alternate storage backend and use `jq` (Module 09) to
  read/write it, behind a `TASKCTL_FORMAT=json` environment variable.
