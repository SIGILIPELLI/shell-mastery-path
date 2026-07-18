# 01 · Arrays & Associative Arrays

Level 1 dealt only with single-value variables. Real scripts often need to
hold a *collection* of things — a list of filenames, a set of hostnames, a
lookup table of config keys. Bash has two kinds of arrays: **indexed arrays**
(ordered, numbered from 0) and **associative arrays** (unordered key/value
maps, like a dictionary).

## Creating indexed arrays

```bash
# explicit declaration
declare -a fruits

# or just assign directly — bash infers it's an array
fruits=("apple" "banana" "cherry")

echo "${fruits[0]}"     # apple
echo "${fruits[1]}"     # banana
```

```bash
# appending elements
fruits+=("date")
echo "${fruits[@]}"     # apple banana cherry date

# assigning to a specific index (sparse arrays are allowed)
fruits[10]="fig"
```

## Reading arrays

```bash
fruits=("apple" "banana" "cherry")

echo "${fruits[@]}"      # all elements: apple banana cherry
echo "${fruits[*]}"      # all elements joined by the first char of IFS
echo "${#fruits[@]}"     # length: 3
echo "${!fruits[@]}"     # all indices: 0 1 2
```

```bash
# looping over an array — always quote "${arr[@]}"
for fruit in "${fruits[@]}"; do
    echo "I like $fruit"
done
```

As with `"$@"`, quoting `"${fruits[@]}"` preserves each element as its own
word even if it contains spaces — unquoted expansion word-splits and breaks
on values like `"New York"`.

## Slicing arrays

```bash
numbers=(10 20 30 40 50)

echo "${numbers[@]:1:3}"    # 20 30 40   (start at index 1, take 3)
echo "${numbers[@]:2}"      # 30 40 50   (from index 2 to the end)
echo "${numbers[-1]}"       # 50          (last element, bash 4.3+)
```

## Removing elements

```bash
fruits=("apple" "banana" "cherry")

unset 'fruits[1]'           # removes banana, LEAVES A GAP at index 1
echo "${fruits[@]}"          # apple cherry
echo "${!fruits[@]}"         # 0 2   <- note the gap

# re-index to close gaps
fruits=("${fruits[@]}")
echo "${!fruits[@]}"         # 0 1
```

## Associative arrays (`declare -A`)

Associative arrays must be declared before use — unlike indexed arrays,
bash can't infer them from a bare assignment.

```bash
declare -A capitals

capitals["France"]="Paris"
capitals["Japan"]="Tokyo"
capitals[Germany]="Berlin"    # quotes around the key are optional if it's a single word

echo "${capitals["Japan"]}"   # Tokyo
```

```bash
# building one in a single declaration
declare -A colors=(
    [red]="#ff0000"
    [green]="#00ff00"
    [blue]="#0000ff"
)

echo "${colors[red]}"    # #ff0000
```

## Iterating an associative array

```bash
declare -A capitals=(
    [France]="Paris"
    [Japan]="Tokyo"
    [Germany]="Berlin"
)

for country in "${!capitals[@]}"; do
    echo "$country -> ${capitals[$country]}"
done
# order is NOT guaranteed — associative arrays are unordered in bash
```

```bash
# checking whether a key exists
if [[ -v capitals["Italy"] ]]; then
    echo "we have Italy"
else
    echo "no entry for Italy"
fi
```

`-v` (bash 4.2+) tests whether a variable — or here, an array key — is set,
without triggering an "unbound variable" error under `set -u`.

## A practical example: counting word frequency

```bash
#!/usr/bin/env bash
declare -A counts

text="the quick brown fox jumps over the lazy dog the fox runs"

for word in $text; do
    ((counts["$word"]++))
done

for word in "${!counts[@]}"; do
    echo "$word: ${counts[$word]}"
done | sort -t: -k2 -rn
# the: 3
# fox: 2
# ...
```

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `arr=(a b c)` | create an indexed array |
| `declare -A arr` | declare an associative array (required before use) |
| `${arr[i]}` | element at index/key `i` |
| `${arr[@]}` | all elements |
| `${#arr[@]}` | number of elements |
| `${!arr[@]}` | all indices/keys |
| `arr+=(x)` | append `x` to an indexed array |
| `unset 'arr[i]'` | remove element at index/key `i` |
| `${arr[@]:start:len}` | slice |

## Exercise

Write `inventory.sh` that builds an associative array mapping item names to
quantities (e.g. `apples=12`, `bananas=7`, `oranges=20`), prints each item
and quantity sorted by quantity descending, and prints the total quantity
across all items using a running sum in a loop.
