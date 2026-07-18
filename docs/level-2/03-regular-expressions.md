# 03 · Regular Expressions in Bash

Regular expressions let you match *patterns* in text rather than exact
strings — essential for validating input, filtering logs, and extracting
structured data out of otherwise messy text.

## `grep -E` — extended regular expressions

```bash
echo "error: connection timed out" | grep -E "error|warning"
# error: connection timed out    (matches "error" OR "warning")

echo "user123" | grep -E "^[a-z]+[0-9]+$"
# user123    (letters followed by digits, whole string)
```

```bash
# common quantifiers
grep -E "colou?r" file.txt        # ? = 0 or 1 of the preceding token: color OR colour
grep -E "ab+c" file.txt            # + = 1 or more: abc, abbc, abbbc...
grep -E "ab*c" file.txt            # * = 0 or more: ac, abc, abbc...
grep -E "a{2,4}" file.txt          # {2,4} = between 2 and 4 repetitions
```

Without `-E`, `grep` uses "basic" regex where `+`, `?`, `|`, and `{}` need a
backslash to be special (`grep "ab\+c"`) — `-E` (or the `egrep` alias) gives
you the more readable "extended" syntax used throughout this module.

## Anchors and character classes

```bash
grep -E "^#" script.sh            # lines starting with #  (comments)
grep -E ";$" script.sh              # lines ending with ;
grep -E "^\s*$" file.txt             # blank or whitespace-only lines

grep -E "[0-9]{3}-[0-9]{4}" contacts.txt   # phone-number-shaped patterns like 555-1234
grep -E "[[:alpha:]]+" file.txt              # POSIX class: alphabetic characters
grep -E "[[:digit:]]+" file.txt              # POSIX class: digits
```

| Class | Meaning |
|-------|---------|
| `^` | start of line |
| `$` | end of line |
| `.` | any single character |
| `[abc]` | any one of a, b, c |
| `[^abc]` | any character except a, b, c |
| `[[:alpha:]]` | any letter |
| `[[:digit:]]` | any digit |
| `[[:space:]]` | any whitespace |

## The `=~` operator — regex matching inside `[[ ]]`

```bash
input="user@example.com"

if [[ "$input" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
    echo "valid email format"
else
    echo "invalid email format"
fi
```

`=~` lets you regex-match directly in a conditional, no external `grep`
process needed — the pattern on the right side should be **unquoted** (or
bash treats it as a literal string instead of a regex).

## Capture groups with `=~` and `BASH_REMATCH`

```bash
log_line="2026-07-18 14:32:01 ERROR Connection refused"

if [[ "$log_line" =~ ^([0-9-]+)\ ([0-9:]+)\ ([A-Z]+)\ (.*)$ ]]; then
    echo "date:    ${BASH_REMATCH[1]}"
    echo "time:    ${BASH_REMATCH[2]}"
    echo "level:   ${BASH_REMATCH[3]}"
    echo "message: ${BASH_REMATCH[4]}"
fi
# date:    2026-07-18
# time:    14:32:01
# level:   ERROR
# message: Connection refused
```

`BASH_REMATCH[0]` holds the whole match; `BASH_REMATCH[1]`, `[2]`, ... hold
each parenthesized capture group, in order — this is the cleanest way to
pull structured fields out of a line of text without spawning `awk`.

## `grep` capture groups with `-o` and `-E`

```bash
echo "Contact: alice@example.com" | grep -oE "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"
# alice@example.com
```

`-o` prints only the matched portion of each line instead of the whole line
— handy for extracting IDs, IPs, or emails out of larger text.

## Practical validation patterns

```bash
is_number() { [[ "$1" =~ ^-?[0-9]+$ ]]; }
is_ipv4()   { [[ "$1" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; }
is_hex_color() { [[ "$1" =~ ^#[0-9a-fA-F]{6}$ ]]; }

is_number "-42" && echo "valid integer"
is_ipv4 "192.168.1.1" && echo "looks like an IPv4 address"
is_hex_color "#3a7bd5" && echo "valid hex color"
```

Note `is_ipv4` here only checks *shape*, not that each octet is `<= 255` —
real validation would combine the regex with a numeric range check per
octet.

## Cheat sheet

| Pattern | Matches |
|---------|---------|
| `^` / `$` | start / end of line |
| `.` | any character |
| `*` / `+` / `?` | 0+, 1+, 0-or-1 of the preceding token |
| `{n,m}` | between n and m repetitions |
| `[...]` / `[^...]` | a character class / its negation |
| `(...)` | a capture group |
| `a\|b` | alternation: a OR b |
| `[[ $s =~ regex ]]` | match `$s` against `regex`, populate `BASH_REMATCH` |

## Exercise

Write `validate_log.sh` that reads a log file line by line, uses `=~` with a
capture-group regex to extract the timestamp, level, and message from lines
shaped like `2026-07-18 14:32:01 ERROR Connection refused`, and prints only
the lines where the level is `ERROR` or `CRITICAL`, reformatted as
`[LEVEL] message (at timestamp)`.
