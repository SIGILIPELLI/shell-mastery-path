# 10 · Capstone Project — Deployment Automation Tool

This is the capstone for the entire Shell Mastery Path: `deploy-tool`, a
production-style deployment automation CLI. It ships a Capistrano-style
release layout (`releases/<timestamp>/` plus a `current` symlink), supports
instant rollbacks, and folds in nearly every skill from Levels 1-4 —
subcommand dispatch, structured JSON logging, least-privilege execution
and secrets handling, `bats` tests, a `Dockerfile`, and a CI pipeline that
lints and tests it automatically on every push.

## What you'll build

- `bin/deploy-tool` — a single CLI with four subcommands: `deploy`,
  `rollback`, `status`, and `logs`
- `lib/releases.sh` — helpers for creating, listing, activating, and
  pruning releases in the Capistrano-style directory layout
- `lib/logging.sh` — shared structured JSON logging (Module 08), used by
  every subcommand
- A least-privilege guard that refuses to deploy as root, plus a
  secrets-aware webhook notifier that reads its token from a mode-600
  file (Module 09)
- `tests/deploy-tool.bats` — a bats suite exercising deploy, status,
  rollback, and pruning end-to-end
- A `Dockerfile` packaging the tool into a minimal, non-root image
- A GitHub Actions pipeline that runs `shellcheck` and the bats suite on
  every push

## Project layout

```text
deploy-tool/
    bin/
        deploy-tool
    lib/
        logging.sh
        releases.sh
    tests/
        deploy-tool.bats
    .github/
        workflows/
            ci.yml
    Dockerfile
```

## lib/logging.sh

```bash
#!/usr/bin/env bash
# lib/logging.sh — structured JSON logging shared by every deploy-tool subcommand

LOG_FILE="${DEPLOY_LOG_FILE:-/var/log/deploy-tool/deploy-tool.log}"
mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null || true

_log() {
    local level="$1"; shift
    local message="$1"
    local timestamp
    timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)

    # one JSON object per line — the same format Module 08 builds up from scratch
    printf '{"time":"%s","level":"%s","pid":%d,"msg":"%s"}\n' \
        "$timestamp" "$level" "$$" "$message" | tee -a "$LOG_FILE" >&2
}

log_info()  { _log "info"  "$1"; }
log_warn()  { _log "warn"  "$1"; }
log_error() { _log "error" "$1"; }
```

## lib/releases.sh

```bash
#!/usr/bin/env bash
# lib/releases.sh — Capistrano-style release management helpers
#
# Expects DEPLOY_ROOT, RELEASES_DIR, CURRENT_LINK, and KEEP_RELEASES to
# already be exported by the caller (bin/deploy-tool does this before
# sourcing this file).

new_release_path() {
    echo "$RELEASES_DIR/$(date -u +%Y%m%d%H%M%S)"
}

list_releases() {
    # oldest to newest, one directory name per line
    # shellcheck disable=SC2012
    ls -1 "$RELEASES_DIR" 2>/dev/null | sort
}

current_release() {
    if [[ -L "$CURRENT_LINK" ]]; then
        basename "$(readlink "$CURRENT_LINK")"
    else
        echo ""
    fi
}

activate_release() {
    local release_name="$1"
    local target="$RELEASES_DIR/$release_name"
    [[ -d "$target" ]] || { log_error "release '$release_name' does not exist"; return 1; }

    # -f replaces an existing symlink, -n avoids dereferencing "current" if
    # it were ever a real directory — the same swap technique Capistrano uses
    ln -sfn "$target" "$CURRENT_LINK"
}

prune_releases() {
    local all_releases total to_remove
    all_releases=$(list_releases)
    total=$(echo "$all_releases" | grep -c . || true)

    if (( total <= KEEP_RELEASES )); then
        return 0
    fi

    to_remove=$(( total - KEEP_RELEASES ))
    echo "$all_releases" | head -n "$to_remove" | while IFS= read -r old; do
        [[ -z "$old" ]] && continue
        rm -rf "${RELEASES_DIR:?}/${old}"
        log_info "pruned old release: $old"
    done
}
```

## bin/deploy-tool

```bash
#!/usr/bin/env bash
# deploy-tool — a small Capistrano-style deployment automation CLI
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

export DEPLOY_ROOT="${DEPLOY_ROOT:-/opt/myapp}"
export KEEP_RELEASES="${KEEP_RELEASES:-5}"
RELEASES_DIR="$DEPLOY_ROOT/releases"
CURRENT_LINK="$DEPLOY_ROOT/current"

# shellcheck source=lib/logging.sh
source "$SCRIPT_DIR/lib/logging.sh"
# shellcheck source=lib/releases.sh
source "$SCRIPT_DIR/lib/releases.sh"

usage() {
    cat <<'EOF'
Usage: deploy-tool <command> [args]

Commands:
  deploy <source_dir>   ship the contents of source_dir as a new release
  rollback [n]            point "current" at the n-th previous release (default 1)
  status                    show the active release and recent history
  logs [-n N]              show the last N structured log lines (default 20)
  help                       show this help

Environment:
  DEPLOY_ROOT      base directory for releases/current (default /opt/myapp)
  KEEP_RELEASES    how many releases to retain (default 5)
  DEPLOY_LOG_FILE  structured log file path
EOF
}

# Module 09: refuse to run as root — deploys should use a dedicated,
# unprivileged service account. DEPLOY_ALLOW_ROOT exists only so the bats
# suite and CI (which often run as root in a container) can opt in.
require_least_privilege() {
    if [[ "$(id -u)" -eq 0 && "${DEPLOY_ALLOW_ROOT:-0}" != "1" ]]; then
        log_error "refusing to run as root — use the 'deploy' service account instead"
        exit 1
    fi
}

# Module 09: read a webhook token from a mode-600 secrets file, never from
# an argument or a hardcoded value. Silently skips if not configured.
notify_webhook() {
    local message="$1"
    local token_file="${DEPLOY_WEBHOOK_TOKEN_FILE:-$DEPLOY_ROOT/shared/webhook_token}"

    [[ -f "$token_file" ]] || { log_warn "no webhook token configured — skipping notification"; return 0; }

    local perms
    perms=$(stat -c '%a' "$token_file" 2>/dev/null || stat -f '%Lp' "$token_file")
    if [[ "$perms" != "600" && "$perms" != "400" ]]; then
        log_error "refusing to read '$token_file' — permissions are $perms, expected 600"
        return 1
    fi

    local token
    token=$(<"$token_file")
    curl -fsS -H "Authorization: Bearer ${token}" \
        -d "{\"text\":\"${message}\"}" \
        "${DEPLOY_WEBHOOK_URL:?DEPLOY_WEBHOOK_URL not set}" > /dev/null
}

cmd_deploy() {
    local source_dir="${1:-}"
    [[ -n "$source_dir" && -d "$source_dir" ]] || { log_error "deploy requires an existing source directory"; exit 1; }

    require_least_privilege
    mkdir -p "$RELEASES_DIR" "$DEPLOY_ROOT/shared"

    local release_path release_name
    release_path=$(new_release_path)
    release_name=$(basename "$release_path")
    log_info "starting deploy from '$source_dir' to '$release_path'"

    mkdir -p "$release_path"
    cp -a "$source_dir"/. "$release_path"/

    if [[ -x "$release_path/bin/post_deploy.sh" ]]; then
        log_info "running post_deploy hook"
        "$release_path/bin/post_deploy.sh"
    fi

    activate_release "$release_name"
    log_info "release $release_name is now live"

    prune_releases
    notify_webhook "deployed $release_name" || log_warn "notification failed, continuing"

    echo "deployed $release_name"
}

cmd_rollback() {
    local steps="${1:-1}"
    mapfile -t releases < <(list_releases)     # bash 4+; oldest first
    local current
    current=$(current_release)

    local current_index=-1
    for i in "${!releases[@]}"; do
        if [[ "${releases[$i]}" == "$current" ]]; then
            current_index=$i
            break
        fi
    done

    if (( current_index < 0 )); then
        log_error "could not determine the current release"
        exit 1
    fi

    local target_index=$(( current_index - steps ))
    if (( target_index < 0 )); then
        log_error "no release $steps step(s) back from '$current'"
        exit 1
    fi

    local target="${releases[$target_index]}"
    log_warn "rolling back from $current to $target"
    activate_release "$target"
    notify_webhook "rolled back to $target" || log_warn "notification failed, continuing"

    echo "rolled back to $target"
}

cmd_status() {
    local current
    current=$(current_release)
    echo "current release: ${current:-none}"
    echo "releases (oldest first):"
    list_releases | sed 's/^/  - /'
}

cmd_logs() {
    local n=20
    if [[ "${1:-}" == "-n" ]]; then
        n="$2"
    fi
    tail -n "$n" "$LOG_FILE" | jq -c '.'
}

main() {
    local command="${1:-help}"
    shift || true
    case "$command" in
        deploy)   cmd_deploy "$@" ;;
        rollback) cmd_rollback "$@" ;;
        status)   cmd_status "$@" ;;
        logs)     cmd_logs "$@" ;;
        help|-h|--help) usage ;;
        *) echo "deploy-tool: unknown command '$command' (see 'deploy-tool help')" >&2; exit 1 ;;
    esac
}

main "$@"
```

## tests/deploy-tool.bats

```bash
#!/usr/bin/env bats
# tests/deploy-tool.bats

setup() {
    export DEPLOY_ALLOW_ROOT=1                  # bats/CI may run as root — bypass the guard for tests
    TEST_ROOT=$(mktemp -d)
    export DEPLOY_ROOT="$TEST_ROOT/app"
    export DEPLOY_LOG_FILE="$TEST_ROOT/deploy-tool.log"
    export KEEP_RELEASES=2

    SRC_V1="$TEST_ROOT/src_v1"
    mkdir -p "$SRC_V1"
    echo "v1" > "$SRC_V1/VERSION"

    SRC_V2="$TEST_ROOT/src_v2"
    mkdir -p "$SRC_V2"
    echo "v2" > "$SRC_V2/VERSION"

    DEPLOY_TOOL="$BATS_TEST_DIRNAME/../bin/deploy-tool"
}

teardown() {
    rm -rf "$TEST_ROOT"
}

@test "deploy creates a release and points current at it" {
    run "$DEPLOY_TOOL" deploy "$SRC_V1"
    [ "$status" -eq 0 ]
    [ -L "$DEPLOY_ROOT/current" ]
    [ "$(cat "$DEPLOY_ROOT/current/VERSION")" = "v1" ]
}

@test "status reports the current release" {
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    run "$DEPLOY_TOOL" status
    [ "$status" -eq 0 ]
    [[ "$output" == *"current release:"* ]]
}

@test "a second deploy switches current to the new release" {
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    "$DEPLOY_TOOL" deploy "$SRC_V2" >/dev/null
    [ "$(cat "$DEPLOY_ROOT/current/VERSION")" = "v2" ]
}

@test "rollback switches current back to the previous release" {
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    sleep 1
    "$DEPLOY_TOOL" deploy "$SRC_V2" >/dev/null
    run "$DEPLOY_TOOL" rollback 1
    [ "$status" -eq 0 ]
    [ "$(cat "$DEPLOY_ROOT/current/VERSION")" = "v1" ]
}

@test "deploy rejects a nonexistent source directory" {
    run "$DEPLOY_TOOL" deploy "/nonexistent/path"
    [ "$status" -ne 0 ]
}

@test "old releases are pruned beyond KEEP_RELEASES" {
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    sleep 1
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    sleep 1
    "$DEPLOY_TOOL" deploy "$SRC_V1" >/dev/null
    release_count=$(find "$DEPLOY_ROOT/releases" -mindepth 1 -maxdepth 1 -type d | wc -l)
    [ "$release_count" -le 2 ]
}
```

## Dockerfile

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends bash coreutils findutils jq curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# a dedicated, unprivileged service account — Module 09's least-privilege pattern
RUN useradd --system --create-home --shell /usr/sbin/nologin deploy

WORKDIR /opt/deploy-tool
COPY bin/ ./bin/
COPY lib/ ./lib/
RUN chmod +x bin/deploy-tool

ENV PATH="/opt/deploy-tool/bin:${PATH}"
ENV DEPLOY_ROOT=/opt/myapp

RUN mkdir -p /opt/myapp/releases /var/log/deploy-tool && \
    chown -R deploy:deploy /opt/myapp /var/log/deploy-tool /opt/deploy-tool

USER deploy
ENTRYPOINT ["deploy-tool"]
CMD ["help"]
```

```bash
docker build -t deploy-tool:latest .
docker run --rm deploy-tool:latest status
# current release: none
# releases (oldest first):
```

## .github/workflows/ci.yml

```yaml
name: CI
on: [push, pull_request]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install shellcheck, bats, and jq
        run: sudo apt-get update && sudo apt-get install -y shellcheck bats jq

      - name: Lint with shellcheck
        run: shellcheck bin/deploy-tool lib/*.sh

      - name: Run bats tests
        run: bats tests/deploy-tool.bats

      - name: Build the Docker image
        run: docker build -t deploy-tool:ci .
```

Every push and pull request now gets the same two gates a human reviewer
would apply manually: does it pass lint, and do the behavioral tests still
pass — exactly the "fail the pipeline on a bad `set -x` habit" idea from
Level 3's git-hooks-and-CI module, just running against this project.

## Running it

```bash
export DEPLOY_ROOT=/opt/myapp
export DEPLOY_ALLOW_ROOT=1     # only for this local walkthrough — real deploys use a service account
sudo mkdir -p "$DEPLOY_ROOT" && sudo chown "$USER" "$DEPLOY_ROOT"

./bin/deploy-tool deploy ./my-app-build
# {"time":"2026-07-18T09:00:00Z","level":"info","pid":4210,"msg":"starting deploy from './my-app-build' to '/opt/myapp/releases/20260718090000'"}
# {"time":"2026-07-18T09:00:00Z","level":"info","pid":4210,"msg":"release 20260718090000 is now live"}
# deployed 20260718090000

./bin/deploy-tool status
# current release: 20260718090000
# releases (oldest first):
#   - 20260718090000

./bin/deploy-tool deploy ./my-app-build-v2
# deployed 20260718091500

./bin/deploy-tool rollback 1
# {"time":"2026-07-18T09:20:00Z","level":"warn","pid":4300,"msg":"rolling back from 20260718091500 to 20260718090000"}
# rolled back to 20260718090000

./bin/deploy-tool logs -n 3
# {"time":"2026-07-18T09:20:00Z","level":"warn","pid":4300,"msg":"rolling back from 20260718091500 to 20260718090000"}
```

## How every level shows up here

| Level | Where it's used |
|-------|-------------------|
| Level 1 — variables, functions, file tests, `set -euo pipefail` | throughout `bin/deploy-tool` and `lib/releases.sh` |
| Level 2 — arrays, `case` dispatch, environment configuration | `mapfile`/array lookup in `cmd_rollback`, subcommand `case` in `main` |
| Level 3 — `sed`/`awk` one-liners, CI shell steps, security basics | `sed 's/^/  - /'` in `cmd_status`, `.github/workflows/ci.yml` |
| Level 4 — CLI subcommands, structured logging, security hardening, jq pipelines | the whole tool: `lib/logging.sh`, `notify_webhook`, `cmd_logs`'s `jq -c` |

## Stretch goals

- Add a `deploy-tool diff` subcommand that shows what changed between the
  current release and the one before it (`diff -rq`).
- Make `activate_release` symlink-swap fully atomic on Linux using a
  temp-symlink-then-`mv` sequence, and document the BSD/macOS difference.
- Add a `--dry-run` flag to `deploy` (Level 1's backup-script stretch goal,
  applied here) that logs every step without touching the filesystem.
- Extend the CI workflow to actually run `tests/deploy-tool.bats` **inside**
  the built Docker image, so CI validates the exact artifact that ships.
- Add a `deploy-tool.json` machine-readable output mode (`--json` flag on
  `status`) so other tools can consume it with the `jq` pipelines from
  Module 06.

You've now built a CLI tool, hardened it, logged it, tested it, containerized
it, and wired it into CI — the same shape as real deployment tooling running
in production today. You've completed the Shell Mastery Path — Entry to Master.
