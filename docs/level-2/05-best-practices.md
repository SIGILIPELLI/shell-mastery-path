# 05 · Scripting Best Practices

Writing a script that *works* is one thing; writing one that's safe, easy to
read six months later, and doesn't silently misbehave is another. This
module collects the conventions professional shell scripts follow.

## Always start with strict mode

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| Setting | Effect |
|---------|--------|
| `set -e` | exit on any unhandled non-zero exit code |
| `set -u` | error on unset variable references |
| `set -o pipefail` | a pipeline fails if any stage fails, not just the last |
| `IFS=$'\n\t'` | word-split only on newline/tab, not every space — safer for filenames with spaces |

This "unofficial strict mode" (from Level 1's error-handling module) should
be the first line of nearly every non-trivial script.

## Quote everything

```bash
# BAD — breaks on filenames with spaces, globs, or empty values
if [ $file = "" ]; then

# GOOD
if [[ -z "$file" ]]; then
```

```bash
# BAD — word-splits and glob-expands
cp $source $dest

# GOOD
cp "$source" "$dest"
```

The rule of thumb: quote every variable expansion unless you have a
specific, understood reason not to (like intentional word-splitting, which
is rare).

## Prefer `[[ ]]` over `[ ]`

```bash
[[ -f "$file" && -r "$file" ]]     # bash-native, supports &&/||, no word-splitting issues
[ -f "$file" ] && [ -r "$file" ]     # POSIX sh, needs separate [ ] per test, more error-prone
```

`[[ ]]` is a bash (and zsh/ksh) keyword, not a command — it doesn't
word-split or glob-expand its operands, which eliminates a whole class of
quoting bugs. Use plain `[ ]` only when a script must run under a strict
POSIX `sh` (see Level 4's "Cross-Shell Compatibility").

## Use `shellcheck` — a static analyzer for shell scripts

```bash
# install (macOS)
brew install shellcheck

# run against a script
shellcheck backup.sh
```

```text
In backup.sh line 12:
cp $source $dest
   ^-- SC2086: Double quote to prevent globbing and word splitting.
   ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

Did you mean:
cp "$source" "$dest"
```

`shellcheck` catches quoting bugs, unreachable code, common typos, and
dozens of other subtle mistakes before they cause a 2am incident — it's
worth running on every script, and wiring into CI (Level 3's "Git Hooks &
CI" module) so it runs automatically.

## Naming and structure conventions

```bash
# constants/config: UPPER_SNAKE_CASE, near the top of the file
readonly MAX_RETRIES=3
readonly LOG_FILE="/var/log/myscript.log"

# regular variables and function names: lower_snake_case
retry_count=0

process_file() {
    local file="$1"
    # ...
}

# always use `local` for function-scoped variables (Level 1, "Functions")
```

```bash
#!/usr/bin/env bash
#
# backup.sh — archive a directory with retention cleanup
#
# Usage: backup.sh <source_dir>
#
set -euo pipefail

# ---- constants ----
# ---- functions ----
# ---- main script logic ----
```

A short header comment (what the script does, how to call it) and clearly
separated sections make a script far easier to pick back up later.

## Prefer `$( )` over backticks

```bash
result=$(command)             # nestable, more readable
nested=$(echo $(date +%F))      # works fine

old=`command`                  # legacy syntax
nested_old=`echo \`date +%F\``   # ugly and error-prone when nested
```

## Fail fast and validate input early

```bash
process() {
    local input_file="$1"

    [[ -n "$input_file" ]] || { echo "Usage: process <file>" >&2; return 1; }
    [[ -f "$input_file" ]] || { echo "Error: '$input_file' not found" >&2; return 1; }
    [[ -r "$input_file" ]] || { echo "Error: '$input_file' not readable" >&2; return 1; }

    # only reaches here once every precondition holds
    echo "processing $input_file..."
}
```

Checking every assumption at the top of a function means the rest of the
logic can trust its inputs completely — no defensive checks scattered
everywhere else.

## Cheat sheet

| Practice | Why |
|----------|-----|
| `set -euo pipefail` | fail loudly instead of silently continuing |
| Quote all variable expansions | avoid word-splitting/glob bugs |
| Use `[[ ]]` not `[ ]` | safer, more expressive conditionals |
| Run `shellcheck` | catches bugs before they run in production |
| `UPPER_SNAKE_CASE` for constants | visually distinguishes config from local state |
| `local` inside every function | avoids leaking variables into global scope |
| `$( )` not backticks | nestable, readable command substitution |
| Validate inputs first, fail fast | keeps the rest of the logic simple |

## Exercise

Take your `backup.sh` from Level 1's capstone project, run `shellcheck` on
it, and fix every warning it reports. Then refactor it to use
`readonly` for its configuration constants and add a header comment block
describing usage — commit the cleaned-up version as `backup_v2.sh`.
