# 05 · Functions & Arguments

## Defining and calling a function

```bash
#!/usr/bin/env bash

greet() {
    echo "Hello, $1!"
}

greet "Ada"        # Hello, Ada!
greet "Grace"      # Hello, Grace!
```

Both `greet() { ... }` and `function greet { ... }` work in Bash; the first
form is more portable (POSIX `sh` doesn't understand the `function` keyword),
so prefer it.

## Positional parameters inside a function

Functions get their own set of positional parameters (`$1`, `$2`, ...) — they
don't inherit the script's, they receive whatever was passed at the call site.

```bash
show_args() {
    echo "first: $1"
    echo "second: $2"
    echo "all args: $@"
    echo "count: $#"
}

show_args one two three
# first: one
# second: two
# all args: one two three
# count: 3
```

## `$@` vs `$*` — always quote `"$@"`

```bash
print_each() {
    for arg in "$@"; do
        echo "arg: [$arg]"
    done
}

print_each "hello world" foo
# arg: [hello world]     <- preserved as ONE argument
# arg: [foo]
```

```bash
print_each_star() {
    for arg in "$*"; do
        echo "arg: [$arg]"
    done
}

print_each_star "hello world" foo
# arg: [hello world foo]   <- collapsed into a SINGLE string
```

`"$@"` expands to each positional parameter as its own separately-quoted
word — this is what you want almost every time you forward arguments. `"$*"`
joins everything into one string. Unquoted `$@` and `$*` both word-split and
lose this distinction, so always quote.

## Return values: `return` vs `echo`

`return` only sets a numeric **exit status** (0–255) — it is not a way to
hand back arbitrary data like a string or number result.

```bash
is_even() {
    local n=$1
    if (( n % 2 == 0 )); then
        return 0     # success/true
    else
        return 1     # failure/false
    fi
}

if is_even 4; then
    echo "4 is even"
fi
```

To return an actual **value** (like a computed string or number), print it
with `echo`/`printf` and capture it with command substitution:

```bash
square() {
    local n=$1
    echo $(( n * n ))
}

result=$(square 5)
echo "5 squared is $result"    # 5 squared is 25
```

## `local` — keep function variables from leaking

```bash
counter=0

bump() {
    local counter=100     # shadows the outer variable, only inside this function
    ((counter++))
    echo "inside: $counter"
}

bump           # inside: 101
echo "outside: $counter"   # outside: 0 — untouched
```

Without `local`, a variable assigned inside a function is **global** by
default — a common source of bugs in longer scripts. Always declare
function-local state with `local`.

## Default values for missing arguments

```bash
backup() {
    local source=$1
    local dest=${2:-/tmp/backups}    # default if $2 wasn't passed
    echo "backing up $source to $dest"
}

backup "/home/user/docs"                 # uses the default dest
backup "/home/user/docs" "/mnt/backup"   # uses the explicit dest
```

## Checking argument count

```bash
require_args() {
    if [[ $# -lt 2 ]]; then
        echo "Usage: require_args <name> <age>" >&2
        return 1
    fi
    echo "$1 is $2 years old"
}

require_args "Ada"          # prints usage to stderr, returns 1
require_args "Ada" 30       # Ada is 30 years old
```

## Cheat sheet

| Symbol | Meaning inside a function |
|--------|-----------------------------|
| `$1`, `$2`, ... | positional arguments passed to the function |
| `$#` | number of arguments passed |
| `"$@"` | all arguments, each preserved as a separate word |
| `"$*"` | all arguments joined into one string |
| `local var` | scope a variable to this function only |
| `return N` | set the function's exit status (0-255, not a value) |
| `echo value` + `$(func)` | the idiom for "returning" an actual value |

## Exercise

Write a function `max_of` that takes two numbers as arguments and echoes the
larger one (capture it with `$( )` when calling). Then write a function
`validate_args` that checks exactly 2 arguments were passed to the script,
printing a usage message to stderr and returning 1 if not.
