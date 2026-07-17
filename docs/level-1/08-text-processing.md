# 08 · Basic Text Processing

Four tools cover the vast majority of everyday shell text processing:
`grep` (find lines), `cut` (extract columns), `sort` (order lines), and
`sed`/`awk` (transform lines). This module introduces all of them; Level 3
goes deep on advanced `sed`/`awk`.

## grep — find lines matching a pattern

```bash
grep "error" app.log             # lines containing "error"
grep -i "error" app.log            # case-insensitive
grep -v "debug" app.log            # invert: lines NOT containing "debug"
grep -c "error" app.log            # count matching lines
grep -n "error" app.log            # show line numbers
grep -r "TODO" src/                 # recursive search through a directory
grep -l "TODO" src/*.sh             # just list matching filenames
grep -E "error|warning" app.log     # extended regex: match either pattern
grep -w "cat" pets.txt              # match whole word only, not "category"
```

```bash
# Common combo: how many times does each user appear in a log?
grep "login" auth.log | wc -l
```

## cut — extract columns/fields

```bash
# /etc/passwd is colon-delimited: root:x:0:0:root:/root:/bin/bash
cut -d: -f1 /etc/passwd            # first field only (usernames)
cut -d: -f1,3 /etc/passwd           # fields 1 and 3
cut -d: -f1-3 /etc/passwd            # fields 1 through 3
```

```bash
# CSV example
echo "name,age,city" | cut -d, -f2   # age
```

`-d` sets the delimiter (default is tab), `-f` selects which field(s).

## sort — order lines

```bash
sort names.txt                  # alphabetical order
sort -r names.txt                # reverse order
sort -n numbers.txt               # numeric order (not lexicographic!)
sort -k2 data.txt                  # sort by the 2nd whitespace-separated field
sort -t: -k3 -n /etc/passwd         # sort by 3rd colon-delimited field, numerically
sort -u names.txt                   # sort AND remove duplicate lines
```

Without `-n`, `sort` compares text lexicographically, so `10` sorts before
`2` — always add `-n` for numeric data.

## uniq — collapse adjacent duplicate lines

`uniq` only removes duplicates that are **adjacent**, so it's almost always
paired with `sort` first:

```bash
sort names.txt | uniq              # unique names
sort names.txt | uniq -c            # count of each unique name
sort names.txt | uniq -c | sort -rn  # most frequent names first
```

## sed — stream editor (simple substitutions)

```bash
sed 's/foo/bar/' file.txt           # replace first "foo" per line with "bar"
sed 's/foo/bar/g' file.txt           # replace ALL occurrences per line
sed -i 's/foo/bar/g' file.txt         # edit the file IN PLACE
sed -i.bak 's/foo/bar/g' file.txt      # in-place, but keep file.txt.bak as backup
sed -n '2,4p' file.txt                  # print only lines 2 through 4
sed '/^#/d' config.txt                   # delete lines starting with #  (comments)
```

Always test a `sed` command without `-i` first (let it print to the screen)
before committing to an in-place edit — there's no undo once `-i` runs
without a backup suffix.

## awk — column-aware processing

`awk` treats each input line as a record split into fields (`$1`, `$2`, ...,
with `$0` meaning the whole line):

```bash
awk '{ print $1 }' data.txt              # print the first whitespace field
awk -F, '{ print $2 }' data.csv           # -F sets the field separator to comma
awk '{ print $1, $3 }' data.txt            # print fields 1 and 3
awk '{ sum += $2 } END { print sum }' sales.txt   # sum a column
awk '$3 > 100 { print $1 }' data.txt        # print field 1 where field 3 > 100
awk 'NR == 1' file.txt                        # print just the first line (NR = record number)
```

## Putting it together

```bash
# Top 5 most frequent IP addresses in an access log
awk '{ print $1 }' access.log | sort | uniq -c | sort -rn | head -5
```

```bash
# Extract, filter, and reformat: usernames with UID >= 1000
awk -F: '$3 >= 1000 { print $1 }' /etc/passwd | sort
```

## Cheat sheet

| Tool | Job |
|------|-----|
| `grep` | find lines matching a pattern |
| `cut` | pull out specific columns by delimiter/position |
| `sort` | order lines (`-n` numeric, `-r` reverse, `-k` by field) |
| `uniq` | collapse adjacent duplicates (`-c` to count) |
| `sed` | stream-edit lines (substitute, delete, print ranges) |
| `awk` | field-aware processing, math, conditional printing |

## Exercise

Given a CSV file `sales.csv` with header `name,amount,region`, write a
one-liner (or short script) using `awk` to sum `amount` per `region` and
print `region: total` sorted by total descending. Then use `grep` and `wc -l`
to count how many rows have `amount` greater than 1000 (hint: combine `awk`'s
filter with `wc -l`).
