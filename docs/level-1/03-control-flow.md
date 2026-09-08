# 03 · Control Flow

## if / elif / else

```bash
#!/usr/bin/env bash

age=20

if [[ $age -lt 13 ]]; then
    echo "child"
elif [[ $age -lt 20 ]]; then
    echo "teenager"
else
    echo "adult"
fi
```

`if` runs the block after `then` when the command right after `if` exits with
status 0 (success). `[[ ... ]]` is itself just a command that evaluates a
condition and exits 0 (true) or 1 (false).

## `test`, `[ ]`, and `[[ ]]`

```bash
test -f "myfile.txt"          # the `test` builtin
[ -f "myfile.txt" ]            # `[` is an alias for `test`, needs closing `]`
[[ -f "myfile.txt" ]]          # bash's extended, safer test — prefer this
```

| Feature | `[ ]` (POSIX `test`) | `[[ ]]` (bash extension) |
|---------|----------------------|---------------------------|
| Word splitting on unquoted vars | happens — needs careful quoting | doesn't happen |
| `&&` / `\|\|` inside | not safely | yes |
| Pattern matching (`==` with `*`) | no | yes |
| Regex (`=~`) | no | yes |
| Portability to `/bin/sh` | yes | no (bash/ksh/zsh only) |

Use `[[ ]]` in bash scripts; fall back to `[ ]` only if you need a script to
run under plain POSIX `sh`.

## Common test operators

```bash
# String comparisons
[[ "$a" == "$b" ]]      # equal
[[ "$a" != "$b" ]]      # not equal
[[ -z "$a" ]]            # true if $a is empty ("zero length")
[[ -n "$a" ]]            # true if $a is NOT empty

# Numeric comparisons (use -eq/-lt etc, NOT == , inside [ ] or [[ ]])
[[ $x -eq $y ]]          # equal
[[ $x -ne $y ]]          # not equal
[[ $x -lt $y ]]          # less than
[[ $x -le $y ]]          # less than or equal
[[ $x -gt $y ]]          # greater than
[[ $x -ge $y ]]          # greater than or equal

# File tests
[[ -e path ]]            # exists
[[ -f path ]]            # exists AND is a regular file
[[ -d path ]]            # exists AND is a directory
[[ -r path ]]            # readable
[[ -w path ]]            # writable
[[ -x path ]]            # executable
[[ -s path ]]            # exists and is non-empty
```

## Combining conditions

```bash
age=25
has_id=true

if [[ $age -ge 18 && $has_id == true ]]; then
    echo "can enter"
fi

if [[ -f "config.yml" || -f "config.yaml" ]]; then
    echo "found a config file"
fi
```

Inside `[[ ]]`, `&&` and `||` work directly. Inside old-style `[ ]`, use `-a`
/ `-o`, or better, chain separate `[ ]` commands with `&&`/`||`.

## Arithmetic conditions with `(( ))`

For pure numeric comparisons, `(( ))` reads more like a normal programming
language:

```bash
count=5
if (( count > 3 )); then
    echo "more than three"
fi

if (( count % 2 == 0 )); then
    echo "even"
else
    echo "odd"
fi
```

## case statements

`case` is Bash's pattern-matching switch — cleaner than a long `if/elif`
chain when checking one value against several patterns:

```bash
#!/usr/bin/env bash

read -rp "Enter a fruit: " fruit

case "$fruit" in
    apple|pear)
        echo "pome fruit"
        ;;
    banana)
        echo "tropical fruit"
        ;;
    orange|lemon|lime)
        echo "citrus fruit"
        ;;
    *)
        echo "unknown fruit"
        ;;
esac
```

`read -rp "prompt: " var` prints the prompt and reads into `var` in one step.
Each `case` branch ends with `;;`. `*)` is the catch-all default, matching
anything not caught above it. Patterns support globs (`*`, `?`, `[abc]`), and
`|` separates alternatives within one branch.

## Exit status of conditionals directly

Since every command has an exit status, you often don't need `if` at all:

```bash
grep -q "error" logfile.txt && echo "found an error"
mkdir -p /tmp/mydir || echo "could not create directory"
```

`&&` runs the right side only if the left side succeeded; `||` runs the right
side only if the left side failed. This is covered more in Module 7.

## How It Actually Works

`if`, `[[ ]]`, and `[ ]` all ultimately reduce to one thing bash cares about:
an **exit status**. `if command; then` doesn't evaluate a boolean — it runs
`command` as a real child process (or builtin), waits for it, and checks
whether the integer exit status stored in `$?` is exactly `0` (success) or
nonzero (failure). `[[ $age -lt 13 ]]` looks like syntax but `[[` is a shell
keyword handled entirely inside bash's parser (no fork needed), while the
older `[ $age -lt 13 ]` is literally a command named `[` (an alias for
`test`) that gets forked and exec'd like any external program on systems
where it isn't a builtin — that's also why `[ ]` needs the closing `]` as a
literal final argument.

`elif` is just sugar for a nested `else if`; the parser builds a single
compound command internally. `&&` and `||` are short-circuit control
operators evaluated left to right at parse time into a pipeline list — bash
only executes the right-hand command if the left-hand one's exit status
makes it necessary, so `mkdir dir && cd dir` skips the `cd` entirely if
`mkdir` returns nonzero, without any explicit `if`.


## Exercise

Write `check_age.sh` that reads an age from user input with `read -rp`, then
uses `if/elif/else` to print `"child"` (under 13), `"teenager"` (13–19), or
`"adult"` (20+). Then write a second version, `grade.sh`, that reads a single
letter grade (A/B/C/D/F) and uses a `case` statement to print a short comment
for each.
