# 02 · String Manipulation

Bash's **parameter expansion** syntax (the `${var...}` forms) does an
enormous amount of string work — substrings, search-and-replace, case
conversion, trimming — without ever calling out to an external tool like
`sed` or `awk`. Knowing this well means shorter, faster scripts.

## Length and substrings

```bash
str="Hello, World!"

echo "${#str}"          # 13              — length
echo "${str:7}"          # World!          — from index 7 to the end
echo "${str:7:5}"        # World           — 5 chars starting at index 7
echo "${str: -6}"        # World!          — last 6 chars (space before - is required)
echo "${str:0:5}"        # Hello           — first 5 chars
```

## Search and replace

```bash
path="/usr/local/bin/script.sh"

echo "${path/bin/BIN}"     # /usr/local/BIN/script.sh   — replace first match
echo "${path//\//_}"        # _usr_local_bin_script.sh   — replace ALL matches
echo "${path/#\/usr/~}"      # ~/local/bin/script.sh      — replace only if it's a PREFIX match
echo "${path/%.sh/.bak}"     # /usr/local/bin/script.bak  — replace only if it's a SUFFIX match
```

| Form | Replaces |
|------|----------|
| `${var/pattern/repl}` | first match only |
| `${var//pattern/repl}` | every match |
| `${var/#pattern/repl}` | match only at the start of the string |
| `${var/%pattern/repl}` | match only at the end of the string |

## Removing prefixes and suffixes

```bash
file="archive.tar.gz"

echo "${file%.gz}"       # archive.tar     — strip shortest match from the end
echo "${file%%.*}"        # archive          — strip longest match from the end
echo "${file#*.}"         # tar.gz           — strip shortest match from the start
echo "${file##*.}"        # gz               — strip longest match from the start (common "get extension" trick)
```

`#`/`##` trim from the **front**, `%`/`%%` trim from the **back**; the single
form is the *shortest* matching trim, the doubled form is the *longest*.
`${file##*.}` — "strip everything up to and including the last dot" — is the
idiomatic way to get a file extension in pure bash.

## Case conversion (bash 4+)

```bash
name="john SMITH"

echo "${name^^}"     # JOHN SMITH     — uppercase everything
echo "${name,,}"     # john smith     — lowercase everything
echo "${name^}"       # John SMITH     — uppercase only the first character
echo "${name,}"       # john SMITH     — lowercase only the first character
```

## Default and alternate values

```bash
unset name
echo "${name:-anonymous}"     # anonymous   — use default if unset/empty (doesn't assign)
echo "${name:=anonymous}"     # anonymous   — same, but ALSO assigns it to name
echo "$name"                    # anonymous   — now it's set

greeting=""
echo "${greeting:+set}"          # (empty)     — ":+" only substitutes if the var IS set/non-empty
name="Ada"
echo "${name:+set}"                # set
```

## Trimming whitespace

```bash
trim() {
    local s="$1"
    s="${s#"${s%%[![:space:]]*}"}"   # strip leading whitespace
    s="${s%"${s##*[![:space:]]}"}"   # strip trailing whitespace
    echo "$s"
}

trimmed=$(trim "   hello world   ")
echo "[$trimmed]"      # [hello world]
```

That pattern looks dense, but it's the standard pure-bash trim idiom — for
anything more elaborate, reaching for `sed` or `awk` (Level 3) is usually
clearer.

## Splitting a string into parts

```bash
csv_line="alice,30,engineer"

IFS=',' read -r name age role <<< "$csv_line"
echo "$name is $age and works as a(n) $role"
# alice is 30 and works as a(n) engineer
```

```bash
# splitting into an array
IFS=',' read -ra fields <<< "$csv_line"
echo "${fields[1]}"     # 30
```

## Joining an array into a string

```bash
parts=("2026" "07" "18")
joined=$(IFS=-; echo "${parts[*]}")
echo "$joined"     # 2026-07-18
```

`"${parts[*]}"` (unlike `"${parts[@]}"`) joins elements using the first
character of `IFS`, which is exactly what's needed here.

## Cheat sheet

| Expansion | Meaning |
|-----------|---------|
| `${#s}` | length of `s` |
| `${s:i:n}` | substring, `n` chars from index `i` |
| `${s/old/new}` | replace first match |
| `${s//old/new}` | replace all matches |
| `${s#pattern}` / `${s##pattern}` | trim shortest/longest match from the front |
| `${s%pattern}` / `${s%%pattern}` | trim shortest/longest match from the back |
| `${s^^}` / `${s,,}` | uppercase / lowercase all |
| `${s:-default}` | use default if unset/empty |
| `${s:=default}` | use AND assign default if unset/empty |

## Exercise

Write `slugify.sh` that takes a string like `"$1"` (e.g. `"Hello, World! It's a Great Day"`),
and prints a URL-safe slug: lowercase, spaces replaced with hyphens, and all
characters that aren't letters, digits, or hyphens removed — using only
parameter expansion (no `sed`/`tr` calls). Test it on a few different inputs.
