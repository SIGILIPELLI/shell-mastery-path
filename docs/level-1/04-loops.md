# 04 · Loops

## for loops — iterating over a list

```bash
for fruit in apple banana cherry; do
    echo "I like $fruit"
done
```

```bash
# Iterating over files matching a glob
for file in *.txt; do
    echo "found: $file"
done
```

```bash
# C-style for loop, when you need a numeric counter
for (( i = 1; i <= 5; i++ )); do
    echo "count: $i"
done
```

```bash
# A range with brace expansion
for i in {1..5}; do
    echo "number $i"
done

for i in {0..10..2}; do   # step of 2
    echo "$i"
done
```

## while loops — run while a condition holds

```bash
count=1
while [[ $count -le 5 ]]; do
    echo "count is $count"
    ((count++))
done
```

```bash
# Reading a file line by line — the idiomatic pattern
while IFS= read -r line; do
    echo "line: $line"
done < "input.txt"
```

`IFS=` (empty) prevents leading/trailing whitespace from being stripped, and
`read -r` prevents backslash escaping — together this is the safest way to
read a file line by line in Bash.

## until loops — run until a condition becomes true

`until` is the mirror image of `while`: it keeps looping **while the
condition is false**.

```bash
count=1
until [[ $count -gt 5 ]]; do
    echo "count is $count"
    ((count++))
done
```

## break and continue

```bash
for i in {1..10}; do
    if (( i == 5 )); then
        break            # exit the loop entirely
    fi
    echo "$i"
done
# prints 1 2 3 4

for i in {1..5}; do
    if (( i % 2 == 0 )); then
        continue         # skip the rest of this iteration
    fi
    echo "$i"
done
# prints 1 3 5 (odd numbers only)
```

## Infinite loops (with a controlled exit)

```bash
count=0
while true; do
    echo "iteration $count"
    ((count++))
    if (( count >= 3 )); then
        break
    fi
done
```

`while true; do ... done` is the standard pattern for "loop forever until I
explicitly break" — common in monitoring scripts (see Level 2's log-monitor
project).

## Looping over command output

```bash
for user in $(cut -d: -f1 /etc/passwd); do
    echo "system user: $user"
done
```

```bash
# Safer alternative for lines that might contain spaces: while + process substitution
while IFS= read -r line; do
    echo "entry: $line"
done < <(ls -la)
```

The second form avoids the word-splitting pitfalls of `for x in $(...)` when
values can contain spaces — prefer `while read` for anything beyond simple,
space-free tokens.

## Cheat sheet

| Loop | Runs while... |
|------|----------------|
| `for x in list; do ... done` | once per item in the list |
| `for (( i=0; i<n; i++ )); do ... done` | C-style numeric counting |
| `while [[ cond ]]; do ... done` | condition is true |
| `until [[ cond ]]; do ... done` | condition is false |
| `break` | — exits the loop immediately |
| `continue` | — skips to the next iteration |

## Exercise

Write `countdown.sh` that takes a starting number as `$1`, counts down to 1
printing each number with a `while` loop, and prints `"Liftoff!"` at the end.
Then write `evens.sh` that uses a `for` loop over `{1..20}` and `continue` to
print only the even numbers.
