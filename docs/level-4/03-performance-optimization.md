# 03 · Performance Optimization

Shell scripts are usually glue, not compute — but a script that forks a
new process per line of a million-line file will feel like it's hung.
This module covers the three habits that matter most for shell script
speed: cutting unnecessary subshells, preferring built-ins over external
commands inside loops, and actually measuring before you optimize.

## Why subshells are expensive

Every `$(...)`, every pipeline stage, and every external command (`cat`,
`grep`, `awk`, ...) forks a new process. That's cheap once, but ruinous
inside a loop that runs thousands of times.

```bash
# SLOW: forks `wc` and a subshell on every iteration of a 10,000-line loop
for file in *.log; do
    lines=$(wc -l < "$file")
    echo "$file: $lines"
done
```

```bash
# FASTER: no subshell at all — bash reads the file itself
for file in *.log; do
    lines=0
    while IFS= read -r _; do
        ((lines++))
    done < "$file"
    echo "$file: $lines"
done
```

In this particular case `wc -l` is actually fine to call once per file
(it's not in a tight per-line loop) — the real trap is calling external
tools **per line**, shown next.

## Avoiding subshells inside per-line loops

```bash
# SLOW: forks `echo`, `tr`, and a pipeline on EVERY line
while IFS= read -r line; do
    upper=$(echo "$line" | tr '[:lower:]' '[:upper:]')
    echo "$upper"
done < data.txt
```

```bash
# FASTER: bash's own parameter expansion does the uppercase, no fork
while IFS= read -r line; do
    echo "${line^^}"
done < data.txt
```

```bash
# FASTEST for large files: let awk do the whole loop in one process
awk '{ print toupper($0) }' data.txt
```

The general rule: **the more lines you process, the more it pays to move
the loop itself into a single-process tool** (`awk`, `sed`) rather than
looping in bash and shelling out per line.

## Built-ins vs external commands

Bash has built-in equivalents for many common one-off external calls.
Built-ins run in the current process — no fork, no exec.

```bash
# external command (forks a process)
result=$(expr 3 + 4)

# built-in arithmetic (no fork)
result=$((3 + 4))
```

```bash
# external: basename/dirname fork a process each
name=$(basename "$path")
dir=$(dirname "$path")

# built-in: parameter expansion does the same with no fork
name="${path##*/}"
dir="${path%/*}"
```

```bash
# external: checking substrings with grep
if echo "$string" | grep -q "needle"; then

# built-in: bash pattern matching, no fork, no pipe
if [[ "$string" == *needle* ]]; then
```

## Efficient loops: read whole files, not line-by-line tools

```bash
# SLOW: spawns `cat` once per line via command substitution in a loop
total=0
for f in data/*.txt; do
    count=$(cat "$f" | wc -l)   # also an unnecessary `cat` (UUOC)
    total=$((total + count))
done
```

```bash
# FASTER: wc reads the file directly, one process per file (not per line)
total=0
for f in data/*.txt; do
    count=$(wc -l < "$f")
    total=$((total + count))
done
```

```bash
# FASTEST: one process total, for any number of files
total=$(cat data/*.txt | wc -l)
# or, avoiding even that cat:
total=$(wc -l data/*.txt | tail -1 | awk '{print $1}')
```

## Batch operations instead of per-item forks

```bash
# SLOW: one `rm` process per file
for f in *.tmp; do
    rm "$f"
done
```

```bash
# FASTER: one `rm` process total (relies on shell globbing expanding *.tmp)
rm -- *.tmp
```

```bash
# for very large argument lists that might exceed shell limits, use xargs
# with a sensible batch size instead of one process per item
find . -name "*.tmp" -print0 | xargs -0 -n 100 rm --
```

## Benchmarking scripts

Never optimize by guesswork — measure. Bash's built-in `time` and the
`TIMEFORMAT` variable are enough for most script-level benchmarking.

```bash
time ./my_script.sh
# real    0m2.341s   <- wall-clock time (what the user feels)
# user    0m1.998s   <- CPU time spent in your process
# sys     0m0.312s   <- CPU time spent in kernel calls on your behalf
```

```bash
# compare two approaches directly
TIMEFORMAT='%3R seconds'

time {
    for f in *.txt; do lines=$(wc -l < "$f"); done
}

time {
    wc -l *.txt >/dev/null
}
```

```bash
# a tiny reusable benchmark helper
benchmark() {
    local label="$1"; shift
    local start end
    start=$(date +%s.%N)
    "$@" >/dev/null
    end=$(date +%s.%N)
    printf "%-20s %.3fs\n" "$label" "$(echo "$end - $start" | bc)"
}

benchmark "loop+wc"  bash -c 'for f in *.log; do wc -l < "$f"; done'
benchmark "single-awk" awk '{c++} END{print c}' *.log
```

## Cheat sheet

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

## Exercise

Take a directory of at least 1,000 small text files and write two
versions of a script that counts total lines across all of them: one
that loops over the files calling `wc -l` in a subshell per file, and
one that passes all files to a single `wc -l` invocation. Wrap both in
`time` and compare `real` time. Then rewrite the slower version's
uppercase-conversion step (if you add one) to use `${var^^}` instead of
piping through `tr`, and re-measure.
