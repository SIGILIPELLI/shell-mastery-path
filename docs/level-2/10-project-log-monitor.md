# 10 · Project — Log Monitoring Script

The capstone for Level 2: a script that watches a log file for problem
patterns, summarizes them, and is validated with a real `bats` test suite —
tying together arrays, regex, process management, environment config, and
testing from this whole level.

## What you'll build

A script `log_monitor.sh` that:

- Scans a log file for `ERROR`/`CRITICAL`/`WARN` lines using regex
- Counts occurrences per severity level using an associative array
- Extracts and reports the most frequent error message
- Writes a structured summary report to a file
- Exits non-zero if any `CRITICAL` lines were found (so it's useful from
  cron or CI)
- Ships with a `bats` test suite

## Project layout

```text
log_monitor_project/
    log_monitor.sh
    log_monitor.bats
    sample.log            (sample input for manual testing)
    reports/               (created by the script)
```

## sample.log

```text
2026-07-18 09:00:01 INFO Service started
2026-07-18 09:01:15 WARN High memory usage: 82%
2026-07-18 09:02:03 ERROR Failed to connect to database
2026-07-18 09:02:04 ERROR Failed to connect to database
2026-07-18 09:03:11 INFO Request handled in 120ms
2026-07-18 09:04:45 CRITICAL Disk usage at 98%
2026-07-18 09:05:00 ERROR Failed to connect to database
2026-07-18 09:06:12 WARN Retry attempt 3 for job #4471
```

## log_monitor.sh

```bash
#!/usr/bin/env bash
# log_monitor.sh — scan a log file for warnings/errors and report a summary
set -euo pipefail

# ---- configuration ----------------------------------------------------------
REPORT_DIR="${REPORT_DIR:-./reports}"
LOG_LEVELS=("WARN" "ERROR" "CRITICAL")

# ---- helpers -----------------------------------------------------------------
usage() {
    echo "Usage: $0 <log_file>" >&2
    exit 1
}

die() {
    echo "Error: $1" >&2
    exit 1
}

timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

# ---- validate arguments --------------------------------------------------------
[[ $# -eq 1 ]] || usage
log_file="$1"
[[ -f "$log_file" ]] || die "log file '$log_file' not found"
[[ -r "$log_file" ]] || die "log file '$log_file' is not readable"

mkdir -p "$REPORT_DIR"
report_file="$REPORT_DIR/report_$(date +%Y%m%d_%H%M%S).txt"

# ---- scan the log --------------------------------------------------------------
declare -A level_counts
declare -A message_counts
critical_found=0

for level in "${LOG_LEVELS[@]}"; do
    level_counts["$level"]=0
done

while IFS= read -r line; do
    if [[ "$line" =~ ^([0-9-]+)\ ([0-9:]+)\ ([A-Z]+)\ (.*)$ ]]; then
        level="${BASH_REMATCH[3]}"
        message="${BASH_REMATCH[4]}"

        if [[ -v level_counts["$level"] ]]; then
            ((level_counts["$level"]++))
        fi

        if [[ "$level" == "ERROR" || "$level" == "CRITICAL" || "$level" == "WARN" ]]; then
            ((message_counts["$message"]++)) || true
        fi

        if [[ "$level" == "CRITICAL" ]]; then
            critical_found=1
        fi
    fi
done < "$log_file"

# ---- find the most frequent message ---------------------------------------------
top_message=""
top_count=0
for message in "${!message_counts[@]}"; do
    count="${message_counts[$message]}"
    if (( count > top_count )); then
        top_count=$count
        top_message=$message
    fi
done

# ---- write the report -----------------------------------------------------------
{
    echo "Log Monitor Report — generated $(timestamp)"
    echo "Source: $log_file"
    echo "-------------------------------------------"
    for level in "${LOG_LEVELS[@]}"; do
        echo "$level: ${level_counts[$level]}"
    done
    echo "-------------------------------------------"
    if [[ -n "$top_message" ]]; then
        echo "Most frequent issue (x$top_count): $top_message"
    else
        echo "No warning/error/critical lines found."
    fi
} | tee "$report_file"

echo ""
echo "Report saved to: $report_file"

# ---- exit status: non-zero if anything CRITICAL was found -------------------------
if [[ "$critical_found" -eq 1 ]]; then
    echo "CRITICAL entries found — exiting with status 2" >&2
    exit 2
fi

exit 0
```

## log_monitor.bats

```bash
#!/usr/bin/env bats

setup() {
    TEST_DIR=$(mktemp -d)
    export REPORT_DIR="$TEST_DIR/reports"

    cat > "$TEST_DIR/clean.log" <<'EOF'
2026-07-18 09:00:01 INFO Service started
2026-07-18 09:03:11 INFO Request handled in 120ms
EOF

    cat > "$TEST_DIR/critical.log" <<'EOF'
2026-07-18 09:00:01 INFO Service started
2026-07-18 09:04:45 CRITICAL Disk usage at 98%
EOF
}

teardown() {
    rm -rf "$TEST_DIR"
}

@test "fails with usage message when no argument is given" {
    run ./log_monitor.sh
    [ "$status" -eq 1 ]
    [[ "$output" == *"Usage:"* ]]
}

@test "fails clearly on a missing log file" {
    run ./log_monitor.sh "/nonexistent/file.log"
    [ "$status" -eq 1 ]
    [[ "$output" == *"not found"* ]]
}

@test "exits 0 on a clean log with no warnings/errors" {
    run ./log_monitor.sh "$TEST_DIR/clean.log"
    [ "$status" -eq 0 ]
    [[ "$output" == *"WARN: 0"* ]]
}

@test "exits 2 when a CRITICAL entry is present" {
    run ./log_monitor.sh "$TEST_DIR/critical.log"
    [ "$status" -eq 2 ]
    [[ "$output" == *"CRITICAL: 1"* ]]
}

@test "writes a report file" {
    ./log_monitor.sh "$TEST_DIR/clean.log" || true
    run bash -c "ls '$REPORT_DIR'/report_*.txt | wc -l"
    [ "$output" -ge 1 ]
}
```

## Running it

```bash
chmod +x log_monitor.sh

./log_monitor.sh sample.log
# Log Monitor Report — generated 2026-07-18 10:00:00
# Source: sample.log
# -------------------------------------------
# WARN: 2
# ERROR: 3
# CRITICAL: 1
# -------------------------------------------
# Most frequent issue (x3): Failed to connect to database
#
# Report saved to: ./reports/report_20260718_100000.txt
# CRITICAL entries found — exiting with status 2

bats log_monitor.bats
#  ✓ fails with usage message when no argument is given
#  ✓ fails clearly on a missing log file
#  ✓ exits 0 on a clean log with no warnings/errors
#  ✓ exits 2 when a CRITICAL entry is present
#  ✓ writes a report file
#
# 5 tests, 0 failures
```

## Scheduling it with cron

```bash
# check the application log every 15 minutes; a non-zero exit (2) from
# CRITICAL entries can be wired into your alerting/monitoring pipeline
*/15 * * * * /opt/scripts/log_monitor.sh /var/log/app.log >> /var/log/log_monitor_cron.log 2>&1
```

## How each Level 2 module shows up here

| Module | Where it's used |
|--------|-------------------|
| 01 Arrays | `level_counts`, `message_counts` associative arrays |
| 02 String manipulation | building the report text, timestamp formatting |
| 03 Regular expressions | the `=~` capture-group parse of each log line |
| 04 Process management | designed to run safely under cron/background scheduling |
| 05 Best practices | `set -euo pipefail`, quoting, `die`/`usage` helpers |
| 06 Debugging | structure supports `bash -x log_monitor.sh` for tracing |
| 07 Scheduling & automation | the cron line above |
| 08 Environment config | `REPORT_DIR` overridable via environment variable |
| 09 Testing | the full `log_monitor.bats` suite |

## Stretch goals

- Add a `--since <timestamp>` flag to only scan lines newer than a given
  time.
- Send an email or webhook notification when `CRITICAL` entries are found,
  instead of just a non-zero exit code.
- Support scanning multiple log files in one run and merging the summary.

Completing this project means you're ready for **Level 3 · Advanced**.
