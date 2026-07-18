# 08 · Logging & Observability for Scripts

A script that fails silently at 3am is worse than one that fails loudly.
This module covers structured (JSON) logging, log levels and correlation
IDs, and wiring scripts into the monitoring, metrics, and alerting systems
that actually watch production.

## Why structured logging

Plain-text logs ("started backup at 3am") are fine for a human tailing a
file, but painful for a machine to parse reliably. Structured logs emit
one JSON object per line instead — every field is queryable without
fragile regexes.

```text
# plain text — easy to read, hard to query
2026-07-18 09:00:00 backup failed: disk full

# structured — one JSON object per line, trivially queryable with jq
{"time":"2026-07-18T09:00:00Z","level":"error","msg":"backup failed: disk full"}
```

## A structured logging function

```bash
LOG_FILE="${LOG_FILE:-/var/log/myscript.log}"

log() {
    local level="$1"; shift
    local message="$*"
    local timestamp
    timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    printf '{"time":"%s","level":"%s","script":"%s","pid":%d,"msg":"%s"}\n' \
        "$timestamp" "$level" "$(basename "$0")" "$$" "$message" \
        | tee -a "$LOG_FILE"
}

log_info()  { log "info"  "$@"; }
log_warn()  { log "warn"  "$@"; }
log_error() { log "error" "$@"; }

log_info "starting job"
log_error "failed to connect to database"
```

```text
{"time":"2026-07-18T09:00:00Z","level":"info","script":"myscript.sh","pid":4210,"msg":"starting job"}
{"time":"2026-07-18T09:00:01Z","level":"error","script":"myscript.sh","pid":4210,"msg":"failed to connect to database"}
```

## Log levels and filtering verbosity

```bash
LOG_LEVEL="${LOG_LEVEL:-info}"     # debug < info < warn < error

should_log() {
    local level="$1"
    case "$LOG_LEVEL" in
        debug) return 0 ;;
        info)  [[ "$level" != "debug" ]] ;;
        warn)  [[ "$level" == "warn" || "$level" == "error" ]] ;;
        error) [[ "$level" == "error" ]] ;;
    esac
}

log() {
    local level="$1"; shift
    should_log "$level" || return 0
    printf '{"time":"%s","level":"%s","msg":"%s"}\n' \
        "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$level" "$*" | tee -a "$LOG_FILE"
}
```

## Correlation IDs for tracing one run across log lines

```bash
RUN_ID="${RUN_ID:-$(uuidgen 2>/dev/null || date +%s%N)}"

log() {
    local level="$1"; shift
    printf '{"time":"%s","level":"%s","run_id":"%s","msg":"%s"}\n' \
        "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$level" "$RUN_ID" "$*" | tee -a "$LOG_FILE"
}
```

Exporting `RUN_ID` before calling downstream scripts (`export RUN_ID; ./step2.sh`)
lets every step of a multi-script job share the same ID, so a log
aggregator can join all of them into one trace by filtering on `run_id`.

## Querying structured logs with jq

```bash
jq 'select(.level == "error")' /var/log/myscript.log
jq -r 'select(.run_id == "abc123") | .msg' /var/log/myscript.log
```

## Sending logs to syslog

```bash
logger -t myscript "job started"                          # tags each line with "myscript" in syslog
logger -t myscript -p user.err "job failed: $reason"       # set a syslog priority

# view it
journalctl -t myscript -f
```

## Emitting metrics for a monitoring system

```bash
# Prometheus node_exporter "textfile collector" pattern — no long-running exporter needed
METRICS_FILE="/var/lib/node_exporter/textfile_collector/myscript.prom"

write_metric() {
    local name="$1" value="$2"
    echo "${name} ${value}" >> "${METRICS_FILE}.$$"
    mv "${METRICS_FILE}.$$" "$METRICS_FILE"    # atomic — never a half-written file
}

start_time=$(date +%s)
# ... do the actual work ...
end_time=$(date +%s)

write_metric "myscript_last_run_timestamp_seconds" "$end_time"
write_metric "myscript_duration_seconds" "$(( end_time - start_time ))"
write_metric "myscript_last_run_success" "1"
```

## Heartbeats and dead man's switches

```bash
# push a heartbeat — an external monitor alerts if this stops arriving on schedule
curl -fsS -m 10 "https://hc-ping.com/your-check-id" >/dev/null || true

# or simpler: touch a file a monitoring agent watches for staleness
touch /var/run/myscript.heartbeat
```

A "dead man's switch" flips the usual alerting logic: instead of alerting
when something goes wrong, it alerts when the expected success signal
*stops showing up* — the only reliable way to catch a cron job that
silently stopped running altogether.

## Alerting on failure automatically

```bash
notify_failure() {
    curl -fsS -X POST -H 'Content-Type: application/json' \
        -d "{\"text\":\"myscript failed — check logs (run_id=$RUN_ID)\"}" \
        "$ALERT_WEBHOOK_URL" >/dev/null || true
}

trap 'log_error "script failed at line $LINENO"; notify_failure' ERR
```

## Cheat sheet

| Technique | Purpose |
|-----------|---------|
| JSON log lines | machine-parseable structured logging |
| log levels (`debug`/`info`/`warn`/`error`) | control verbosity |
| a `run_id`/correlation ID | trace one execution across log lines/systems |
| `logger -t tag` | send lines to syslog/journald |
| textfile-collector `.prom` file | expose metrics to Prometheus without a daemon |
| heartbeat file/ping | detect when a scheduled job silently stopped running |
| `trap ... ERR` + webhook | alert immediately on failure |

## Exercise

Add structured JSON logging with `debug`/`info`/`warn`/`error` levels and a
`run_id` to Level 1's `backup.sh`. Then add a Prometheus textfile-collector
metric recording the backup's duration and success/failure, and a
`trap ... ERR` that posts a failure notification to a webhook URL read from
an environment variable.
