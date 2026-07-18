# 04 · DevOps Automation

Shell is the glue language of DevOps: it's what runs inside container
entrypoints and what stitches together the steps of a CI/CD pipeline.
This module covers writing robust Docker entrypoint scripts and
pipeline-scripting patterns that make builds fail fast, fail loud, and
stay debuggable.

## Anatomy of a Docker entrypoint script

An entrypoint script's job is to prepare the container's runtime
environment, then hand off to the real process — usually via `exec` so
the process becomes PID 1 and receives signals correctly.

```bash
#!/usr/bin/env bash
# entrypoint.sh
set -euo pipefail

echo "==> starting container for ${APP_NAME:-app}"

# wait for a dependency (e.g. a database) to be reachable before continuing
wait_for() {
    local host="$1" port="$2" retries=30
    until nc -z "$host" "$port" 2>/dev/null; do
        ((retries--)) || { echo "Timed out waiting for $host:$port" >&2; exit 1; }
        echo "waiting for $host:$port..."
        sleep 1
    done
}

wait_for "${DB_HOST:-db}" "${DB_PORT:-5432}"

# run one-time setup (migrations, config templating) before the main process
if [[ "${RUN_MIGRATIONS:-false}" == "true" ]]; then
    echo "==> running database migrations"
    ./manage.py migrate --noinput
fi

# hand off to the main process; exec replaces this script's PID so
# signals (SIGTERM from `docker stop`) reach the real app directly
exec "$@"
```

```dockerfile
# Dockerfile (relevant lines)
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
CMD ["gunicorn", "app:server", "--bind", "0.0.0.0:8000"]
```

The `CMD` becomes `"$@"` inside the entrypoint — this lets users override
the command at `docker run` time while still going through your setup
logic first.

## Why `exec "$@"` matters

```bash
# WRONG: the script stays as PID 1, the app runs as a child process;
# `docker stop` sends SIGTERM to the script, not the app, and the app
# may not shut down gracefully within the grace period
"$@"

# RIGHT: exec replaces the current process image entirely — the app
# becomes PID 1 and receives signals directly
exec "$@"
```

## Handling signals for graceful shutdown

```bash
#!/usr/bin/env bash
set -euo pipefail

cleanup() {
    echo "==> received shutdown signal, stopping gracefully"
    kill -TERM "$child_pid" 2>/dev/null
    wait "$child_pid"
}
trap cleanup SIGTERM SIGINT

"$@" &
child_pid=$!
wait "$child_pid"
```

This variant is needed when you must run cleanup logic *before* the main
process exits (flushing a queue, deregistering from a load balancer) —
otherwise prefer the simpler `exec "$@"` form above.

## Environment-driven configuration templating

Entrypoints often need to render a config file from environment
variables before the app starts:

```bash
# render nginx config from a template using envsubst
: "${BACKEND_HOST:?BACKEND_HOST must be set}"
: "${BACKEND_PORT:=8080}"

envsubst '${BACKEND_HOST} ${BACKEND_PORT}' \
    < /etc/nginx/nginx.conf.template \
    > /etc/nginx/nginx.conf

exec nginx -g 'daemon off;'
```

## CI/CD pipeline scripting

Pipeline scripts (whether invoked from GitHub Actions, GitLab CI, or
Jenkins) benefit from the same discipline as any other production
script: strict mode, clear failure messages, and steps that can be run
locally to reproduce a CI failure.

```bash
#!/usr/bin/env bash
# ci/run-tests.sh — callable identically from CI and from a dev machine
set -euo pipefail

echo "::group::Installing dependencies"
pip install -r requirements.txt -r requirements-dev.txt
echo "::endgroup::"

echo "::group::Linting"
if ! flake8 src/; then
    echo "::error::Linting failed — run 'flake8 src/' locally to see details"
    exit 1
fi
echo "::endgroup::"

echo "::group::Running tests"
pytest --tb=short --junitxml=test-results.xml
echo "::endgroup::"

echo "All checks passed."
```

`::group::`/`::endgroup::` are GitHub Actions log-folding markers — they
degrade harmlessly to plain text on other runners, so the script stays
portable.

## Failing fast and reporting clearly in pipelines

```bash
set -euo pipefail

run_step() {
    local name="$1"; shift
    echo "==> $name"
    if ! "$@"; then
        echo "FAILED: $name" >&2
        exit 1
    fi
}

run_step "unit tests"        pytest tests/unit
run_step "integration tests" pytest tests/integration
run_step "build image"       docker build -t myapp:ci .
```

A pipeline that names each step as it fails saves a debugging round-trip
compared to scrolling through raw tool output looking for the first error.

## Passing data between pipeline stages

CI systems often need a script to set output variables for later stages:

```bash
# GitHub Actions: write to $GITHUB_OUTPUT so later steps can read it
version=$(git describe --tags --always)
echo "version=$version" >> "$GITHUB_OUTPUT"
```

```bash
# generic pattern: write JSON artifacts other stages/tools can consume
jq -n --arg version "$version" --arg status "success" \
    '{version: $version, status: $status}' > build-metadata.json
```

## Cheat sheet

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

## Exercise

Write `entrypoint.sh` for a fictional web app: it should wait for a
`DB_HOST`/`DB_PORT` (default `db`/`5432`) to become reachable using a
`wait_for` function, fail with a clear error if `APP_SECRET` is unset,
print a startup banner, and finish with `exec "$@"` so it can be used as
a Docker `ENTRYPOINT` with any `CMD`. Test it locally by running
`./entrypoint.sh echo "app started"` both with and without `APP_SECRET`
set, and confirm the failure path exits non-zero with a clear message.
