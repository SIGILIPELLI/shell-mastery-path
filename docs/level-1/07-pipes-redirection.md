---
description: "Pipes & Redirection — /dev/null is a special file that silently discards anything written to it — the standard way to suppress output you don't care about."
---

# 07 · Pipes & Redirection

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/sm0OOgTBPHY" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


## The three standard streams

Every process has three default file descriptors:

| Descriptor | Name | Number | Default destination |
|------------|------|--------|-----------------------|
| stdin | standard input | 0 | keyboard / whatever is piped in |
| stdout | standard output | 1 | terminal screen |
| stderr | standard error | 2 | terminal screen |

## Redirecting output

```bash
echo "hello" > out.txt        # write stdout to a file, OVERWRITING it
echo "world" >> out.txt        # append stdout to a file

cat out.txt
# hello
# world
```

```bash
ls /nonexistent 2> errors.txt      # redirect stderr only, to a file
ls /nonexistent 2>> errors.txt      # append stderr

ls . > out.txt 2> errors.txt         # stdout and stderr to separate files
ls . > combined.txt 2>&1              # BOTH stdout and stderr into one file
ls . &> combined.txt                   # bash shorthand for the line above
```

Order matters with `2>&1`: it must come **after** the stdout redirect, since
it means "point fd 2 at wherever fd 1 currently points."

```bash
some_command > /dev/null 2>&1     # discard all output entirely
```

`/dev/null` is a special file that silently discards anything written to it —
the standard way to suppress output you don't care about.

## Redirecting input

```bash
sort < unsorted.txt          # feed the file as stdin to `sort`

wc -l < names.txt              # count lines, without printing the filename
```

## Pipes — chaining commands together

A pipe (`|`) connects one command's stdout directly to the next command's
stdin, without touching disk:

```bash
ls -la | grep ".sh"                 # filter ls output for .sh files
cat access.log | grep "ERROR" | wc -l   # count ERROR lines in a log

ps aux | grep "python" | grep -v "grep"   # find python processes, exclude grep itself
```

```bash
history | tail -20 | sort          # last 20 commands, sorted
```

## Combining pipes and redirection

```bash
grep "ERROR" app.log | sort | uniq -c | sort -rn > error_summary.txt
```

This chain: filters ERROR lines, sorts them, counts unique occurrences,
sorts by count descending, and writes the final report to a file.

## `tee` — split output to a file AND the screen

```bash
echo "deployment started" | tee deploy.log
# prints to terminal AND writes to deploy.log

build_script.sh | tee -a deploy.log     # -a appends instead of overwriting
```

`tee` is invaluable for scripts where you want to both watch progress live
and keep a permanent log.

## Here-strings and quick input

```bash
grep "root" <<< "$(cat /etc/passwd)"    # feed a variable's content as stdin

wc -w <<< "count these words please"     # 4
```

(Full here-**documents**, the multi-line `<<EOF ... EOF` form, are covered in
Level 3's "Process Substitution & Here-Docs" module.)

## Command substitution

```bash
current_date=$(date +%F)
file_count=$(ls | wc -l)

echo "On $current_date there are $file_count files here"
```

`$( ... )` runs the command inside and substitutes its stdout output as text
— the modern, nestable replacement for the older backtick syntax
`` `command` ``.

## How It Actually Works

A pipe (`cmd1 | cmd2`) is built on the `pipe(2)` syscall, which gives the
shell a pair of connected file descriptors — one for writing, one for
reading — backed by a fixed-size kernel buffer (commonly 64KB on Linux). The
shell `fork()`s once per pipeline stage, and in each child rewires file
descriptor 1 (stdout) or 0 (stdin) onto the pipe's ends via `dup2(2)` before
`exec`ing the command. All stages then run **concurrently**, not
sequentially — `cmd1` blocks on `write()` once the kernel buffer fills, and
`cmd2` blocks on `read()` when the buffer is empty, so the pipeline behaves
like a bounded producer/consumer queue with the kernel doing the scheduling.

Redirection (`>`, `>>`, `<`, `2>&1`) works by manipulating file descriptors
*before* `exec`: `>` opens the target file (truncating it, `O_TRUNC`) and
`dup2`s it onto fd 1; `2>&1` duplicates whatever fd 1 currently points to
onto fd 2, which is why the *order* of redirections matters — `cmd > f 2>&1`
sends both to `f`, but `cmd 2>&1 > f` sends stderr to the terminal (fd 2
still pointed at fd 1's *old* target when the duplication happened) and only
stdout to `f`.

Command substitution `$(cmd)` also uses a pipe internally: bash forks a
subshell, redirects its stdout into the write end of an internal pipe, reads
everything from the other end into a buffer, and — critically — strips
trailing newlines from the captured string before substituting it into the
command line.


## Cheat sheet

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

## 🔀 See this in another language

- [PowerShell — Working with Files](https://sigilipelli.github.io/powershell-mastery-path/level-1/07-working-with-files/)
- [C++ — Classes & Objects Basics](https://sigilipelli.github.io/cpp-mastery-path/level-1/07-classes-objects/)
- [Swift — Structs & Classes Basics](https://sigilipelli.github.io/swift-mastery-path/level-1/07-structs-classes/)

## Exercise

Write `logscan.sh` that pipes the contents of a log file through `grep` to
find lines containing `"WARN"` or `"ERROR"` (hint: `grep -E "WARN|ERROR"`),
uses `tee` to save the matches to `issues.log` while also printing them to
the screen, and finally prints a count of how many matching lines were found.
