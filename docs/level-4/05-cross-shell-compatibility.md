# 05 · Cross-Shell Compatibility

Not every system where your script runs has bash, and not every bash is
the same version. This module covers the practical differences between
POSIX `sh`, bash, and zsh, and how to write scripts that either run
correctly everywhere or fail with a clear message instead of a confusing
one.

## Why this matters

`#!/bin/sh` on Debian/Ubuntu is `dash`, not bash — a strict POSIX shell
with none of bash's extensions. macOS ships an old bash 3.2 (licensing,
not neglect) but makes zsh the default interactive shell. Alpine Linux
containers (common in Docker images) often have *only* `sh` (BusyBox
ash), no bash at all. A script that assumes "bash everywhere" will break
silently — or loudly — on any of these.

## Picking the right shebang

```bash
#!/bin/sh
# Use this when your script only needs POSIX features — most portable,
# runs on dash, ash, bash, and zsh's sh-compatibility mode.

#!/usr/bin/env bash
# Use this when you need bash-specific features (arrays, [[ ]], etc.)
# `env` finds bash on $PATH rather than assuming /bin/bash exists.
```

Never write `#!/bin/bash` directly on scripts meant to be portable —
bash isn't guaranteed to live at `/bin/bash` on every system (NixOS,
some BSDs). `#!/usr/bin/env bash` is the safer bash shebang.

## Bash-only features that break in POSIX `sh`

```bash
# arrays — NOT in POSIX sh, works in bash/zsh
names=(alice bob carol)
echo "${names[1]}"          # bob

# [[ ]] extended test — NOT in POSIX sh, use [ ] instead
[[ "$str" == *foo* ]]       # bash/zsh only
[ "${str#*foo*}" != "$str" ]  # POSIX-portable equivalent (rough)

# string manipulation shortcuts — NOT in POSIX sh
echo "${var^^}"              # bash 4+ only: uppercase
echo "${var,,}"               # bash 4+ only: lowercase

# process substitution — NOT in POSIX sh
diff <(sort file1) <(sort file2)   # bash/zsh only

# `local` is common but technically not POSIX (widely supported anyway)
my_func() { local x=1; }
```

## POSIX-safe equivalents

```bash
# instead of arrays, use a space/newline-separated string or positional params
names="alice bob carol"
for name in $names; do
    echo "$name"
done

# instead of [[ == glob ]], use case, which IS POSIX
case "$str" in
    *foo*) echo "matched" ;;
esac

# instead of ${var^^} (uppercase), use tr — works everywhere
upper=$(echo "$var" | tr '[:lower:]' '[:upper:]')

# instead of process substitution, use temp files or pipes
sort file1 > /tmp/s1
sort file2 > /tmp/s2
diff /tmp/s1 /tmp/s2
rm -f /tmp/s1 /tmp/s2
```

## `[[ ]]` vs `[ ]`

```bash
# [ ] is the POSIX test command — an ordinary program, needs careful quoting
if [ "$name" = "alice" ]; then echo "hi alice"; fi
[ -z "$var" ]           # true if $var is empty — ALWAYS quote $var in [ ]

# [[ ]] is a bash/zsh/ksh keyword — safer (no word-splitting surprises),
# supports pattern matching and && / || directly, but not portable to sh
if [[ "$name" == "alice" && -n "$id" ]]; then echo "hi"; fi
```

If a script must run under `sh`, use `[ ]` and combine conditions with
separate `[ ]` tests joined by `&&`/`||` rather than relying on `[[ ]]`.

## zsh-specific gotchas (vs. bash)

```bash
# zsh does NOT word-split unquoted variables by default (bash does)
files="a.txt b.txt"
for f in $files; do echo "$f"; done
# bash: two iterations (a.txt, b.txt)
# zsh:  ONE iteration ("a.txt b.txt") unless `setopt SH_WORD_SPLIT` is set

# array indexing: bash arrays are 0-indexed, zsh arrays are 1-indexed by default
arr=(a b c)
echo "${arr[1]}"
# bash: "b"   (index 1 is the second element)
# zsh:  "a"   (index 1 is the FIRST element)

# zsh has no `getopts`-free equivalent gap, but option-parsing built-ins
# (zparseopts) differ from bash's getopts entirely
```

If a codebase must support both, avoid relying on either shell's default
word-splitting or array-indexing behavior — use explicit quoting and
`"${arr[@]}"`-style expansion, which behaves consistently in both.

## Writing scripts that work in all three

```bash
#!/bin/sh
# portable-example.sh — runs under sh, bash, and zsh identically
set -eu   # note: no `pipefail` — it's not POSIX; see below

name="${1:-world}"

case "$name" in
    "") echo "name must not be empty" >&2; exit 1 ;;
esac

echo "Hello, $name!"

count=0
for item in one two three; do
    count=$((count + 1))
    echo "$count: $item"
done
```

## The `pipefail` problem

```bash
# `set -o pipefail` is a bash/zsh/ksh extension — NOT in POSIX sh (dash
# silently ignores it rather than erroring, which is its own trap)
set -euo pipefail   # fine in bash, but do not rely on it under #!/bin/sh
```

If a script must be POSIX-portable AND needs pipeline-failure detection,
check each command in the pipeline separately, or use
`PIPESTATUS`/`pipestatus` (bash/zsh only, different array names) instead
of `pipefail`.

## Detecting which shell you're actually running under

```bash
if [ -n "${BASH_VERSION:-}" ]; then
    echo "running under bash $BASH_VERSION"
elif [ -n "${ZSH_VERSION:-}" ]; then
    echo "running under zsh $ZSH_VERSION"
else
    echo "running under a plain POSIX sh"
fi
```

This is useful in shared library scripts (`source`d from `.bashrc` and
`.zshrc` alike) that need to branch on shell-specific syntax.

## Cheat sheet

| Feature | POSIX `sh` | bash | zsh (default opts) |
|---------|:---:|:---:|:---:|
| Arrays `arr=(a b)` | no | yes, 0-indexed | yes, 1-indexed |
| `[[ ]]` extended test | no | yes | yes |
| `${var^^}` / `${var,,}` | no | yes (bash 4+) | no |
| Process substitution `<(...)` | no | yes | yes |
| `set -o pipefail` | no | yes | yes |
| Unquoted var word-splitting | yes | yes | no (needs `SH_WORD_SPLIT`) |
| `local` in functions | not POSIX, widely supported | yes | yes |
| Safe cross-shell test | `case`/`[ ]` | anything | anything except relying on word-splitting |

## Exercise

Take the `backup.sh` script from Level 1 (or any bash script you've
written) and identify every bash-only construct it uses (arrays,
`[[ ]]`, `${var^^}`, etc.). Rewrite a POSIX-`sh`-compatible version using
only `case`, `[ ]`, and `tr`/`sed` as needed, then verify it with
`sh ./backup-posix.sh` (or `dash ./backup-posix.sh` if installed) to
confirm it runs without bash-specific errors.
