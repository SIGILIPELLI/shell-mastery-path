---
description: "Security Considerations — A shell script that processes untrusted input — a filename, a URL, data from an API — can be tricked into running arbitrary…"
---

# 07 · Security Considerations

A shell script that processes untrusted input — a filename, a URL, data
from an API — can be tricked into running arbitrary commands if it's not
written carefully. This module covers the most common shell security
pitfalls and how to avoid them.

## Command injection via unquoted/unsanitized input

```bash
# DANGEROUS — if $filename is "; rm -rf ~ ;" this executes that too
eval "cat $filename"

# SAFE — never eval untrusted input; just run the command directly
cat -- "$filename"
```

```bash
# DANGEROUS — a filename like "$(rm -rf ~)" gets command-substituted
echo "processing $filename"      # actually safe: echo doesn't re-evaluate its argument
result=$(some_cmd "$filename")     # SAFE — "$filename" is passed as a single literal argument
```

The danger isn't variable expansion itself — it's passing untrusted text
into `eval`, or into a context that re-parses it as shell code (like an
unquoted heredoc or a second round of `$( )` on already-substituted text).

## `eval` — avoid it, or sanitize hard before using it

```bash
# DANGEROUS pattern seen in real scripts
config_line='name="Ada"; rm -rf /'
eval "$config_line"      # executes BOTH statements!
```

```bash
# if you truly need dynamic variable names, prefer indirect expansion
# or `declare -n` (nameref) over eval
declare -n ref="my_var"    # ref now REFERS to my_var
my_var="hello"
echo "$ref"                  # hello
```

`eval` re-parses and executes its argument as shell code — if any part of
that string came from user input, a file, or a network response, that
input can inject arbitrary commands. Nearly every legitimate use of `eval`
has a safer alternative (arrays, namerefs, `printf -v`).

## The `--` argument separator

```bash
# a filename that starts with a dash gets misread as an OPTION
filename="--help"
rm "$filename"          # rm: unrecognized option '--help'   (fails, or worse, does something unintended)

rm -- "$filename"         # SAFE — "--" tells rm "everything after this is a filename, not an option"
```

Always use `--` before user-supplied filenames/arguments when calling
external commands (`rm`, `cp`, `grep`, ...) — it's the standard fix for
filenames that happen to look like flags.

## Quoting to prevent word-splitting and globbing

```bash
# DANGEROUS with a value like "* ; rm -rf ~"
files=$1
ls $files              # unquoted — glob-expands and word-splits, unpredictably

# SAFE
ls -- "$files"
```

## Path and filename validation

```bash
sanitize_path() {
    local path="$1"
    # reject anything containing .. (directory traversal) or null bytes
    if [[ "$path" == *".."* ]]; then
        echo "Error: path traversal ('..') not allowed" >&2
        return 1
    fi
    if [[ "$path" != /var/app/data/* ]]; then
        echo "Error: path must be within /var/app/data" >&2
        return 1
    fi
    echo "$path"
}

safe_path=$(sanitize_path "$user_supplied_path") || exit 1
```

Any script that builds a file path from user input (a web upload handler,
an API that takes a "filename" parameter) needs this kind of check — without
it, a value like `../../etc/passwd` can escape the intended directory
entirely.

## Least-privilege file permissions

```bash
umask 077                      # new files/dirs created after this are readable only by the owner

install -m 600 secrets.conf /etc/myapp/secrets.conf     # explicit, readable-owner-only permissions
chmod 700 /opt/myapp/scripts/deploy.sh                     # executable only by the owner
```

```bash
# check a file isn't world-writable before trusting/sourcing it
if [[ -w "$config_file" ]] && [[ "$(stat -f '%A' "$config_file" 2>/dev/null || stat -c '%a' "$config_file")" == *"2" ]]; then
    echo "Warning: config file is group/world-writable — refusing to source it" >&2
    exit 1
fi
```

## Handling secrets safely

```bash
# BAD — secret ends up in shell history and `ps aux` output as a plain argument
curl -H "Authorization: Bearer sk_live_abc123" https://api.example.com

# BETTER — read from an environment variable, set OUTSIDE the script
curl -H "Authorization: Bearer ${API_TOKEN}" https://api.example.com

# BETTER STILL — read from a restricted-permission file, never logged
api_token=$(cat /run/secrets/api_token)
curl -H "Authorization: Bearer ${api_token}" https://api.example.com
```

```bash
set +x    # make ABSOLUTELY sure tracing is off before handling secrets —
          # `set -x` would print the secret value to the trace output
```

Never hardcode secrets in a script's source, and be careful that `set -x`
tracing, error messages, or logs don't accidentally leak a token or
password — Level 4's "Security Hardening" module goes further into secrets
management for production systems.

## How It Actually Works

Command injection is possible because bash's parser doesn't distinguish
"data" from "code" at the string level — if untrusted input reaches a
context where the parser performs word splitting, globbing, or (worse)
where it's interpolated into a string later passed to `eval` or a `-c`
invocation of another shell, the parser will happily treat metacharacters
like `;`, `|`, `` ` ``, or `$(...)` in that input as syntax to execute, not
literal text. Quoting a variable (`"$input"`) defeats this specifically
because double quotes suppress the word-splitting and globbing expansion
passes — but they do *not* suppress command or parameter substitution, so
`"$( $input )"`-style constructs can still be dangerous depending on
exactly which expansion touches the untrusted string.

Handling secrets safely comes down to which storage layer a value passes
through: an environment variable is visible to every process that can read
`/proc/<pid>/environ` for its own children (and, depending on OS
permissions, sometimes to other users), while a value only ever read from a
file with restrictive permissions (`chmod 600`) and piped directly into a
program's stdin never touches the process table or environment block at
all — this is why `--password-stdin`-style flags exist on many CLI tools,
specifically to avoid a secret ever appearing in `ps aux` output (which
shows a process's `argv`, readable by other users on many systems) or in
the environment block.

`set -x` tracing and shell history files are both security-relevant for the
same reason: both persistently record the *expanded* command line,
meaning a secret interpolated into a command's arguments can end up
readable in a debug log or `~/.bash_history` even if it was never written
to a file on purpose.


## Cheat sheet

| Risk | Mitigation |
|------|------------|
| `eval` on untrusted input | avoid `eval`; use arrays/namerefs instead |
| Filenames starting with `-` | always pass `--` before user-supplied args |
| Unquoted variables | quote everything; prevents word-splitting/globbing |
| Path traversal (`../`) | validate/reject `..` and enforce an allowed base path |
| World-writable config files | `umask`, `chmod`, check permissions before sourcing |
| Secrets in arguments/logs | pass via env vars or restricted files, never as CLI text; keep `set -x` off around them |

## Exercise

Write `safe_delete.sh` that takes a filename argument and only deletes it
if the resolved absolute path (use `realpath`) is inside an allowed
directory (e.g. `/tmp/scratch`), rejects any path containing `..`, uses
`--` before the filename in the actual `rm` call, and refuses to run at all
if invoked with `eval` anywhere in its own logic. Test it against a normal
file, a `../../etc/passwd`-style path, and a `--`-prefixed filename.
