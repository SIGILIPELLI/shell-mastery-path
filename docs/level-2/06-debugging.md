# 06 · Debugging Scripts

Shell scripts fail in ways that can be hard to see just from reading the
source — a variable that's empty when you expected a value, a pipeline that
"succeeds" despite an inner failure, a quoting bug that only shows up on
certain input. This module covers bash's built-in debugging tools.

## `bash -n` — syntax check without running anything

```bash
bash -n myscript.sh
# prints nothing if the syntax is valid

bash -n broken.sh
# broken.sh: line 14: syntax error near unexpected token `fi'
```

`-n` ("no-exec") parses the script and reports syntax errors without
actually running any of it — a fast first check before you run something
you're not sure about, and a good pre-commit hook check (Level 3).

## `set -x` — trace every command as it executes

```bash
#!/usr/bin/env bash
set -x     # turn tracing ON

name="Ada"
greeting="Hello, $name"
echo "$greeting"

set +x     # turn tracing OFF
```

```text
+ name=Ada
+ greeting='Hello, Ada'
+ echo 'Hello, Ada'
Hello, Ada
```

Each traced line is prefixed with `+` and shows the command **after**
variable expansion — invaluable for seeing exactly what a variable actually
contained at the moment a command ran.

```bash
# trace only part of a script
some_setup_steps
set -x
suspect_function_call
set +x
more_steps
```

```bash
# trace an entire script from the command line without editing it
bash -x myscript.sh
```

## Customizing the trace prefix with `PS4`

```bash
export PS4='+ ${BASH_SOURCE}:${LINENO}: '
set -x

echo "hi"
# + myscript.sh:5: echo hi
```

Adding the filename and line number to `PS4` makes `set -x` output far more
useful in scripts that source other files or call many functions.

## `trap ... ERR` — running code when any command fails

```bash
#!/usr/bin/env bash
set -e

on_error() {
    echo "ERROR on line $1, exit code $2" >&2
}
trap 'on_error $LINENO $?' ERR

echo "step 1"
false           # this triggers the ERR trap, then set -e exits the script
echo "never reached"
```

```text
step 1
ERROR on line 10, exit code 1
```

`trap ... ERR` fires whenever a command exits non-zero (subject to the same
rules as `set -e`) — combined with `$LINENO`, it pinpoints exactly where a
script died, which is far more useful than a bare "command failed"
somewhere in a long script.

## Debugging with `declare -p` and `${var@Q}`

```bash
name="  hello  "
declare -p name
# declare -- name="  hello  "

echo "${name@Q}"
# '  hello  '
```

`declare -p` prints a variable's exact declaration, including type
(array, associative array, etc.) — great for confirming a variable actually
holds what you think it does, especially around whitespace or quoting bugs.
`${var@Q}` (bash 4.4+) shows a value already quoted for re-use as shell
input.

## Common pitfalls checklist

```bash
# 1. Unquoted variables that break on spaces or empty values
for f in $files; do ...     # BAD if $files can contain spaces
for f in "${files[@]}"; do ...   # GOOD — use an array + quoting

# 2. Using = instead of == inside [[ ]] (both work, but == is clearer intent)
[[ "$a" == "$b" ]]

# 3. Comparing strings with -eq/-ne (those are for NUMBERS)
[[ "$a" -eq "$b" ]]     # BAD if $a/$b are strings — use == / !=

# 4. Forgetting that `local var=$(cmd)` masks the command's exit code
local result
result=$(cmd)            # GOOD — separate declaration and assignment
                            # (with `local result=$(cmd)`, `local`'s own exit
                            #  status is what set -e sees, not cmd's)

# 5. A pipeline "succeeding" even though an early stage failed
grep "x" nonexistent.txt | wc -l    # wc always exits 0 — need `set -o pipefail`
```

## Debugging step-by-step interactively

```bash
# insert a breakpoint-style pause
read -p "Press enter to continue after checking state..." _
echo "resuming, var was: $suspect_var"
```

```bash
# print-debugging with clear markers so it's easy to grep back out later
echo "DEBUG: count=$count, status=$status" >&2
```

## Cheat sheet

| Tool | Purpose |
|------|---------|
| `bash -n script.sh` | check syntax only, don't execute |
| `set -x` / `set +x` | trace commands as they run / stop tracing |
| `bash -x script.sh` | trace an entire script without editing it |
| `PS4` | customize the `set -x` trace prefix (add `$LINENO`, etc.) |
| `trap 'handler' ERR` | run a handler whenever a command fails |
| `declare -p var` | print a variable's exact type and value |
| `$LINENO` | current line number |

## Exercise

Take a script with a deliberately introduced bug — for example, one that
loops over `$files` (unquoted, unset) instead of a proper array — and use
`bash -x` plus a `trap ... ERR` handler with `$LINENO` to locate and
diagnose the failure without reading the source top to bottom. Write up the
one-line fix.
