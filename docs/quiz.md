---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Shell Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```bash\nset -u\nargs() { echo \"$#\"; for a in \"$@\"; do echo \"<$a>\"; done; }\nv='a b'\nargs $v\nargs \"$v\"\nargs\n```\n",
    "options": [
      "2\n<a>\n<b>\n1\n<a b>\n0",
      "1\n<a b>\n1\n<a b>\n0",
      "2\n<a>\n<b>\n1\n<a b>\n1",
      "2\n<a>\n<b>\n2\n<a>\n<b>\n0"
    ],
    "correctIndex": 0,
    "explanation": "Unquoted expansion word-splits into two arguments; quoted stays one. Verified by running it in Bash 5."
  },
  {
    "question": "What does this print?\n```bash\nn=0\nprintf '1\\n2\\n3\\n' | while read -r x; do n=$((n + x)); done\necho \"n=$n\"\nn=0\nwhile read -r x; do n=$((n + x)); done < <(printf '1\\n2\\n3\\n')\necho \"n=$n\"\n```\n",
    "options": [
      "n=0\nn=6",
      "n=0\nn=0",
      "n=6\nn=0",
      "n=6\nn=6"
    ],
    "correctIndex": 0,
    "explanation": "A pipeline runs the loop in a subshell, so the variable change is lost; process substitution keeps the loop in the current shell. Verified by running it in Bash 5."
  },
  {
    "question": "What does this print?\n```bash\nf=archive.tar.gz\necho ${f%.gz} ${f%%.*} ${f##*.}\ns='  pad  '; s=${s#\"${s%%[![:space:]]*}\"}; echo \"[$s]\"\nx=${UNSET:-fallback}; echo $x ${#x}\n```\n",
    "options": [
      "archive.tar archive tar.gz\n[pad  ]\nfallback 8",
      "archive.tar.gz archive gz\n[pad  ]\nfallback 8",
      "archive.tar archive gz\n[pad  ]\nfallback 8",
      "archive.tar archive gz\n[pad]\nfallback 9"
    ],
    "correctIndex": 2,
    "explanation": "% and # trim suffixes and prefixes. Verified by running it in Bash 5."
  },
  {
    "question": "What does this print?\n```bash\ndeclare -A m=([b]=2 [a]=1 [c]=3)\nfor k in $(printf '%s\\n' \"${!m[@]}\" | sort); do printf '%s=%s ' $k ${m[$k]}; done; echo\necho ${#m[@]} ${m[z]:-none}\n```\n",
    "options": [
      "a=1 b=2 c=3 \n3 0",
      "a=1 b=2 c=3 \n2 none",
      "b=2 a=1 c=3 \n3 none",
      "a=1 b=2 c=3 \n3 none"
    ],
    "correctIndex": 3,
    "explanation": "Associative array order is unspecified, so sort the keys. Verified by running it in Bash 5."
  },
  {
    "question": "What does this print?\n```bash\ncleanup() { echo bye; }\ntrap cleanup EXIT\nfalse || echo recovered\n( exit 5 ); echo \"sub=$?\"\n```\n",
    "options": [
      "recovered\nbye\nsub=5",
      "recovered\nsub=5\nbye",
      "bye\nrecovered\nsub=5",
      "recovered\nsub=0\nbye"
    ],
    "correctIndex": 1,
    "explanation": "The EXIT trap fires last, when the shell exits. Verified by running it in Bash 5."
  },
  {
    "question": "A script runs rm -rf \"$BUILD_DIR/\"* and BUILD_DIR is accidentally empty. What protects you?",
    "options": [
      "Quotes alone",
      "set -u or \"${BUILD_DIR:?}\" so an unset or empty variable aborts the script",
      "Nothing",
      "Using sudo"
    ],
    "correctIndex": 1,
    "explanation": "An empty variable would expand to /*."
  },
  {
    "question": "Why does set -e not make every failure stop a script?",
    "options": [
      "It needs root",
      "It only works in zsh",
      "It is buggy",
      "It is ignored in conditions, || and && lists, and some function contexts, so explicit checks are still needed"
    ],
    "correctIndex": 3,
    "explanation": "Handle errors deliberately."
  },
  {
    "question": "Why is parsing ls output unsafe?",
    "options": [
      "ls needs root",
      "ls is slow",
      "ls is removed",
      "Filenames can contain spaces and newlines; use globs or find -print0 with read -d ''"
    ],
    "correctIndex": 3,
    "explanation": "Globs preserve names."
  },
  {
    "question": "What does 2>&1 | tee log do?",
    "options": [
      "Logs only stdout",
      "Hides errors",
      "Swaps streams",
      "Sends both stdout and stderr through the pipe so they are shown and logged"
    ],
    "correctIndex": 3,
    "explanation": "Order matters: redirect before piping."
  },
  {
    "question": "Why trap EXIT for temp-file cleanup rather than cleaning at the end?",
    "options": [
      "The trap runs on errors and interruptions too, not just on the happy path",
      "It is required",
      "It is shorter",
      "It is faster"
    ],
    "correctIndex": 0,
    "explanation": "Also handle INT and TERM."
  },
  {
    "question": "What makes a cron job safe from overlapping itself?",
    "options": [
      "Run it more often",
      "A lock such as flock -n so a new run exits if the previous one is still running",
      "Use sudo",
      "Use sleep"
    ],
    "correctIndex": 1,
    "explanation": "Overlaps corrupt state."
  },
  {
    "question": "How should a script treat untrusted input in commands?",
    "options": [
      "Eval it",
      "Strip spaces only",
      "Quote it, validate against an allowlist and pass -- before it; never eval it",
      "Trust it"
    ],
    "correctIndex": 2,
    "explanation": "Command injection is the risk."
  },
  {
    "question": "Why should scripts avoid secrets in command-line arguments?",
    "options": [
      "Colour",
      "Length limits only",
      "They show up in ps output and shell history",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "Prefer environment variables, files with strict permissions or stdin."
  },
  {
    "question": "What is the macOS default bash version problem?",
    "options": [
      "It ships Bash 3.2, which lacks associative arrays, mapfile and ${var^^}; install a newer bash or target POSIX",
      "It is too new",
      "It has no pipes",
      "It has no bash"
    ],
    "correctIndex": 0,
    "explanation": "Check bash --version and document requirements."
  },
  {
    "question": "What does the exit status of a pipeline normally reflect without pipefail?",
    "options": [
      "The average",
      "Any failure",
      "The first command",
      "Only the last command"
    ],
    "correctIndex": 3,
    "explanation": "set -o pipefail changes this."
  },
  {
    "question": "When is it time to rewrite a shell script in another language?",
    "options": [
      "When it needs complex data structures, heavy logic or serious unit testing",
      "Never",
      "After 10 lines",
      "Always"
    ],
    "correctIndex": 0,
    "explanation": "Shell is great glue, poor general-purpose language."
  },
  {
    "question": "Which exit code convention signals 'command not found'?",
    "options": [
      "0",
      "127",
      "1",
      "255"
    ],
    "correctIndex": 1,
    "explanation": "126 means found but not executable."
  },
  {
    "question": "What does ShellCheck do for you?",
    "options": [
      "Installs packages",
      "Formats code",
      "Runs tests",
      "Finds quoting, portability and logic mistakes statically"
    ],
    "correctIndex": 3,
    "explanation": "Run it in CI and locally."
  },
  {
    "question": "Why does the first run of a deploy script need to be repeatable?",
    "options": [
      "Colour",
      "Speed",
      "Re-running after a partial failure must reach the same end state (idempotency)",
      "Law"
    ],
    "correctIndex": 2,
    "explanation": "Check before creating; use -p and -f flags."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A tiny subcommand CLI

Build taskbook, a subcommand tool (add, done, list, stats) that keeps tasks in a file, uses declare -F for dispatch, reports errors on stderr with exit status 2 and cleans up with a trap. Run it with 2>&1 to see messages in order. It runs against a temporary file, so nothing on your system is touched.

**Starter code**

```bash
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

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>#!/usr/bin/env bash
# taskbook: a tiny subcommand CLI storing tasks in a file (runs in a temp dir)
set -euo pipefail
DB=$(mktemp)
trap &#x27;rm -f &quot;$DB&quot;&#x27; EXIT

die() { echo &quot;error: $*&quot; &gt;&amp;2; return 2; }

cmd_add() {
  [ $# -ge 1 ] || { die &quot;add needs a title&quot;; return; }
  local id=$(( $(wc -l &lt; &quot;$DB&quot;) + 1 ))
  printf &#x27;%d|open|%s\n&#x27; &quot;$id&quot; &quot;$*&quot; &gt;&gt; &quot;$DB&quot;
  echo &quot;added #$id&quot;
}

cmd_done() {
  [ $# -eq 1 ] || { die &quot;done needs one id&quot;; return; }
  grep -q &quot;^$1|&quot; &quot;$DB&quot; || { die &quot;no task #$1&quot;; return; }
  grep -q &quot;^$1|done|&quot; &quot;$DB&quot; &amp;&amp; { echo &quot;#$1 already done&quot;; return 0; }
  sed &quot;s/^$1|open|/$1|done|/&quot; &quot;$DB&quot; &gt; &quot;$DB.new&quot; &amp;&amp; mv &quot;$DB.new&quot; &quot;$DB&quot;
  echo &quot;completed #$1&quot;
}

cmd_list() {
  local filter=${1:-all}
  while IFS=&#x27;|&#x27; read -r id state title; do
    [[ $filter == all || $filter == &quot;$state&quot; ]] &amp;&amp; printf &#x27;%s [%s] %s\n&#x27; &quot;$id&quot; &quot;$([[ $state == done ]] &amp;&amp; echo x || echo &#x27; &#x27;)&quot; &quot;$title&quot;
  done &lt; &quot;$DB&quot;
  return 0
}

cmd_stats() {
  awk -F&#x27;|&#x27; &#x27;{n[$2]++} END {printf &quot;open=%d done=%d\n&quot;, n[&quot;open&quot;], n[&quot;done&quot;]}&#x27; &quot;$DB&quot;
}

main() {
  local c=${1:-help}; shift || true
  if declare -F &quot;cmd_$c&quot; &gt; /dev/null; then &quot;cmd_$c&quot; &quot;$@&quot;; else die &quot;unknown command: $c&quot;; fi
}

main add Buy milk
main add Write report
main add Call Ana
main done 2
main done 2
main done 9 || echo &quot;rc=$?&quot;
main add || echo &quot;rc=$?&quot;
main list
main list open
main stats
main frobnicate || echo &quot;rc=$?&quot;
</code></pre>

<p>Each subcommand is a function, the dispatcher checks for it with declare -F and unknown commands fail with a message and status. The sed-to-new-file-then-mv pattern avoids the non-portable sed -i. Verified by running it with Bash 5.3 on macOS (output captured with 2&gt;&amp;1).</p>
</details>

### A validated sales report with associative arrays

Read a small CSV, skip and report invalid rows on stderr, total amounts per region with a Bash 4+ associative array (arithmetic in awk for decimals) and print regions ranked by total. Run it with Bash 4 or later (macOS ships 3.2, so install a newer bash) and with 2>&1.

**Starter code**

```bash
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

**Expected output**

```text
skip line 5: invalid amount 'abc' for di
skip line 6: invalid amount '' for ed
skip line 8: invalid amount '-20' for gu
north    1500.75 (2 sales)
south    1250.75 (2 sales)
east      950.00 (1 sales)
skipped: 3
```

<details><summary>Show solution and explanation</summary>

<pre><code>#!/usr/bin/env bash
# sales report: validate rows, total per region with an associative array, rank regions
set -euo pipefail
TMP=$(mktemp -d); trap &#x27;rm -rf &quot;$TMP&quot;&#x27; EXIT
cat &gt; &quot;$TMP/sales.csv&quot; &lt;&lt;&#x27;CSV&#x27;
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
  [ &quot;$line&quot; -eq 1 ] &amp;&amp; continue
  if [[ ! $amount =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
    echo &quot;skip line $line: invalid amount &#x27;${amount}&#x27; for $rep&quot; &gt;&amp;2
    bad=$((bad + 1)); continue
  fi
  total[$region]=$(awk -v a=&quot;${total[$region]:-0}&quot; -v b=&quot;$amount&quot; &#x27;BEGIN {printf &quot;%.2f&quot;, a + b}&#x27;)
  count[$region]=$(( ${count[$region]:-0} + 1 ))
done &lt; &quot;$TMP/sales.csv&quot;

for r in &quot;${!total[@]}&quot;; do echo &quot;$r ${total[$r]} ${count[$r]}&quot;; done | sort -k2,2nr | while read -r r t c; do
  printf &#x27;%-6s %9s (%d sales)\n&#x27; &quot;$r&quot; &quot;$t&quot; &quot;$c&quot;
done
echo &quot;skipped: $bad&quot;
</code></pre>

<p>Validation with a regex stops bad data early, awk does the decimal arithmetic bash cannot, and sorting a generated listing gives a stable ranking because associative array order is unspecified. Verified by running it with Bash 5.3 on macOS (output captured with 2&gt;&amp;1).</p>
</details>
