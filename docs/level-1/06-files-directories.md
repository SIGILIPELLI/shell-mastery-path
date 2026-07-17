# 06 · Working with Files & Directories

## Creating, copying, moving, removing

```bash
mkdir logs                  # create a directory
mkdir -p data/raw/2026       # create nested directories, no error if they exist

touch notes.txt               # create an empty file (or update its timestamp)

cp notes.txt notes.bak        # copy a file
cp -r logs logs_backup        # copy a directory recursively

mv notes.bak archive/notes.bak   # move (or rename) a file
mv old_name.txt new_name.txt     # mv is also how you rename

rm notes.txt                  # remove a file
rm -r logs_backup             # remove a directory recursively
rm -rf temp_dir                # remove without confirmation, ignore missing — use carefully
```

`rm -rf` is permanent and has no undo — always double-check the path (and
consider `set -e` from Module 9, plus quoting variables) before using it in a
script.

## File test operators

These pair with `if [[ ... ]]` from Module 3 to make scripts that check
before acting:

```bash
path="config.yml"

if [[ -e "$path" ]]; then echo "exists"; fi
if [[ -f "$path" ]]; then echo "is a regular file"; fi
if [[ -d "$path" ]]; then echo "is a directory"; fi
if [[ -L "$path" ]]; then echo "is a symlink"; fi
if [[ -r "$path" ]]; then echo "readable"; fi
if [[ -w "$path" ]]; then echo "writable"; fi
if [[ -x "$path" ]]; then echo "executable"; fi
if [[ -s "$path" ]]; then echo "non-empty"; fi
```

| Operator | True when... |
|----------|---------------|
| `-e` | path exists (any type) |
| `-f` | path exists and is a regular file |
| `-d` | path exists and is a directory |
| `-L` | path exists and is a symbolic link |
| `-r` / `-w` / `-x` | readable / writable / executable |
| `-s` | exists and has size greater than zero |
| `-nt` / `-ot` | file is newer/older than another (e.g. `a -nt b`) |

## Listing and inspecting

```bash
ls -la                 # long listing, including hidden files
ls -lh                 # human-readable sizes (K, M, G)
find . -name "*.log"    # find files by name pattern, recursively
find . -type d -name "tmp*"    # find directories matching a pattern
du -sh some_dir         # total size of a directory, human-readable
stat notes.txt           # detailed metadata: size, permissions, timestamps
```

## Permissions

```bash
chmod +x deploy.sh        # add execute permission for everyone
chmod 755 deploy.sh        # rwxr-xr-x explicitly (owner rwx, group/other rx)
chmod 644 config.yml       # rw-r--r-- — typical for a non-executable data file
chmod u+w,go-w shared.txt  # owner gets write, group/other lose write

chown alice notes.txt       # change file owner (may need sudo)
```

Permission digits are `read=4, write=2, execute=1`, added together per
category (owner, group, other) — `755` = owner `7` (4+2+1=rwx), group `5`
(4+1=r-x), other `5` (r-x).

## Building paths safely in scripts

```bash
base_dir="/var/log/myapp"
today=$(date +%F)                 # 2026-07-18
log_file="$base_dir/$today.log"

mkdir -p "$base_dir"                # ensure the directory exists first
touch "$log_file"
echo "writing to $log_file"
```

Always quote path variables (`"$base_dir"`) — an unquoted path containing a
space will be split into multiple arguments by commands like `mkdir` or `rm`.

## Looping over files safely

```bash
# Good: handles filenames with spaces correctly
for file in *.csv; do
    [[ -e "$file" ]] || continue   # skip if the glob matched nothing
    echo "processing $file"
done
```

```bash
# For recursive processing, prefer find -print0 piped to a null-delimited read
find . -name "*.log" -print0 | while IFS= read -r -d '' file; do
    echo "log: $file"
done
```

`-print0` / `-d ''` use the NUL byte as a separator instead of newlines — the
only fully safe way to handle filenames that might contain spaces, or even
newlines.

## Exercise

Write `organize.sh` that creates a directory called `sorted/`, then loops
over every file in the current directory and moves each one into a
subdirectory of `sorted/` named after its extension (e.g. `report.pdf` goes
to `sorted/pdf/report.pdf`). Skip directories and handle files with no
extension by putting them in `sorted/other/`.
