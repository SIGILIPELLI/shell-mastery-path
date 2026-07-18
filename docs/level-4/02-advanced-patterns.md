# 02 · Advanced Scripting Patterns

Once a script grows past a few dozen lines, "just add another `if`"
stops working. This module covers two patterns that keep complex scripts
readable as they grow: state machines for scripts that move through
distinct phases, and config-driven scripts that separate *what to do*
from *how to do it*.

## Why state machines

Scripts that install software, run multi-step deployments, or process
data pipelines naturally move through phases — download, verify, extract,
install, cleanup. Modeling those phases explicitly as a **state machine**
(instead of one long linear script) makes it possible to resume after a
failure, log progress meaningfully, and reason about what "currently
running" means.

## A basic state machine in bash

```bash
#!/usr/bin/env bash
# installer.sh — a simple state machine
set -euo pipefail

state="start"

while [[ "$state" != "done" ]]; do
    case "$state" in
        start)
            echo "==> checking prerequisites"
            command -v curl >/dev/null || { echo "curl required" >&2; exit 1; }
            state="download"
            ;;
        download)
            echo "==> downloading package"
            curl -fsSL -o /tmp/pkg.tar.gz "https://example.com/pkg.tar.gz"
            state="verify"
            ;;
        verify)
            echo "==> verifying checksum"
            # (checksum check would go here)
            state="install"
            ;;
        install)
            echo "==> installing"
            tar -xzf /tmp/pkg.tar.gz -C /opt/myapp
            state="cleanup"
            ;;
        cleanup)
            echo "==> cleaning up"
            rm -f /tmp/pkg.tar.gz
            state="done"
            ;;
        *)
            echo "Unknown state: $state" >&2
            exit 1
            ;;
    esac
done

echo "Install complete."
```

Each state does one thing and explicitly sets the next state. That
explicitness is the whole point — you can log "entering state X", handle
errors per-state, or jump straight to a specific state when debugging.

## Making state resumable

Persist the current state to a file so a failed run can pick up where
it left off instead of restarting from scratch:

```bash
STATE_FILE="/var/tmp/installer.state"

save_state() { echo "$1" > "$STATE_FILE"; }
load_state() { [[ -f "$STATE_FILE" ]] && cat "$STATE_FILE" || echo "start"; }

state=$(load_state)

while [[ "$state" != "done" ]]; do
    save_state "$state"     # persist BEFORE running the step
    case "$state" in
        start)    state="download" ;;
        download) state="verify" ;;
        verify)   state="install" ;;
        install)  state="cleanup" ;;
        cleanup)  state="done" ;;
    esac
done

rm -f "$STATE_FILE"   # clean up only once fully done
```

If the script crashes mid-`install`, re-running it resumes at `install`
instead of re-downloading everything.

## State transition tables

For state machines with conditional transitions (not just a straight
line), an associative array can encode the transition table separately
from the logic that runs each state:

```bash
declare -A NEXT_STATE=(
    [start]="check_disk_space"
    [check_disk_space:ok]="download"
    [check_disk_space:low]="cleanup_old_versions"
    [cleanup_old_versions]="check_disk_space"
    [download]="install"
    [install]="done"
)

state="start"
while [[ "$state" != "done" ]]; do
    case "$state" in
        check_disk_space)
            if [[ $(df / | awk 'NR==2{print $4}') -gt 1000000 ]]; then
                result="ok"
            else
                result="low"
            fi
            state="${NEXT_STATE[${state}:${result}]}"
            continue
            ;;
        *)
            echo "-> $state"
            state="${NEXT_STATE[$state]:-done}"
            ;;
    esac
done
```

## Config-driven scripts

Instead of hardcoding hostnames, paths, and thresholds inside the script
body, read them from an external config file. This lets the same script
behave differently per environment without editing code.

```ini
# deploy.conf
APP_NAME=myapp
DEPLOY_DIR=/opt/myapp
BACKUP_COUNT=5
HEALTHCHECK_URL=http://localhost:8080/health
```

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="${1:-deploy.conf}"
[[ -f "$CONFIG_FILE" ]] || { echo "Config not found: $CONFIG_FILE" >&2; exit 1; }

# shellcheck source=/dev/null
source "$CONFIG_FILE"

: "${APP_NAME:?APP_NAME must be set in $CONFIG_FILE}"
: "${DEPLOY_DIR:?DEPLOY_DIR must be set in $CONFIG_FILE}"

echo "Deploying $APP_NAME to $DEPLOY_DIR (keeping $((BACKUP_COUNT)) backups)"
```

`source`-ing a config file is simple but trusts its contents as shell
code — only load config files you control. The `: "${VAR:?message}"`
idiom is a cheap required-field check: it exits with `message` if `VAR`
is unset or empty.

## Config-driven with key=value parsing (no `source`)

When the config file might come from an untrusted or user-editable
source, parse it as data instead of executing it:

```bash
declare -A config

while IFS='=' read -r key value; do
    [[ -z "$key" || "$key" == \#* ]] && continue   # skip blanks/comments
    config["$key"]="$value"
done < deploy.conf

echo "App: ${config[APP_NAME]}"
echo "Dir: ${config[DEPLOY_DIR]}"
```

## Driving behavior from config: action tables

Combine a config file with a dispatch table so adding a new supported
action means adding a config line, not editing the script:

```bash
declare -A ACTIONS=(
    [start]="start_app"
    [stop]="stop_app"
    [restart]="restart_app"
    [status]="status_app"
)

start_app()   { echo "starting ${config[APP_NAME]}"; }
stop_app()    { echo "stopping ${config[APP_NAME]}"; }
restart_app() { stop_app; start_app; }
status_app()  { echo "checking ${config[HEALTHCHECK_URL]}"; }

command="${1:?Usage: $0 <start|stop|restart|status>}"
fn="${ACTIONS[$command]:-}"
[[ -z "$fn" ]] && { echo "Unknown command: $command" >&2; exit 1; }
"$fn"
```

## Cheat sheet

| Pattern | Purpose |
|---------|---------|
| `case "$state" in ... esac` loop | core of a bash state machine |
| `save_state` / `load_state` to a file | make a state machine resumable |
| `declare -A NEXT_STATE=(...)` | table-driven state transitions |
| `source config.conf` | load trusted config as shell variables |
| `: "${VAR:?msg}"` | fail fast if a required config value is missing |
| `while IFS='=' read -r k v` | parse untrusted config as plain data |
| `declare -A ACTIONS=([cmd]=fn)` | dispatch table for config-driven commands |

## Exercise

Build `pipeline.sh`, a state machine with states `fetch` → `transform` →
`load` → `done`, where each state prints what it's doing and sleeps 1
second to simulate work. Persist the current state to
`/tmp/pipeline.state` before each transition. Run it, kill it with
`Ctrl-C` partway through, then re-run it and confirm it resumes from the
interrupted state instead of starting over.
