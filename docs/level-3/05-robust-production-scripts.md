# 05 · Writing Robust Production Scripts

Everything so far has focused on individual techniques. This module is about
combining them into a mindset: writing scripts that behave predictably when
run by a stranger, at 3am, from cron, against production data, for the
hundredth time in a row.

## Input validation as a first-class concern

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat <<EOF
Usage: $0 --env <dev|staging|prod> --region <region> [--dry-run]
EOF
    exit 1
}

env=""
region=""
dry_run=false

while [[ $# -gt 0 ]]; do
    case "$1" in
        --env)     env="$2"; shift 2 ;;
        --region)  region="$2"; shift 2 ;;
        --dry-run) dry_run=true; shift ;;
        -h|--help) usage ;;
        *) echo "Unknown argument: $1" >&2; usage ;;
    esac
done

[[ -n "$env" ]] || { echo "Error: --env is required" >&2; usage; }
[[ "$env" =~ ^(dev|staging|prod)$ ]] || { echo "Error: --env must be dev, staging, or prod" >&2; usage; }
[[ -n "$region" ]] || { echo "Error: --region is required" >&2; usage; }
```

Validating every input *before* doing anything irreversible means a typo
("prod " with a trailing space, an unsupported region) fails loudly and
immediately — not halfway through a deploy.

## Idempotency — safe to run more than once

```bash
# NOT idempotent — running it twice creates two identical entries
echo "export PATH=\$PATH:/opt/tool/bin" >> ~/.bashrc

# IDEMPOTENT — only adds the line if it's not already there
grep -qxF 'export PATH=$PATH:/opt/tool/bin' ~/.bashrc || \
    echo 'export PATH=$PATH:/opt/tool/bin' >> ~/.bashrc
```

```bash
# NOT idempotent — fails the second time (directory already exists)
mkdir /opt/myapp

# IDEMPOTENT
mkdir -p /opt/myapp
```

```bash
# NOT idempotent — creates a second cron entry every run
(crontab -l; echo "0 2 * * * /opt/myapp/backup.sh") | crontab -

# IDEMPOTENT — remove any existing entry for this script first
(crontab -l 2>/dev/null | grep -v "/opt/myapp/backup.sh"; echo "0 2 * * * /opt/myapp/backup.sh") | crontab -
```

A production script that gets re-run — after a partial failure, by a retry
policy, or just by an operator being cautious — should reach the same end
state whether it's the first run or the fifth. Design every step to check
"is this already done?" before doing it.

## Dry-run mode

```bash
run() {
    if $dry_run; then
        echo "[DRY RUN] would run: $*"
    else
        "$@"
    fi
}

run rm -rf "$stale_dir"
run systemctl restart myapp
```

A `--dry-run` flag that prints what *would* happen, without doing it, lets
operators (and CI) safely preview a risky script before committing to it.

## Structured, leveled logging

```bash
LOG_LEVEL="${LOG_LEVEL:-INFO}"

log() {
    local level="$1"; shift
    local levels=("DEBUG" "INFO" "WARN" "ERROR")
    local current_idx msg_idx

    for i in "${!levels[@]}"; do
        [[ "${levels[$i]}" == "$LOG_LEVEL" ]] && current_idx=$i
        [[ "${levels[$i]}" == "$level" ]] && msg_idx=$i
    done

    if (( msg_idx >= current_idx )); then
        printf '%s [%s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$level" "$*" >&2
    fi
}

log INFO "starting deploy to $env/$region"
log DEBUG "this only shows when LOG_LEVEL=DEBUG"
log ERROR "deploy failed: connection refused"
```

Leveled logging (respecting a `LOG_LEVEL` env var) means the same script
can run quiet in normal operation and verbose when you're actively
debugging it — without editing the script itself.

## Retries with backoff for flaky operations

```bash
retry() {
    local max_attempts=5
    local delay=1
    local attempt=1

    until "$@"; do
        if (( attempt >= max_attempts )); then
            echo "giving up after $attempt attempts: $*" >&2
            return 1
        fi
        echo "attempt $attempt failed, retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
        ((delay *= 2))     # exponential backoff: 1s, 2s, 4s, 8s...
    done
}

retry curl -sf "https://api.example.com/health"
```

## Preconditions and postconditions

```bash
# preconditions: fail before starting if the environment isn't ready
command -v docker >/dev/null || { echo "docker is required" >&2; exit 1; }
[[ -n "${DEPLOY_TOKEN:-}" ]] || { echo "DEPLOY_TOKEN must be set" >&2; exit 1; }

# ... do the work ...

# postconditions: verify the result actually happened before reporting success
if ! curl -sf "https://myapp.example.com/health" > /dev/null; then
    log ERROR "post-deploy health check failed"
    exit 1
fi
log INFO "deploy verified healthy"
```

## Cheat sheet

| Practice | Why |
|----------|-----|
| Validate all inputs up front | fail fast, before anything irreversible |
| Make every step idempotent | safe to re-run after a partial failure |
| Support `--dry-run` | lets operators preview risky actions |
| Leveled logging (`LOG_LEVEL`) | quiet by default, verbose on demand |
| Retry with backoff | tolerate transient/flaky failures |
| Check post-conditions, not just exit codes | confirms the *actual* desired end state |

## Exercise

Take the `log_monitor.sh` project from Level 2 and harden it: add `--dry-run`
support (skip writing the report file, just print what it would contain),
add leveled logging respecting a `LOG_LEVEL` env var, and make report-file
creation idempotent (skip regenerating if an identical report already
exists for the same log file content, e.g. compared by checksum).
