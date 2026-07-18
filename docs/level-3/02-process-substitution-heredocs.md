# 02 · Process Substitution & Here-Docs

Two techniques that make scripts dramatically more compact: **process
substitution** (`<(...)`/`>(...)`), which lets a command's output act like a
file, and **here-documents** (`<<EOF`), which embed multi-line text directly
in a script.

## Process substitution: `<(...)` — treat command output as a file

```bash
diff <(sort file1.txt) <(sort file2.txt)
# compares the SORTED versions of both files without ever writing a temp file
```

```bash
diff <(curl -s https://api.example.com/v1/config) <(curl -s https://api.example.com/v2/config)
# diff two live API responses directly
```

`<(command)` runs `command`, and substitutes a path (actually a named pipe
or `/dev/fd/N` entry) that other tools can open and read from as if it were
a regular file — many commands like `diff` and `comm` need two real file
arguments, and process substitution lets you feed them command output
instead.

## Process substitution with `while read`

```bash
# feeding a command's output into a while loop WITHOUT a subshell
count=0
while IFS= read -r line; do
    ((count++))
done < <(grep "ERROR" app.log)

echo "found $count error lines"
```

Compare this to `grep "ERROR" app.log | while read -r line; do ((count++)); done` —
piping into a `while` loop runs the loop in a **subshell**, so `count`
would reset to 0 outside the loop. Using `< <(...)` keeps the loop in the
current shell, so variables it sets are still visible afterward.

## Process substitution: `>(...)` — treat a command as a writable file

```bash
# split a stream to two different processing pipelines at once
some_command | tee >(grep "ERROR" > errors.txt) >(grep "WARN" > warnings.txt) > all_output.txt
```

`>(command)` gives you a "file" that, when written to, feeds its content as
stdin to `command` — combined with `tee`, this fans a single stream out to
multiple simultaneous consumers.

## Here-documents: multi-line input, embedded in the script

```bash
cat <<EOF
This is line one.
Today's date is $(date +%F).
This is line three.
EOF
```

```bash
# feeding a here-doc as input to a command
mysql -u root <<EOF
USE mydb;
SELECT COUNT(*) FROM users;
EOF
```

Everything between `<<EOF` and the closing `EOF` is passed as stdin (or, in
the `cat` example, as arguments to print) — variables and command
substitution **are** expanded inside a here-doc by default.

## Suppressing expansion: quoted delimiter

```bash
cat <<'EOF'
This $variable will NOT be expanded.
Neither will this: $(command substitution)
EOF
```

Quoting the delimiter (`<<'EOF'` instead of `<<EOF`) makes bash treat the
whole block as a literal string — essential when writing a template file
that itself contains `$` characters meant literally (like a script that
generates *another* script).

## `<<-` — stripping leading tabs for clean indentation

```bash
if true; then
    cat <<-EOF
	This line can be indented with TABS to match the code around it,
	and the leading tabs are stripped from the output.
	EOF
fi
```

`<<-` strips **leading tab characters** (not spaces) from each line and from
the closing delimiter — useful for keeping a here-doc visually indented
inside an `if`/function without that indentation ending up in the output.

## Writing a file with a here-doc

```bash
cat > deploy_config.yml <<EOF
app_name: myservice
environment: ${DEPLOY_ENV:-production}
replicas: 3
generated_at: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF

echo "wrote deploy_config.yml"
```

This is a common pattern for generating config files, Dockerfiles, or
templated scripts from a shell script, filling in values from variables and
command substitution.

## Here-strings — a single-line shortcut

```bash
grep "root" <<< "$(cat /etc/passwd)"     # from Level 1 — a single value as stdin
```

`<<<` (a "here-string") is the single-line cousin of a here-document — use
it when you just need to feed one existing string/variable as stdin,
reaching for `<<EOF` only when you need genuinely multi-line literal text.

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `<(cmd)` | run `cmd`, expose its stdout as a readable "file" |
| `>(cmd)` | expose a writable "file" whose content becomes `cmd`'s stdin |
| `cmd <<EOF ... EOF` | here-document: multi-line stdin, `$vars` expanded |
| `cmd <<'EOF' ... EOF` | here-document with NO expansion (literal) |
| `cmd <<-EOF ... EOF` | here-document, strips leading tabs |
| `cmd <<< "$string"` | here-string: a single value as stdin |

## Exercise

Write a script `compare_dirs.sh` that takes two directory paths and uses
`diff` with process substitution to compare the *sorted* output of
`find dir -type f -printf '%f\n'` for each directory (i.e., compare which
filenames exist in one but not the other) without writing any temp files.
Then write a second script that generates a `docker-compose.yml` file using
a here-doc, filling in the service name and port from script arguments.
