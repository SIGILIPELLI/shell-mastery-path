# 01 · Advanced sed & awk

`sed` (stream editor) and `awk` (a full text-processing language) can
replace long chains of `grep`/`cut`/`sort` with a single, precise command.
This module goes beyond Level 1's basics into multi-line processing, full
awk scripts, and safe in-place editing.

## sed: substitution recap and flags

```bash
sed 's/foo/bar/' file.txt        # replace first match per line
sed 's/foo/bar/g' file.txt        # replace ALL matches per line
sed 's/foo/bar/2' file.txt         # replace only the 2nd match per line
sed 's/foo/bar/gi' file.txt         # case-insensitive, all matches
```

## sed: addressing specific lines

```bash
sed '3d' file.txt              # delete line 3
sed '2,4d' file.txt              # delete lines 2 through 4
sed '/^#/d' file.txt              # delete lines starting with #  (comments)
sed -n '5,10p' file.txt            # print ONLY lines 5-10 (-n suppresses default output)
sed '$d' file.txt                    # delete the LAST line
sed '1!d' file.txt                    # print only line 1 (opposite of 1d)
```

## sed: in-place editing (with a safety net)

```bash
sed -i.bak 's/localhost/production.example.com/g' config.ini
# .bak suffix keeps a backup — config.ini.bak — before overwriting the original

# GNU sed (Linux): no space between -i and the suffix
sed -i.bak 's/foo/bar/' file.txt
# BSD sed (macOS): same syntax, but -i REQUIRES an argument (even '' for none)
sed -i '' 's/foo/bar/' file.txt
```

Always test a `sed -i` command without `-i` first (let it print to stdout)
to confirm the pattern matches what you expect — an in-place edit that goes
wrong has no undo unless you kept a `.bak`.

## sed: multi-line processing

```bash
# join every 2 lines into 1 (N appends the next line into the pattern space)
sed 'N;s/\n/ /' file.txt
```

```bash
# delete a block of lines between two markers (inclusive)
sed '/START/,/END/d' file.txt
```

```bash
# print only the text BETWEEN two markers, excluding the markers themselves
sed -n '/START/,/END/{/START/d;/END/d;p}' file.txt
```

## sed: capture groups and backreferences

```bash
echo "John Smith" | sed -E 's/([A-Za-z]+) ([A-Za-z]+)/\2, \1/'
# Smith, John

echo "2026-07-18" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# 18/07/2026
```

`-E` enables extended regex (matching `grep -E` from Level 2); `\1`, `\2`,
... refer to each parenthesized capture group in the replacement.

## awk: the field-based model

```bash
# awk automatically splits each line into $1, $2, ... by whitespace ($0 = whole line)
echo "alice 30 engineer" | awk '{ print $1, $3 }'
# alice engineer
```

```bash
awk -F, '{ print $2 }' data.csv        # -F sets the field separator (comma here)
awk -F: '{ print $1 }' /etc/passwd       # list usernames
```

## awk: patterns and actions

```bash
# pattern { action } — action runs only on lines matching the pattern
awk '/ERROR/ { print $0 }' app.log             # print lines containing ERROR
awk '$3 > 100 { print $1, $3 }' data.txt         # print rows where field 3 exceeds 100
awk 'NR == 1 { print }' file.txt                    # print only the first line (NR = record/line number)
awk 'NF > 3' file.txt                                 # print lines with more than 3 fields
```

## awk: built-in variables

| Variable | Meaning |
|----------|---------|
| `$0` | the whole current line |
| `$1`, `$2`, ... | individual fields |
| `NF` | number of fields on the current line |
| `NR` | current line/record number (across all input) |
| `FS` | input field separator (default: whitespace) |
| `OFS` | output field separator (used when reassembling `$0`) |

## awk: full scripts with BEGIN/END and variables

```bash
awk '
BEGIN { FS = ","; OFS = "\t"; total = 0 }
{
    total += $3
    print $1, $2, $3
}
END { print "TOTAL:", total }
' sales.csv
```

`BEGIN` runs once before any input is read (good for setting up separators
and initial values); `END` runs once after all input is processed (good for
final totals/summaries); the middle block runs once per line.

## awk: a practical report

```bash
# summarize response codes from a web server log, sorted by count
awk '{ count[$9]++ } END { for (code in count) print code, count[code] }' access.log \
    | sort -k2 -rn
```

```bash
# compute the average of column 2 across a CSV
awk -F, '{ sum += $2; n++ } END { if (n > 0) print "average:", sum/n }' data.csv
```

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `sed 's/x/y/' ` | substitute first match per line |
| `sed 's/x/y/g'` | substitute all matches per line |
| `sed -n 'N,Mp'` | print only lines N through M |
| `sed -i.bak 's/x/y/'` | in-place edit with a backup |
| `sed '/A/,/B/d'` | delete a block between two markers |
| `awk '{print $N}'` | print field N |
| `awk -F, ...` | set the field separator |
| `awk 'pattern {action}'` | run action only on matching lines |
| `awk 'BEGIN{...} {...} END{...}'` | setup / per-line / final summary |

## Exercise

Given a CSV file `employees.csv` with columns `name,department,salary`,
write a single `awk` command that prints the average salary per department
(hint: use a per-department running total and count in two associative
arrays, then loop over them in `END`). Then write a `sed` one-liner that
converts every line's `department` field to uppercase.
