# 02 · Variables & Data Types

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/7vPInOnZ9lM" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


Bash doesn't have real "data types" the way Python or Java do — everything is
stored as a string by default, and arithmetic contexts reinterpret strings as
integers. Understanding this is the key to not being surprised later.

## Declaring and using variables

```bash
name="Ada"          # NO spaces around = — this is critical
age=30

echo "$name is $age years old"
# Ada is 30 years old
```

```bash
name = "Ada"         # WRONG: bash parses this as running a command `name`
                      # with arguments `=` and `Ada"` — syntax error
```

Variable names are case-sensitive, can contain letters, digits, and
underscores, and can't start with a digit.

## Reading a variable: always quote it

```bash
food="fried rice"
echo $food            # risky: word-splits on spaces
echo "$food"           # safe: prints "fried rice" as one value
```

Without quotes, Bash performs **word splitting** on the variable's value
before passing it along — a value with spaces can silently turn into multiple
arguments. Quoting `"$var"` is the single most important habit in this whole
curriculum.

## Numbers and arithmetic

There's no native "integer" type — arithmetic happens inside `$(( ... ))` or
`(( ... ))`, which treat the content as numeric expressions:

```bash
a=5
b=3

echo $((a + b))       # 8
echo $((a * b))       # 15
echo $((a / b))       # 1  — integer division, no decimals!
echo $((a % b))       # 2  — remainder

((a++))               # increment a by 1 (a is now 6)
echo "$a"              # 6
```

For decimals, Bash has no built-in support — reach for `bc` or `awk`:

```bash
echo "10 / 3" | bc -l   # 3.33333333333333333333
awk 'BEGIN { print 10 / 3 }'   # 3.33333
```

## Quoting rules — the part everyone trips on

| Quote style | Variable expansion? | Word splitting? | Use for |
|-------------|---------------------|------------------|---------|
| `"double quotes"` | Yes | No | almost everything |
| `'single quotes'` | No (literal) | No | fixed text, no `$var` inside |
| no quotes | Yes | Yes (dangerous) | rarely — only literals with no spaces |

```bash
name="World"
echo "Hello, $name"     # Hello, World       (expands)
echo 'Hello, $name'     # Hello, $name        (literal, no expansion)
echo Hello, $name       # works here, but fragile — avoid the habit
```

## Read-only and unsetting variables

```bash
readonly PI=3.14159
PI=3            # error: PI: readonly variable

count=5
unset count
echo "${count:-not set}"    # not set  (default-value expansion, see below)
```

## Environment variables vs. shell variables

```bash
local_var="only visible in this shell"
export SHARED_VAR="visible to child processes too"

bash -c 'echo "$SHARED_VAR"'   # prints the value — export propagates it
bash -c 'echo "$local_var"'    # prints nothing — not exported
```

`export` marks a variable so it's copied into the environment of any child
process the shell starts (like scripts it runs, or other programs).

## Useful default/built-in variables

```bash
echo "$0"     # the script's own name/path
echo "$1"     # first positional argument (Module 5 covers these in depth)
echo "$#"     # number of positional arguments passed to the script
echo "$$"     # PID of the current shell
echo "$?"     # exit code of the last command (Module 9 covers this in depth)
```

## Parameter expansion basics (defaults)

```bash
greeting="${name:-friend}"     # if $name is unset or empty, use "friend"
echo "$greeting"

: "${PORT:=8080}"               # if PORT is unset, SET it to 8080
echo "$PORT"
```

## How It Actually Works

Bash variables are stored in the shell's own process memory as a flat table
of name/value string pairs — there's no int, float, or boolean type at the
storage layer. `age=30` stores the two-character string `"30"`. Arithmetic
contexts like `$(( ))`, `let`, and `[[ ... -lt ... ]]` don't change that
storage; they just ask bash's arithmetic evaluator to parse the string as a
number for the duration of that one expression, using base-10 by default
(a leading `0` triggers octal, `0x` triggers hex).

`export`ing a variable doesn't change how it's stored inside bash — it flags
that name/value pair to be copied into the **environment block** that gets
handed to every child process at `fork()`/`execve()` time. That's why a
non-exported variable is invisible to a subshell script you invoke, and why
each child gets its own private copy: environment variables are passed by
value at process-creation time, not shared memory, so a child mutating its
copy of `$PATH` never affects the parent shell.

Parameter expansion (`${name}`, `${name:-default}`) happens during the
shell's **word expansion** pass, which runs in a fixed order every time a
command line is parsed: brace expansion, tilde expansion, parameter/variable
expansion, arithmetic expansion, and command substitution all happen before
word splitting and globbing. That's why quoting matters — expansions inside
double quotes still happen, but the *result* is protected from the later
word-splitting and pathname-expansion steps.


## Exercise

Write `bmi.sh` that sets `weight_kg=70` and `height_m=1.75` as variables,
computes BMI using `awk` (since it needs decimals), and prints a sentence
like `Your BMI is 22.86`. Then add a third variable `unit="metric"` and print
it using double quotes to show correct expansion.
