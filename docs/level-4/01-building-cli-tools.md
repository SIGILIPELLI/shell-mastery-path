# 01 · Building Full CLI Tools

A script that only works when called exactly one way isn't a tool, it's a
trap. This module covers what turns a script into something other people
(including future you) can pick up and use: consistent flag parsing,
`--help` text that actually helps, and `getopts` as the standard way to
parse short options in POSIX-compatible shells.

## The shape of a real CLI tool

```bash
#!/usr/bin/env bash
# deploy.sh — example of a well-shaped CLI tool
set -euo pipefail

PROG=$(basename "$0")
VERSION="1.2.0"

usage() {
    cat <<EOF
Usage: $PROG [OPTIONS] <target>

Deploy the application to a target environment.

Arguments:
  target            environment name (staging|production)

Options:
  -e, --env FILE    load environment variables from FILE
  -f, --force       skip confirmation prompts
  -v, --verbose     print detailed progress
  -h, --help        show this help and exit
      --version     show version and exit

Examples:
  $PROG staging
  $PROG --force --env prod.env production
EOF
}
```

A predictable `usage()` function is the single biggest readability win you
can add to a script. Every flag, every argument, one example — that's the
convention this module builds toward.

## Manual flag parsing (simple cases)

For a handful of long-only flags, a plain `while`/`case` loop is often
enough and needs no extra library:

```bash
verbose=false
force=false

while [[ $# -gt 0 ]]; do
    case "$1" in
        --verbose) verbose=true; shift ;;
        --force)   force=true; shift ;;
        --help)    usage; exit 0 ;;
        --)        shift; break ;;      # explicit end-of-options marker
        -*)        echo "Unknown option: $1" >&2; usage; exit 1 ;;
        *)         break ;;              # first positional argument
    esac
done

target="${1:-}"
```

This scales poorly once you need combined short flags (`-vf`) or flags
with attached values (`-e prod.env`) — that's where `getopts` earns its
keep.

## `getopts` deep dive

`getopts` is the POSIX-standard way to parse short options and is built
into `sh`, `bash`, and `zsh`. The option string encodes which flags take
an argument: a letter followed by `:` means "this flag consumes the next
word as its value."

```bash
#!/usr/bin/env bash
set -euo pipefail

env_file=""
force=false
verbose=false

# "e:fvh" — 'e' takes a value (the ':'), f/v/h are boolean switches
while getopts ":e:fvh" opt; do
    case "$opt" in
        e) env_file="$OPTARG" ;;      # OPTARG holds the value after -e
        f) force=true ;;
        v) verbose=true ;;
        h) usage; exit 0 ;;
        \?) echo "Invalid option: -$OPTARG" >&2; usage; exit 1 ;;
        :) echo "Option -$OPTARG requires an argument" >&2; exit 1 ;;
    esac
done
shift $((OPTIND - 1))   # drop parsed options, leaving positional args in $@

target="${1:-}"
[[ -z "$target" ]] && { echo "Error: <target> is required" >&2; usage; exit 1; }
```

Key details:

- The **leading `:`** in `":e:fvh"` switches `getopts` into "silent error"
  mode, so you handle bad flags yourself via the `\?` and `:` cases
  instead of it printing its own message.
- **`OPTIND`** tracks the index of the next argument to process; resetting
  the positional parameters with `shift $((OPTIND - 1))` is what makes
  `$1`, `$2`, ... refer to the real positional arguments afterward.
- `getopts` only understands **short options** (`-e`) and bundled short
  flags (`-vf` = `-v -f`); it has no built-in notion of `--long-options`.

## Combining getopts with long options

A common pattern is to translate long flags to short ones before the
`getopts` loop runs, giving you both without a third-party library:

```bash
args=()
for arg in "$@"; do
    case "$arg" in
        --env)     args+=("-e") ;;
        --force)   args+=("-f") ;;
        --verbose) args+=("-v") ;;
        --help)    args+=("-h") ;;
        *)         args+=("$arg") ;;
    esac
done
set -- "${args[@]}"   # replace $@ with the translated argument list

while getopts ":e:fvh" opt; do
    # ... same as above
    :
done
```

## Argument-parsing frameworks

For larger tools, hand-rolled parsing gets unwieldy. Two common approaches:

**`docopt`-style** — write the `usage()` text first, generate the parser
from it (via a language binding), so help text and behavior can never
drift apart.

**Reusable parse function** — define an associative array of flag specs
and loop over it once, keeping every script's parsing logic identical:

```bash
declare -A OPTS=( [force]=false [verbose]=false [env]="" )

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --force)   OPTS[force]=true; shift ;;
            --verbose) OPTS[verbose]=true; shift ;;
            --env)     OPTS[env]="$2"; shift 2 ;;
            *)         POSITIONAL+=("$1"); shift ;;
        esac
    done
}
POSITIONAL=()
parse_args "$@"
set -- "${POSITIONAL[@]}"
```

This is worth extracting into a shared `lib/args.sh` once you maintain
more than two or three CLI tools, so every tool parses flags the same way.

## Exit codes and standard streams

A well-behaved CLI tool follows two conventions that make it composable
with other tools:

```bash
# errors and diagnostics go to stderr, not stdout
echo "Error: config file not found" >&2

# exit codes communicate success/failure to calling scripts and CI
exit 0   # success
exit 1   # generic failure
exit 2   # usage error (bad arguments) — a common convention
```

## Cheat sheet

| Pattern | Purpose |
|---------|---------|
| `while [[ $# -gt 0 ]]; do case "$1" in ...` | manual parsing for long-only flags |
| `getopts ":e:fvh" opt` | POSIX short-option parsing, `e:` = takes a value |
| `$OPTARG` | value attached to the current `getopts` flag |
| `$OPTIND` | index of the next unparsed argument |
| `shift $((OPTIND - 1))` | drop parsed flags, leaving positionals in `$@` |
| `--` | conventional "end of options" marker |
| `exit 2` | usage-error convention (vs. `1` for general failure) |

## Exercise

Write `greet.sh`, a CLI tool that accepts `-n NAME` (required), `-g GREETING`
(optional, default `"Hello"`), and `-h`/`--help`. Parse the flags with
`getopts`, print a proper `usage()` block on `-h` or on missing `-n`, and
have the tool print `"$GREETING, $NAME!"` on success. Test it with
`./greet.sh -n Alice`, `./greet.sh -n Bob -g Hi`, and `./greet.sh` (which
should fail with a usage error and exit code `2`).
