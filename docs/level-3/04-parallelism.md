# 04 · Parallelism

Level 2 introduced backgrounding jobs with `&` and waiting with `wait`. That
works for a handful of jobs, but running hundreds of tasks that way would
overwhelm the system — you need to control **how many** run at once. This
module covers `xargs -P` and GNU `parallel` for controlled concurrency.

## The problem with naive backgrounding

```bash
# BAD at scale — launches ALL 500 downloads simultaneously,
# exhausting file descriptors / bandwidth / memory
for url in $(cat urls.txt); do
    curl -s -O "$url" &
done
wait
```

## `xargs -P` — a controlled worker pool

```bash
# run up to 4 downloads at a time, from a file of URLs
xargs -P 4 -I {} curl -s -O {} < urls.txt
```

```bash
# -n 1 : pass one argument per invocation
# -P 4 : run at most 4 processes concurrently
cat hostnames.txt | xargs -n 1 -P 4 ping -c 1
```

```bash
# combine with find to process many files concurrently
find . -name "*.log" -print0 | xargs -0 -P 4 -I {} gzip {}
```

`-0`/`-print0` use NUL-separated names instead of newlines — the safe way
to handle filenames that might contain spaces or even newlines.

## Checking each job's result with `xargs`

```bash
xargs -P 4 -I {} bash -c 'curl -sf -o /dev/null "{}" && echo "OK: {}" || echo "FAIL: {}"' < urls.txt
```

Wrapping the command in `bash -c '...'` lets you use shell logic (`&&`,
`||`) per-item, since `xargs` itself only runs a single command with
arguments appended.

## GNU `parallel` — a more powerful alternative

```bash
# macOS
brew install parallel
```

```bash
parallel -j 4 curl -s -O {} :::: urls.txt
```

```bash
# apply to a range of numbers
parallel -j 8 'echo "processing item {}"; sleep 1' ::: {1..20}
```

```bash
# combine multiple input lists — parallel computes the cartesian product
parallel echo {1}-{2} ::: a b c ::: 1 2
# a-1
# a-2
# b-1
# b-2
# c-1
# c-2
```

| Tool | Strength |
|------|----------|
| `xargs -P` | ubiquitous (no install needed), good for simple fan-out |
| GNU `parallel` | richer templating, built-in progress bar (`--bar`), result logging (`--joblog`) |

## Tracking progress and failures with `parallel`

```bash
parallel --bar -j 4 --joblog job.log 'curl -sf -o /dev/null {} && echo ok || echo fail' :::: urls.txt

# job.log records exit status, runtime, and command for every job — useful
# for retrying just the failures afterward
awk '$7 != 0 { print $9 }' job.log     # print the commands that failed (column 7 = exit code)
```

## A manual worker-pool pattern (no external tools)

```bash
#!/usr/bin/env bash
set -euo pipefail

max_jobs=4

run_job() {
    local item="$1"
    echo "processing $item"
    sleep 1
}

for item in "${@}"; do
    run_job "$item" &

    # throttle: if we've hit the limit, wait for ANY one job to finish
    while (( $(jobs -r -p | wc -l) >= max_jobs )); do
        wait -n
    done
done

wait     # wait for any stragglers
echo "all jobs complete"
```

`wait -n` (bash 4.3+) waits for the **next** background job to finish
(rather than all of them) — combined with counting `jobs -r -p` (running
job PIDs), this throttles concurrency without any external dependency.

## When NOT to parallelize

```bash
# writes to a SHARED file from multiple parallel workers — a race condition!
parallel 'echo "{}" >> shared_output.txt' ::: a b c d
```

```bash
# safer: each worker writes its OWN file, merge afterward
parallel 'echo "{}" > "output_{#}.txt"' ::: a b c d
cat output_*.txt > merged.txt
rm output_*.txt
```

Concurrent writes to the same file (or the same lock, the same counter
variable in the parent shell) are a classic source of subtle, hard-to-
reproduce bugs — give each parallel worker its own output, and merge
afterward.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `xargs -P N` | run up to N processes concurrently |
| `xargs -0` | NUL-delimited input (safe with `find -print0`) |
| `parallel -j N` | GNU parallel, N concurrent jobs |
| `parallel --joblog file` | record exit status/runtime per job |
| `jobs -r -p \| wc -l` | count currently running background jobs |
| `wait -n` | wait for the next background job (not all) to finish |

## Exercise

Write `fetch_all.sh` that reads a list of URLs from a file, downloads each
with `curl` using `xargs -P 4`, and writes an `OK`/`FAIL` line per URL to a
results file. Then rewrite it a second way using GNU `parallel` with
`--joblog`, and compare the two approaches for a list of 20 URLs.
