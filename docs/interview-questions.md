---
title: "Shell Interview Questions and Answers (179 Questions)"
description: "179 Shell interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Shell Mastery Path."
---

# Shell Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Shell Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Script](level-1/01-setup.md)

**Q1.** What does the first line #!/usr/bin/env bash do?

**Answer:** Tells the OS which interpreter to run the script with, found via the PATH

The file also needs the executable bit (chmod +x).

**Q2.** Which command makes a script runnable as ./script.sh?

**Answer:** chmod +x script.sh

Without it you must run bash script.sh.

**Q3.** What is the difference between sh and bash?

**Answer:** sh is the POSIX shell interface (often dash or a bash in POSIX mode); bash adds extensions such as [[ ]] and arrays

Use a bash shebang when you use bash features.

**Q4.** Which tool statically checks shell scripts for common mistakes?

**Answer:** ShellCheck

Run it in CI.

### [02 · Variables &amp; Data Types](level-1/02-variables-data-types.md)

**Q5.** What does this print?

```bash
name='Ana Lee'
echo "$name" | wc -w | tr -d ' '
set -- $name; echo $#
set -- "$name"; echo $#
readonly PI=3; echo $PI
```

**Answer:** 2
2
1
3

Unquoted expansion word-splits; quoted keeps one argument. Verified by running it in Bash 5.

**Q6.** What does this print?

```bash
x=5; y="$x"3; echo $y
echo $((x * 2 + 1))
n=abc; echo $((n))
echo ${#n}
```

**Answer:** 53
11
0
3

Shell variables are strings; arithmetic treats a non-numeric word as 0. Verified by running it in Bash 5.

**Q7.** What is the correct way to assign a variable?

**Answer:** name=value (no spaces around =)

Spaces make it a command.

**Q8.** What does ${var:-default} do?

**Answer:** Uses default when var is unset or empty

${var:=default} also assigns it.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```bash
x=7
if [[ $x -gt 5 && $x -lt 10 ]]; then echo mid; elif [ "$x" -ge 10 ]; then echo hi; else echo lo; fi
case "hello" in h*) echo starts-h;; *) echo other;; esac
[ -z "" ] && echo empty
[ -e /nonexistent ] || echo missing
```

**Answer:** mid
starts-h
empty
missing

Test operators and case patterns. Verified by running it in Bash 5.

**Q10.** What is the difference between [ ] and [[ ]]?

**Answer:** [[ ]] is a bash keyword with pattern matching and no word splitting; [ ] is the POSIX test command

Quote variables in [ ].

**Q11.** What does an exit code of 0 mean in shell conditions?

**Answer:** Success (true)

if runs the then branch when the command exits 0.

**Q12.** Why quote "$var" in tests like [ "$var" = x ]?

**Answer:** An empty or space-containing value would otherwise break the test

Prevents word splitting and globbing.

### [04 · Loops](level-1/04-loops.md)

**Q13.** What does this print?

```bash
for i in 1 2 3; do [ $i -eq 2 ] && continue; echo -n "$i "; done; echo
i=0; while [ $i -lt 3 ]; do i=$((i+1)); done; echo $i
for f in *.nomatch; do echo "[$f]"; done
for ((k=0;k<3;k++)); do echo -n $k; done; echo
```

**Answer:** 1 3 
3
[*.nomatch]
012

Unmatched globs stay literal unless nullglob is set. Verified by running it in Bash 5.

**Q14.** What does break do?

**Answer:** Exits the innermost loop

continue skips to the next iteration.

**Q15.** What does while read -r line; do ...; done &lt; file do?

**Answer:** Reads the file line by line without mangling backslashes

Use IFS= to preserve whitespace.

**Q16.** Why avoid for f in $(ls)?

**Answer:** It breaks on spaces and special characters; use globs

for f in *.

### [05 · Functions &amp; Arguments](level-1/05-functions-arguments.md)

**Q17.** What does this print?

```bash
greet() { local who=${1:-world}; echo "hi $who"; }
greet; greet Ana
args() { echo "$# [$*] [$@]"; for a in "$@"; do echo "<$a>"; done; }
args 'a b' c
ret() { return 3; }; ret; echo $?
```

**Answer:** hi world
hi Ana
2 [a b c] [a b c]
&lt;a b&gt;
&lt;c&gt;
3

"$@" preserves each argument; $* joins them. Verified by running it in Bash 5.

**Q18.** What does local do?

**Answer:** Limits a variable's scope to the function

Prevents clobbering globals.

**Q19.** How does a function return data?

**Answer:** By printing it (captured with $(...)) and a numeric status via return

Return codes are 0-255.

**Q20.** What is $0?

**Answer:** The name of the script (or function context shell name)

$1 is the first argument.

### [06 · Working with Files &amp; Directories](level-1/06-files-directories.md)

**Q21.** What does this print?

```bash
d=$(mktemp -d); cd $d
mkdir -p a/b/c && touch a/b/c/f.txt a/x.log
find a -type f | sort
ls a | head -n 2 | tr '\n' ' '; echo
cp a/x.log a/y.log; mv a/y.log a/z.log; ls a | tr '\n' ' '; echo
rm -r a/b; [ -d a/b ] || echo gone
```

**Answer:** a/b/c/f.txt
a/x.log
b x.log 
b x.log z.log 
gone

mkdir -p creates parents; find lists files; mv renames. Verified by running it in Bash 5.

**Q22.** What does rm -rf $dir/ risk when dir is empty or unset?

**Answer:** It can expand to rm -rf / (use set -u and "${dir:?}")

Guard destructive commands.

**Q23.** What does mktemp -d give you?

**Answer:** A unique, safely created temporary directory

Clean up with trap.

**Q24.** What does test -f path check?

**Answer:** That path exists and is a regular file

-d is directory.

### [07 · Pipes &amp; Redirection](level-1/07-pipes-redirection.md)

**Q25.** What does this print?

```bash
printf 'b\na\nb\n' | sort | uniq -c | awk '{print $2":"$1}'
echo hi > f; echo there >> f; cat f | tr '\n' ' '; echo
ls /nonexistent 2>/dev/null; echo "rc=$?"
printf 'x\ny\n' | tee out.txt | wc -l | tr -d ' '
```

**Answer:** a:1
b:2
hi there 
rc=1
2

&gt; truncates, &gt;&gt; appends, 2&gt;/dev/null hides errors but not the exit code. Verified by running it in Bash 5.

**Q26.** What does 2&gt;&amp;1 do?

**Answer:** Sends stderr to the same place as stdout

Order matters: cmd &gt;file 2&gt;&amp;1.

**Q27.** What does a pipe connect?

**Answer:** The stdout of one command to the stdin of the next

Each stage runs concurrently.

**Q28.** What does command &gt; file do to an existing file?

**Answer:** Truncates it first (use &gt;&gt; to append)

set -o noclobber protects.

### [08 · Basic Text Processing](level-1/08-text-processing.md)

**Q29.** What does this print?

```bash
printf 'ana,30,NY\nbo,25,LA\ncy,41,NY\n' > d.csv
awk -F, '$3=="NY" {s+=$2; n++} END {print s/n}' d.csv
cut -d, -f1 d.csv | paste -sd' ' -
sed 's/NY/New York/' d.csv | head -1
grep -c ',NY' d.csv
tr a-z A-Z <<< hello
```

**Answer:** 35.5
ana bo cy
ana,30,New York
2
HELLO

awk, cut, sed, grep and tr cover most text tasks. Verified by running it in Bash 5.

**Q30.** What does grep -v do?

**Answer:** Prints lines that do NOT match

-i ignores case.

**Q31.** When use awk instead of cut?

**Answer:** When you need conditions, arithmetic or variable-width separators

awk is a mini language.

**Q32.** What does sed -n '2,4p' do?

**Answer:** Prints only lines 2 to 4

-n suppresses automatic printing.

### [09 · Exit Codes &amp; Basic Error Handling](level-1/09-exit-codes-error-handling.md)

**Q33.** What does this print?

```bash
ok() { return 0; }; bad() { return 2; }
ok && echo A; bad || echo "B$?"
false; echo $?
( false; echo not ) || echo sub
cmd=notacommand; $cmd 2>/dev/null; echo $?
```

**Answer:** A
B2
1
not
127

127 means command not found; $? holds the last status; a subshell's status is that of its last command (echo succeeded), so || does not run. Verified by running it in Bash 5.

**Q34.** What does set -u do?

**Answer:** Treats unset variables as errors

Catches typos.

**Q35.** What does set -o pipefail do?

**Answer:** Makes a pipeline fail if any stage fails, not just the last

Combine with -e and -u.

**Q36.** What does trap 'cleanup' EXIT do?

**Answer:** Runs cleanup when the script exits, however it exits

Use for temp files.

### [10 · Project — Backup Script](level-1/10-project-backup-script.md)

**Q37.** What is the aim of the backup script project?

**Answer:** A script that copies files into dated archives with logging, error handling and cleanup of old backups

Combine the level.

**Q38.** What does this print?

```bash
d=$(mktemp -d); cd $d; mkdir src; echo data > src/a.txt
stamp=2026-01-02
tar -czf backup-$stamp.tar.gz src && echo made
tar -tzf backup-$stamp.tar.gz | sort | tr '\n' ' '; echo
[ -s backup-$stamp.tar.gz ] && echo nonempty
```

**Answer:** made
src/ src/a.txt 
nonempty

tar lists directories too; -s tests for a non-empty file. Verified by running it in Bash 5.

**Q39.** Why verify a backup after creating it?

**Answer:** A corrupt or empty archive is useless when you need it

List or test the archive.

**Q40.** How should old backups be pruned?

**Answer:** Keep a defined number or age and log what is removed

Retention policy.

## Level 2 · Intermediate

### [01 · Arrays &amp; Associative Arrays](level-2/01-arrays.md)

**Q41.** What does this print?

```bash
arr=(one two 'three four')
echo ${#arr[@]}
arr+=(five)
echo "${arr[2]}" "${arr[-1]}"
for x in "${arr[@]:1:2}"; do echo "[$x]"; done
declare -A cap=([fr]=Paris [de]=Berlin)
echo ${cap[de]} ${#cap[@]}
```

**Answer:** 3
three four five
[two]
[three four]
Berlin 2

Quote "${arr[@]}" to keep elements intact; associative arrays need declare -A (bash 4+). Verified by running it in Bash 5.

**Q42.** What does "${arr[@]}" do?

**Answer:** Expands to each element as a separate word, preserving spaces

"${arr[*]}" joins them.

**Q43.** What does ${!arr[@]} give?

**Answer:** The indexes (or keys) of the array

Useful for associative arrays.

**Q44.** Which macOS default bash lacks associative arrays?

**Answer:** Bash 3.2 (install a newer bash)

Check bash --version.

### [02 · String Manipulation](level-2/02-string-manipulation.md)

**Q45.** What does this print?

```bash
s='Hello, World'
echo ${#s} ${s^^} ${s,,}
echo ${s:7:5} ${s/World/Bash} ${s%%,*} ${s#*, }
f=report.final.tar.gz; echo ${f%%.*} ${f%.*} ${f##*.}
printf '%05.1f|%-6s|%x\n' 3.14159 ab 255
```

**Answer:** 12 HELLO, WORLD hello, world
World Hello, Bash Hello World
report report.final.tar gz
003.1|ab    |ff

Parameter expansion trims prefixes and suffixes without external tools. Verified by running it in Bash 5.

**Q46.** What do ${var%pattern} and ${var#pattern} remove?

**Answer:** % removes the shortest suffix match; # removes the shortest prefix match

Doubled (%% ##) remove the longest.

**Q47.** Why is parameter expansion preferred over calling sed or cut in a loop?

**Answer:** It avoids spawning a process per line

Faster and simpler.

**Q48.** What does printf '%s\n' "$var" avoid compared with echo?

**Answer:** echo's inconsistent handling of backslashes and leading dashes

printf is portable.

### [03 · Regular Expressions in Bash](level-2/03-regular-expressions.md)

**Q49.** What does this print?

```bash
[[ 'abc123' =~ ^([a-z]+)([0-9]+)$ ]] && echo "${BASH_REMATCH[1]} ${BASH_REMATCH[2]}"
echo 'a1b22c333' | grep -oE '[0-9]+' | tr '\n' ' '; echo
echo 'foo bar' | sed -E 's/(foo) (bar)/\2 \1/'
[[ 'file.txt' == *.txt ]] && echo glob
echo 'x@y.com' | grep -cE '^[^@]+@[^@]+\.[a-z]+$'
```

**Answer:** abc 123
1 22 333 
bar foo
glob
1

=~ uses POSIX ERE and fills BASH_REMATCH; globs are different from regexes. Verified by running it in Bash 5.

**Q50.** What is the difference between a glob and a regex?

**Answer:** Globs match filenames with * and ?; regexes match text patterns with a richer syntax

Different languages.

**Q51.** What does grep -E enable?

**Answer:** Extended regular expressions (+, ?, |, ())

-F is fixed strings.

**Q52.** Why quote regex patterns?

**Answer:** The shell would otherwise expand * and ? as globs first

Use single quotes.

### [04 · Process Management](level-2/04-process-management.md)

**Q53.** What does this print?

```bash
sleep 0.2 & pid=$!
wait $pid; echo "rc=$?"
(exit 3) & wait $!; echo "rc=$?"
sleep 5 & p=$!; kill $p; wait $p 2>/dev/null; echo "killed=$?"
jobs | wc -l | tr -d ' '
```

**Answer:** rc=0
rc=3
killed=143
0

$! is the last background PID; a SIGTERM kill gives status 128+15 = 143. Verified by running it in Bash 5.

**Q54.** What does &amp; at the end of a command do?

**Answer:** Runs it in the background

Use wait to join.

**Q55.** What signal does kill send by default?

**Answer:** SIGTERM (15), which can be handled

kill -9 cannot be caught.

**Q56.** What does nohup do?

**Answer:** Keeps a command running after the terminal closes

Output goes to nohup.out.

### [05 · Scripting Best Practices](level-2/05-best-practices.md)

**Q57.** What does this print?

```bash
set -u
f() { local d; d=$(mktemp -d) || return 1; trap 'rm -rf "$d"' RETURN; echo "$d" > path.txt; [ -d "$d" ] && echo created; }
f
[ -d "$(cat path.txt)" ] || echo cleaned
x="a b"; printf '[%s]\n' $x "$x"
```

**Answer:** created
cleaned
[a]
[b]
[a b]

Unquoted $x splits into two arguments; quoted stays one. A RETURN trap cleans up. Verified by running it in Bash 5.

**Q58.** Why use set -euo pipefail?

**Answer:** Fail fast on errors, unset variables and broken pipelines

But understand its exceptions.

**Q59.** What do you do instead of parsing ls output?

**Answer:** Use globs or find -print0 with read -d ''

Filenames may contain spaces and newlines.

**Q60.** Which naming convention avoids clobbering environment variables?

**Answer:** Lowercase for locals and uppercase for exported constants

Style guides agree.

### [06 · Debugging Scripts](level-2/06-debugging.md)

**Q61.** What does bash -x script.sh do?

**Answer:** Prints each command as it runs, with expansions

set -x inside scripts.

**Q62.** What does bash -n script.sh do?

**Answer:** Checks syntax without executing

A quick lint.

**Q63.** What does this print with a custom PS4?

```bash
PS4='+ '; exec 2>&1; set -x; a=1; b=$((a+1)); set +x; echo $b
```

**Answer:** + a=1
+ b=2
+ set +x
2

set -x traces each command to stderr, including the set +x that turns it off. Verified by running it in Bash 5.

**Q64.** What helps pinpoint the failing line?

**Answer:** trap 'echo failed at line $LINENO' ERR

Also use set -x selectively.

### [07 · Scheduling &amp; Automation](level-2/07-scheduling-automation.md)

**Q65.** What does the crontab entry 0 3 * * 0 mean?

**Answer:** At 03:00 every Sunday

Fields: minute hour dom month dow.

**Q66.** Why do cron jobs need absolute paths?

**Answer:** Cron runs with a minimal PATH and working directory

Set PATH or use full paths.

**Q67.** What does flock add to a scheduled job?

**Answer:** Prevents overlapping runs

flock -n lockfile cmd.

**Q68.** Where do cron job outputs go by default?

**Answer:** Emailed to the user (or lost); redirect to a log

&gt;&gt; log 2&gt;&amp;1.

### [08 · Environment &amp; Configuration Management](level-2/08-environment-config.md)

**Q69.** What does this print?

```bash
export APP_ENV=prod
env_file=cfg.env; printf 'A=1\nB=two words\n# comment\n' > $env_file
while IFS='=' read -r k v; do [[ $k =~ ^[A-Z_]+$ ]] && export "$k=$v"; done < $env_file
echo "$A|$B|$APP_ENV"
bash -c 'echo ${A:-unset}'
unset A; echo ${A:-gone}
```

**Answer:** 1|two words|prod
1
gone

Exported variables are inherited by child processes; unset removes them. Verified by running it in Bash 5.

**Q70.** What is the difference between a shell variable and an environment variable?

**Answer:** Environment variables are exported to child processes

export marks them.

**Q71.** Where should secrets for scripts come from?

**Answer:** The environment or a secrets manager, never hard-coded

Mind ps and shell history.

**Q72.** What does source file.sh do?

**Answer:** Runs the file's commands in the current shell, so variables persist

Also written as . file.sh.

### [09 · Testing Shell Scripts](level-2/09-testing-shell-scripts.md)

**Q73.** What is a unit test for a shell function?

**Answer:** Source the script and call the function with controlled inputs, asserting output and status

bats makes it tidy.

**Q74.** What is bats?

**Answer:** A Bash Automated Testing System

Tests are @test blocks.

**Q75.** What does this tiny test harness print?

```bash
assert_eq() { [ "$1" = "$2" ] && echo pass || echo "FAIL: expected $2 got $1"; }
add() { echo $(($1 + $2)); }
assert_eq "$(add 2 3)" 5
assert_eq "$(add 2 3)" 6
```

**Answer:** pass
FAIL: expected 6 got 5

Test status and output. Verified by running it in Bash 5.

**Q76.** What should tests avoid?

**Answer:** Touching real files, networks or the real system

Use mktemp and fakes.

### [10 · Project — Log Monitoring Script](level-2/10-project-log-monitor.md)

**Q77.** What does this log summariser print?

```bash
printf '2026-01-01 INFO ok\n2026-01-01 ERROR db down\n2026-01-02 ERROR db down\n2026-01-02 WARN slow\n' > app.log
awk '{c[$2]++} END {for (l in c) print l, c[l]}' app.log | sort
grep ERROR app.log | cut -d' ' -f3- | sort | uniq -c | awk '{print $1, $2, $3}'
tail -n 1 app.log
```

**Answer:** ERROR 2
INFO 1
WARN 1
2 db down
2026-01-02 WARN slow

A small pipeline for log triage. Verified by running it in Bash 5.

**Q78.** How would a log monitor avoid re-alerting on the same line?

**Answer:** Track the last read position or use a state file

Offset or inode tracking.

**Q79.** What should a monitor do on a threshold breach?

**Answer:** Send one clear alert with context and a rate limit

Avoid alert storms.

**Q80.** What does tail -F do that -f does not?

**Answer:** Follows by name, surviving log rotation

Useful for rotated logs.

## Level 3 · Advanced

### [01 · Advanced sed &amp; awk](level-3/01-advanced-sed-awk.md)

**Q81.** What does this print?

```bash
printf '1 a\n2 b\n3 c\n' > f
sed -n '2p' f
sed -E 's/^([0-9]) (.)$/\2\1/' f | tr '\n' ' '; echo
sed '/2/d' f | wc -l | tr -d ' '
awk 'NR>1 {print $2}' f | paste -sd, -
awk '{a[NR]=$2} END {for (i=NR;i>0;i--) printf "%s", a[i]; print ""}' f
```

**Answer:** 2 b
a1 b2 c3 
2
b,c
cba

sed handles line edits; awk handles fields and state. Verified by running it in Bash 5.

**Q82.** What does the awk BEGIN block run?

**Answer:** Before reading any input

END runs after the last line.

**Q83.** What does sed -i do?

**Answer:** Edits the file in place (BSD sed needs an argument like -i '')

Portability gotcha.

**Q84.** When is awk better than sed?

**Answer:** When you need fields, variables or arithmetic

Different tools.

### [02 · Process Substitution &amp; Here-Docs](level-3/02-process-substitution-heredocs.md)

**Q85.** What does this print?

```bash
diff <(printf 'a\nb\n') <(printf 'a\nc\n') | grep -c '^[<>]'
while read -r l; do echo "got $l"; done <<EOF
one
two
EOF
cat <<'EOF'
$HOME stays literal
EOF
mapfile -t lines < <(printf 'x\ny\n'); echo ${#lines[@]}
```

**Answer:** 2
got one
got two
$HOME stays literal
2

&lt;(...) is process substitution; a quoted here-doc delimiter disables expansion. Verified by running it in Bash 5.

**Q86.** What does &lt;(command) provide?

**Answer:** A file-like path containing the command's output

Avoids temp files.

**Q87.** Why use mapfile or while read instead of for i in $(...)?

**Answer:** They preserve whole lines and avoid word splitting

Safer parsing.

**Q88.** What is a here-document?

**Answer:** Inline multi-line input fed to a command's stdin

Quote the delimiter to prevent expansion.

### [03 · Signal Handling](level-3/03-signal-handling.md)

**Q89.** What does this print?

```bash
cleanup() { echo cleanup; }
trap cleanup EXIT
trap 'echo got-term; exit 4' TERM
sleep 5 & p=$!
( sleep 0.1; kill -TERM $$ ) &
wait
echo not-reached
```

**Answer:** got-term
cleanup

A handled TERM runs its trap, then exit triggers the EXIT trap. Verified by running it in Bash 5.

**Q90.** Which signal can't be trapped?

**Answer:** SIGKILL (and SIGSTOP)

Use SIGTERM for graceful stops.

**Q91.** What does trap '' INT do?

**Answer:** Ignores Ctrl-C

Use with care.

**Q92.** Why set traps for cleanup?

**Answer:** Temp files and locks are released even on failure or interruption

EXIT, INT and TERM.

### [04 · Parallelism](level-3/04-parallelism.md)

**Q93.** What does this print?

```bash
start=$SECONDS
for i in 1 2 3; do ( sleep 0.3; echo "job$i" ) & done | sort
wait
[ $((SECONDS - start)) -lt 2 ] && echo parallel
printf '1\n2\n3\n' | xargs -P 3 -I{} sh -c 'echo run{}' | sort | tr '\n' ' '; echo
```

**Answer:** job1
job2
job3
parallel
run1 run2 run3

Background jobs and xargs -P run work concurrently. Verified by running it in Bash 5.

**Q94.** What does xargs -P 4 do?

**Answer:** Runs up to four processes at once

Limit concurrency.

**Q95.** Why limit parallelism?

**Answer:** To avoid exhausting CPU, memory or remote services

Use a pool size.

**Q96.** What does GNU parallel add?

**Answer:** Job control, output grouping and retries

Optional tool.

### [05 · Writing Robust Production Scripts](level-3/05-robust-production-scripts.md)

**Q97.** What does this robust-script skeleton print?

```bash
set -euo pipefail
IFS=$'\n\t'
tmp=$(mktemp -d); trap 'rm -rf "$tmp"' EXIT
die() { echo "error: $*" >&2; exit 2; }
(
  [ -n "${REQUIRED:-}" ] || die 'REQUIRED not set'
) 2>&1 || echo "rc=$?"
REQUIRED=x; echo ok
```

**Answer:** error: REQUIRED not set
rc=2
ok

Fail clearly, with a non-zero status, on bad input. Verified by running it in Bash 5.

**Q98.** What should a production script log?

**Answer:** What it does, with timestamps, and errors to stderr

Helps operations.

**Q99.** Why validate inputs at the top of a script?

**Answer:** Fail early before doing damage

Check args and paths.

**Q100.** What is a lock file used for?

**Answer:** Preventing concurrent runs of the same job

Use flock or mkdir.

### [06 · Integrating with Git Hooks &amp; CI Scripts](level-3/06-git-hooks-ci.md)

**Q101.** What does a pre-commit hook do?

**Answer:** Runs checks before a commit is created and can block it

Located in .git/hooks or managed by a tool.

**Q102.** What should a CI shell step do on failure?

**Answer:** Exit non-zero so the job fails

set -e.

**Q103.** Why keep hook scripts fast?

**Answer:** Slow hooks tempt developers to skip them

Check only changed files.

**Q104.** Can client-side hooks be relied on for enforcement?

**Answer:** No, they can be bypassed; enforce on the server or in CI

Use CI as the gate.

### [07 · Security Considerations](level-3/07-security-considerations.md)

**Q105.** What is the risk of eval "$user_input"?

**Answer:** Arbitrary command injection

Never eval untrusted input.

**Q106.** How do you safely pass untrusted values to commands?

**Answer:** Quote them and use -- to end option parsing

cmd -- "$arg".

**Q107.** Why avoid putting secrets in command-line arguments?

**Answer:** They are visible in ps and shell history

Use environment or stdin.

**Q108.** What does umask 077 do?

**Answer:** New files are private to the owner

Set before creating secrets.

### [08 · Packaging &amp; Distributing Scripts](level-3/08-packaging-distributing.md)

**Q109.** What makes a script easy to distribute?

**Answer:** A shebang, usage help, versioning and an install path

Support --help.

**Q110.** How can you package a script?

**Answer:** As a single file, a tarball or a package (brew, deb)

Choose by audience.

**Q111.** What does shellcheck catch?

**Answer:** Quoting, portability and logic mistakes

Run in CI.

**Q112.** What should install scripts avoid?

**Answer:** Requiring root unnecessarily or piping unknown code to sh

Least privilege.

### [09 · Networking from the Shell](level-3/09-networking.md)

**Q113.** Which tool tests whether a port is open?

**Answer:** nc -z host port (or bash /dev/tcp)

Add a timeout.

**Q114.** What does this print?

```bash
code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 2 http://127.0.0.1:1 || true); echo "code=${code:-none}"
(exec 3<>/dev/tcp/127.0.0.1/1) 2>/dev/null && echo open || echo closed
```

**Answer:** code=000
closed

curl reports 000 when the connection fails; /dev/tcp tests a port. Verified by running it in Bash 5.

**Q115.** Why add --max-time or timeouts to network calls in scripts?

**Answer:** A hung call would block the script forever

Also use --retry.

**Q116.** What does curl -fsS do?

**Answer:** Fails on HTTP errors, stays quiet but shows errors

Good defaults in scripts.

### [10 · Project — CLI Tool with Git Hook Integration](level-3/10-project-cli-tool.md)

**Q117.** What does a well-designed CLI tool provide?

**Answer:** Usage help, options, exit codes and predictable output

Follow Unix conventions.

**Q118.** What does getopts do?

**Answer:** Parses short options in a script

For long options write a loop.

**Q119.** Where should errors go?

**Answer:** Standard error with a non-zero exit code

Keep stdout clean for pipes.

**Q120.** What does the project demonstrate?

**Answer:** Combining parsing, functions, error handling and tests in one tool

A capstone for Level 3.

## Level 4 · Master

### [01 · Building Full CLI Tools](level-4/01-building-cli-tools.md)

**Q121.** What does --help output help with?

**Answer:** Discoverability for users

Describe options and examples.

**Q122.** What is a subcommand style CLI?

**Answer:** One command with verbs such as tool add, tool list

Like git.

**Q123.** What does this tiny dispatcher print?

```bash
cmd_add() { echo "add $*"; }
cmd_list() { echo list; }
main() { local c=${1:-help}; shift || true; if declare -F "cmd_$c" >/dev/null; then "cmd_$c" "$@"; else echo "unknown: $c"; return 1; fi; }
main add x y; main list; main nope || echo "rc=$?"
```

**Answer:** add x y
list
unknown: nope
rc=1

declare -F tests whether a function exists. Verified by running it in Bash 5.

**Q124.** What should a CLI do when given bad arguments?

**Answer:** Print a usage message to stderr and exit non-zero

Exit code 2 is conventional.

### [02 · Advanced Scripting Patterns](level-4/02-advanced-patterns.md)

**Q125.** What does this print?

```bash
declare -A seen; out=()
for w in a b a c b; do [[ -n ${seen[$w]:-} ]] || { seen[$w]=1; out+=("$w"); }; done
echo "${out[*]}"
name=ref; declare -n r=name; r=changed; echo $name
f() { local -n arr=$1; arr+=(z); }; list=(x); f list; echo "${list[*]}"
```

**Answer:** a b c
changed
x z

Associative arrays dedupe; namerefs (declare -n) alias variables. Verified by running it in Bash 5.

**Q126.** What does coproc do?

**Answer:** Runs a command as a coprocess with pipes to talk to it

Advanced.

**Q127.** What is a safe way to handle filenames with newlines?

**Answer:** find -print0 | while IFS= read -r -d '' f

NUL-delimited.

**Q128.** When should you rewrite a shell script in another language?

**Answer:** When it needs complex data structures, large logic or heavy testing

Pick the right tool.

### [03 · Performance Optimization](level-4/03-performance-optimization.md)

**Q129.** What does this timing comparison show?

```bash
n=2000
s=$SECONDS; for ((i=0;i<n;i++)); do x=$((i%7)); done; a=$((SECONDS-s))
s=$SECONDS; for ((i=0;i<300;i++)); do x=$(echo $((i%7))); done; b=$((SECONDS-s))
echo builtin-fast
[ "$a" -le "$b" ] && echo ok
```

**Answer:** builtin-fast
ok

Builtins avoid forking processes. (Only the relative result is checked here, not exact timings.) Verified by running it in Bash 5.

**Q130.** What is the biggest shell performance cost?

**Answer:** Forking external processes in loops

Use builtins and one awk pass.

**Q131.** Why prefer one awk call over many cut/grep calls per line?

**Answer:** One process instead of thousands

Process spawn dominates.

**Q132.** What does time give you?

**Answer:** Elapsed, user and system time of a command

Measure before optimising.

### [04 · DevOps Automation](level-4/04-devops-automation.md)

**Q133.** What is infrastructure automation with shell good for?

**Answer:** Glue: orchestrating CLIs, files and services

Keep it simple.

**Q134.** Why make deploy scripts idempotent?

**Answer:** Safe to re-run after partial failure

Check before change.

**Q135.** What does this idempotent step print?

```bash
f=$(mktemp)
for i in 1 2 3; do grep -qx 'listen 80' "$f" || echo 'listen 80' >> "$f"; done
wc -l < "$f" | tr -d ' '
mkdir -p x/y; mkdir -p x/y; ln -sfn x link; ln -sfn x link; ls -l link | awk '{print $NF}'
```

**Answer:** 1
x

Check-then-write and -p/-f options make steps repeatable. Verified by running it in Bash 5.

**Q136.** How should automation report failures?

**Answer:** Non-zero exit codes and clear messages for the pipeline

CI relies on status.

### [05 · Cross-Shell Compatibility](level-4/05-cross-shell-compatibility.md)

**Q137.** What is POSIX sh?

**Answer:** The portable shell standard, without bash extensions

Use it for maximum portability.

**Q138.** Which is NOT POSIX sh?

**Answer:** [[ ]] and arrays

Bash extensions.

**Q139.** What does this print in POSIX-style code?

```bash
f() { [ "$1" = a ] && echo yes || echo no; }
f a; f b
set -- x y; for a in "$@"; do printf '%s\n' "$a"; done | tr '\n' ' '; echo
i=0; while [ "$i" -lt 3 ]; do i=$((i+1)); done; echo $i
```

**Answer:** yes
no
x y 
3

[ ], case, $(( )) and printf are POSIX. Verified by running it in Bash 5.

**Q140.** How do you stay portable between BSD and GNU tools?

**Answer:** Avoid GNU-only flags or detect and adapt (for example sed -i, date -d)

Test on both.

### [06 · Advanced Text Processing Pipelines](level-4/06-text-processing-pipelines.md)

**Q141.** What does this pipeline print?

```bash
printf 'GET /a 200\nGET /b 500\nPOST /a 200\nGET /a 404\n' |
 awk '{p[$2]++; if ($3>=400) e[$2]++} END {for (k in p) printf "%s %d %d\n", k, p[k], e[k]+0}' | sort | column -t
```

**Answer:** /a  3  1
/b  1  1

Counting totals and errors per key with awk; column -t aligns. Verified by running it in Bash 5.

**Q142.** What does sort -u do?

**Answer:** Sorts and removes duplicates

Equivalent to sort | uniq.

**Q143.** When does join help?

**Answer:** Combining two sorted files on a common field

Both inputs must be sorted.

**Q144.** What does tr -d '\r' do?

**Answer:** Removes Windows carriage returns

Fix CRLF files.

### [07 · System Administration Scripting](level-4/07-sysadmin-scripting.md)

**Q145.** What should sysadmin scripts do before destructive actions?

**Answer:** Confirm, dry-run option and log what they will do

Support --dry-run.

**Q146.** Why run scripts as least privilege?

**Answer:** Mistakes have smaller blast radius

Use sudo only where needed.

**Q147.** What does df -h and du -sh show?

**Answer:** Filesystem and directory usage in human-readable form

Basic capacity checks.

**Q148.** Why log script actions to syslog or a file?

**Answer:** Auditing and troubleshooting

logger writes to syslog.

### [08 · Logging &amp; Observability for Scripts](level-4/08-logging-observability.md)

**Q149.** What does this logging helper print?

```bash
log() { local lvl=$1; shift; printf '%s [%s] %s\n' LEVEL "$lvl" "$*" >&2; }
log INFO starting 2>&1
log ERROR 'disk full' 2>&1 | sed 's/^LEVEL //'
exec 3>&1; echo data >&3
```

**Answer:** LEVEL [INFO] starting
[ERROR] disk full
data

Logs go to stderr so stdout stays clean for data. Verified by running it in Bash 5.

**Q150.** What is structured logging?

**Answer:** Key=value or JSON lines that tools can parse

Easier to search.

**Q151.** Why include timestamps and levels?

**Answer:** To order events and filter severity

Use ISO 8601 UTC.

**Q152.** What should never be logged?

**Answer:** Secrets and personal data

Scrub them.

### [09 · Security Hardening for Production Scripts](level-4/09-security-hardening.md)

**Q153.** What is a safe default for permissions on a script holding secrets?

**Answer:** chmod 600 (owner read/write only)

Least privilege.

**Q154.** What does quoting prevent in security terms?

**Answer:** Injection through spaces and special characters in values

Always quote expansions.

**Q155.** Why prefer absolute paths for commands in privileged scripts?

**Answer:** A manipulated PATH could run a malicious binary

Set PATH explicitly.

**Q156.** Why validate inputs with an allowlist?

**Answer:** Reject anything not explicitly expected

Better than blocklists.

### [10 · Capstone Project — Deployment Automation Tool](level-4/10-capstone-project.md)

**Q157.** What is the capstone's aim?

**Answer:** A production-quality shell tool with tests, help, logging and safe defaults

Bring it together.

**Q158.** What belongs in the README?

**Answer:** Install, usage, options, exit codes and examples

Users start there.

**Q159.** What proves the tool is reliable?

**Answer:** Tests, ShellCheck clean, idempotent runs and clear failure behaviour

Evidence.

**Q160.** How do you handle platform differences?

**Answer:** Detect OS and tools and document requirements

Test on target platforms.

## Mixed questions across all levels

**Q161.** What does this print?

```bash
set -u
args() { echo "$#"; for a in "$@"; do echo "<$a>"; done; }
v='a b'
args $v
args "$v"
args
```

**Answer:** 2
&lt;a&gt;
&lt;b&gt;
1
&lt;a b&gt;
0

Unquoted expansion word-splits into two arguments; quoted stays one. Verified by running it in Bash 5.

**Q162.** What does this print?

```bash
n=0
printf '1\n2\n3\n' | while read -r x; do n=$((n + x)); done
echo "n=$n"
n=0
while read -r x; do n=$((n + x)); done < <(printf '1\n2\n3\n')
echo "n=$n"
```

**Answer:** n=0
n=6

A pipeline runs the loop in a subshell, so the variable change is lost; process substitution keeps the loop in the current shell. Verified by running it in Bash 5.

**Q163.** What does this print?

```bash
f=archive.tar.gz
echo ${f%.gz} ${f%%.*} ${f##*.}
s='  pad  '; s=${s#"${s%%[![:space:]]*}"}; echo "[$s]"
x=${UNSET:-fallback}; echo $x ${#x}
```

**Answer:** archive.tar archive gz
[pad  ]
fallback 8

% and # trim suffixes and prefixes. Verified by running it in Bash 5.

**Q164.** What does this print?

```bash
declare -A m=([b]=2 [a]=1 [c]=3)
for k in $(printf '%s\n' "${!m[@]}" | sort); do printf '%s=%s ' $k ${m[$k]}; done; echo
echo ${#m[@]} ${m[z]:-none}
```

**Answer:** a=1 b=2 c=3 
3 none

Associative array order is unspecified, so sort the keys. Verified by running it in Bash 5.

**Q165.** What does this print?

```bash
cleanup() { echo bye; }
trap cleanup EXIT
false || echo recovered
( exit 5 ); echo "sub=$?"
```

**Answer:** recovered
sub=5
bye

The EXIT trap fires last, when the shell exits. Verified by running it in Bash 5.

**Q166.** A script runs rm -rf "$BUILD_DIR/"* and BUILD_DIR is accidentally empty. What protects you?

**Answer:** set -u or "${BUILD_DIR:?}" so an unset or empty variable aborts the script

An empty variable would expand to /*.

**Q167.** Why does set -e not make every failure stop a script?

**Answer:** It is ignored in conditions, || and &amp;&amp; lists, and some function contexts, so explicit checks are still needed

Handle errors deliberately.

**Q168.** Why is parsing ls output unsafe?

**Answer:** Filenames can contain spaces and newlines; use globs or find -print0 with read -d ''

Globs preserve names.

**Q169.** What does 2&gt;&amp;1 | tee log do?

**Answer:** Sends both stdout and stderr through the pipe so they are shown and logged

Order matters: redirect before piping.

**Q170.** Why trap EXIT for temp-file cleanup rather than cleaning at the end?

**Answer:** The trap runs on errors and interruptions too, not just on the happy path

Also handle INT and TERM.

**Q171.** What makes a cron job safe from overlapping itself?

**Answer:** A lock such as flock -n so a new run exits if the previous one is still running

Overlaps corrupt state.

**Q172.** How should a script treat untrusted input in commands?

**Answer:** Quote it, validate against an allowlist and pass -- before it; never eval it

Command injection is the risk.

**Q173.** Why should scripts avoid secrets in command-line arguments?

**Answer:** They show up in ps output and shell history

Prefer environment variables, files with strict permissions or stdin.

**Q174.** What is the macOS default bash version problem?

**Answer:** It ships Bash 3.2, which lacks associative arrays, mapfile and ${var^^}; install a newer bash or target POSIX

Check bash --version and document requirements.

**Q175.** What does the exit status of a pipeline normally reflect without pipefail?

**Answer:** Only the last command

set -o pipefail changes this.

**Q176.** When is it time to rewrite a shell script in another language?

**Answer:** When it needs complex data structures, heavy logic or serious unit testing

Shell is great glue, poor general-purpose language.

**Q177.** Which exit code convention signals 'command not found'?

**Answer:** 127

126 means found but not executable.

**Q178.** What does ShellCheck do for you?

**Answer:** Finds quoting, portability and logic mistakes statically

Run it in CI and locally.

**Q179.** Why does the first run of a deploy script need to be repeatable?

**Answer:** Re-running after a partial failure must reach the same end state (idempotency)

Check before creating; use -p and -f flags.

## Practical coding questions

### A tiny subcommand CLI

Build taskbook, a subcommand tool (add, done, list, stats) that keeps tasks in a file, uses declare -F for dispatch, reports errors on stderr with exit status 2 and cleans up with a trap. Run it with 2&gt;&amp;1 to see messages in order. It runs against a temporary file, so nothing on your system is touched.

Starter code:

```
#!/usr/bin/env bash
# taskbook: a tiny subcommand CLI storing tasks in a file (runs in a temp dir)
set -euo pipefail
DB=$(mktemp)
trap 'rm -f "$DB"' EXIT

die() { echo "error: $*" >&2; return 2; }

# TODO: cmd_add <title...>   append "<id>|open|<title>" (id = number of lines + 1), print "added #<id>"; with no title call die and return its status
# TODO: cmd_done <id>        mark open -> done with sed (write a new file, then mv); "completed #<id>", "#<id> already done" (status 0),
#                            or die "no task #<id>" for an unknown id; wrong argument count -> die "done needs one id"
# TODO: cmd_list [all|open|done]   print "<id> [x] title" for done tasks and "<id> [ ] title" for open ones
# TODO: cmd_stats            print "open=N done=M" with awk
# TODO: main                 dispatch "<cmd>" to cmd_<cmd> with declare -F, unknown -> die "unknown command: <cmd>"

main add Buy milk
main add Write report
main add Call Ana
main done 2
main done 2
main done 9 || echo "rc=$?"
main add || echo "rc=$?"
main list
main list open
main stats
main frobnicate || echo "rc=$?"
```

Expected output:

```text
added #1
added #2
added #3
completed #2
#2 already done
error: no task #9
rc=2
error: add needs a title
rc=2
1 [ ] Buy milk
2 [x] Write report
3 [ ] Call Ana
1 [ ] Buy milk
3 [ ] Call Ana
open=2 done=1
error: unknown command: frobnicate
rc=2
```

??? success "Show a solution"

    ```
    #!/usr/bin/env bash
    # taskbook: a tiny subcommand CLI storing tasks in a file (runs in a temp dir)
    set -euo pipefail
    DB=$(mktemp)
    trap 'rm -f "$DB"' EXIT

    die() { echo "error: $*" >&2; return 2; }

    cmd_add() {
      [ $# -ge 1 ] || { die "add needs a title"; return; }
      local id=$(( $(wc -l < "$DB") + 1 ))
      printf '%d|open|%s\n' "$id" "$*" >> "$DB"
      echo "added #$id"
    }

    cmd_done() {
      [ $# -eq 1 ] || { die "done needs one id"; return; }
      grep -q "^$1|" "$DB" || { die "no task #$1"; return; }
      grep -q "^$1|done|" "$DB" && { echo "#$1 already done"; return 0; }
      sed "s/^$1|open|/$1|done|/" "$DB" > "$DB.new" && mv "$DB.new" "$DB"
      echo "completed #$1"
    }

    cmd_list() {
      local filter=${1:-all}
      while IFS='|' read -r id state title; do
        [[ $filter == all || $filter == "$state" ]] && printf '%s [%s] %s\n' "$id" "$([[ $state == done ]] && echo x || echo ' ')" "$title"
      done < "$DB"
      return 0
    }

    cmd_stats() {
      awk -F'|' '{n[$2]++} END {printf "open=%d done=%d\n", n["open"], n["done"]}' "$DB"
    }

    main() {
      local c=${1:-help}; shift || true
      if declare -F "cmd_$c" > /dev/null; then "cmd_$c" "$@"; else die "unknown command: $c"; fi
    }

    main add Buy milk
    main add Write report
    main add Call Ana
    main done 2
    main done 2
    main done 9 || echo "rc=$?"
    main add || echo "rc=$?"
    main list
    main list open
    main stats
    main frobnicate || echo "rc=$?"
    ```

    Each subcommand is a function, the dispatcher checks for it with declare -F and unknown commands fail with a message and status. The sed-to-new-file-then-mv pattern avoids the non-portable sed -i. Verified by running it with Bash 5.3 on macOS (output captured with 2&gt;&amp;1).

### A validated sales report with associative arrays

Read a small CSV, skip and report invalid rows on stderr, total amounts per region with a Bash 4+ associative array (arithmetic in awk for decimals) and print regions ranked by total. Run it with Bash 4 or later (macOS ships 3.2, so install a newer bash) and with 2&gt;&amp;1.

Starter code:

```
#!/usr/bin/env bash
# sales report: validate rows, total per region with an associative array, rank regions
set -euo pipefail
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
cat > "$TMP/sales.csv" <<'CSV'
region,rep,amount
north,ana,1200.50
south,bo,800
north,cy,300.25
east,di,abc
west,ed,
south,fy,450.75
north,gu,-20
east,hy,950
CSV

# TODO: read the CSV (skip the header) and for each row
#   - if the amount is not a non-negative number, print "skip line <n>: invalid amount '<value>' for <rep>" to stderr and count it
#   - otherwise add the amount to total[<region>] (two decimals, use awk for the arithmetic) and count[<region>]
# TODO: print one line per region, highest total first:  printf '%-6s %9s (%d sales)\n'
# TODO: finish with "skipped: <n>"
# Run with 2>&1 to see the skip messages in order.
```

Expected output:

```text
skip line 5: invalid amount 'abc' for di
skip line 6: invalid amount '' for ed
skip line 8: invalid amount '-20' for gu
north    1500.75 (2 sales)
south    1250.75 (2 sales)
east      950.00 (1 sales)
skipped: 3
```

??? success "Show a solution"

    ```
    #!/usr/bin/env bash
    # sales report: validate rows, total per region with an associative array, rank regions
    set -euo pipefail
    TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
    cat > "$TMP/sales.csv" <<'CSV'
    region,rep,amount
    north,ana,1200.50
    south,bo,800
    north,cy,300.25
    east,di,abc
    west,ed,
    south,fy,450.75
    north,gu,-20
    east,hy,950
    CSV

    declare -A total count
    bad=0 line=0
    while IFS=, read -r region rep amount; do
      line=$((line + 1))
      [ "$line" -eq 1 ] && continue
      if [[ ! $amount =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
        echo "skip line $line: invalid amount '${amount}' for $rep" >&2
        bad=$((bad + 1)); continue
      fi
      total[$region]=$(awk -v a="${total[$region]:-0}" -v b="$amount" 'BEGIN {printf "%.2f", a + b}')
      count[$region]=$(( ${count[$region]:-0} + 1 ))
    done < "$TMP/sales.csv"

    for r in "${!total[@]}"; do echo "$r ${total[$r]} ${count[$r]}"; done | sort -k2,2nr | while read -r r t c; do
      printf '%-6s %9s (%d sales)\n' "$r" "$t" "$c"
    done
    echo "skipped: $bad"
    ```

    Validation with a regex stops bad data early, awk does the decimal arithmetic bash cannot, and sorting a generated listing gives a stable ranking because associative array order is unspecified. Verified by running it with Bash 5.3 on macOS (output captured with 2&gt;&amp;1).

---

More free practice: the [Shell Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
