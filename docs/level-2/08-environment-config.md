---
description: "Environment & Configuration Management — Every shell session has an environment — a set of variables inherited by every process it starts — plus a chain…"
---

# 08 · Environment & Configuration Management

Every shell session has an **environment** — a set of variables inherited by
every process it starts — plus a chain of startup files that build it up.
Understanding this is essential for scripts that behave differently
depending on where or how they're run.

## Shell variables vs environment variables

```bash
my_var="hello"          # shell variable — visible only in THIS shell
echo "$my_var"            # hello

export my_var              # promote it to an environment variable —
                              # now child processes inherit it too

bash -c 'echo "$my_var"'    # hello   (only works AFTER export)
```

```bash
env                 # list all environment variables
printenv PATH         # print one specific one
echo "$PATH"           # same thing, shell-native
```

An unexported shell variable exists only in the current shell; `export`
copies it into the environment block that every child process (subshells,
scripts you call, `bash -c`) receives automatically.

## `.bashrc` vs `.bash_profile`/`.profile`

| File | Runs for |
|------|----------|
| `~/.bashrc` | every new **interactive**, non-login shell (e.g. opening a new terminal tab) |
| `~/.bash_profile` / `~/.profile` | **login** shells (e.g. SSH-ing in, or a fresh terminal on macOS) |
| `/etc/environment` | system-wide, for ALL users, before per-user files |
| `~/.bash_logout` | runs when a login shell exits |

```bash
# typical ~/.bash_profile pattern — load .bashrc too, so login shells
# get the same setup as interactive ones
if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
fi
```

This split exists for historical reasons (expensive one-time setup in the
login file, cheap interactive setup like aliases in `.bashrc`) — in
practice, many people just source one from the other, as above.

## Common `.bashrc` contents

```bash
# ~/.bashrc

# custom prompt
PS1='\u@\h:\w\$ '

# aliases
alias ll='ls -lah'
alias gs='git status'

# add a personal scripts directory to PATH
export PATH="$HOME/bin:$PATH"

# environment variables used by scripts/tools
export EDITOR="vim"
export MY_APP_ENV="development"
```

## Sourcing vs executing a script

```bash
./config.sh        # EXECUTES in a new subshell — variables it sets vanish when it exits
source config.sh     # SOURCES (aka `. config.sh`) — runs in the CURRENT shell,
                        # so any variables it sets persist afterward
```

```bash
# config.sh
DB_HOST="localhost"
DB_PORT="5432"
```

```bash
source config.sh
echo "$DB_HOST:$DB_PORT"    # localhost:5432 — only works because we SOURCED it
```

`source` (or the `.` shorthand) is the standard way to load a shared
configuration file into a script — using `./config.sh` instead would run it
in a throwaway subshell and none of its variables would reach your script.

## A config-file pattern for scripts

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="${1:-./myapp.conf}"

if [[ ! -f "$CONFIG_FILE" ]]; then
    echo "Error: config file '$CONFIG_FILE' not found" >&2
    exit 1
fi

# shellcheck source=/dev/null
source "$CONFIG_FILE"

echo "connecting to ${DB_HOST}:${DB_PORT} as ${DB_USER}"
```

```bash
# myapp.conf
DB_HOST="db.internal"
DB_PORT="5432"
DB_USER="app_service"
```

This lets non-developers (or different environments — dev/staging/prod)
change behavior by editing a plain config file, without touching the
script's logic at all.

## Checking and requiring environment variables

```bash
require_env() {
    local var_name="$1"
    if [[ -z "${!var_name:-}" ]]; then
        echo "Error: required environment variable '$var_name' is not set" >&2
        exit 1
    fi
}

require_env "API_KEY"
require_env "DEPLOY_ENV"

echo "starting with DEPLOY_ENV=$DEPLOY_ENV"
```

`${!var_name}` is **indirect expansion** — it looks up the variable whose
*name* is stored in `var_name`, which is how you can validate an arbitrary
list of required env vars in a loop.

## How It Actually Works

The environment is a flat block of `NAME=value\0` strings that the kernel
copies into a new process's address space as part of `execve(2)` — every
`export`ed variable in your shell gets serialized into this block at the
moment you launch a child, and the child receives its own private copy, not
a live link back to the parent. Setting `export FOO=bar` after a child has
already started never affects that already-running child; it only affects
processes forked *afterward*.

Startup files (`.bashrc`, `.bash_profile`, `.profile`) are read according to
rules baked into bash itself based on how it was invoked: a *login* shell
reads `.bash_profile` (falling back to `.profile`), while a plain
*interactive, non-login* shell reads `.bashrc` — bash decides which mode it's
in by checking how `argv[0]` was set and whether `-l`/`--login` was passed,
which is why the exact same terminal emulator can source different files
depending on whether it launches bash as a login shell or not.

`.bashrc` deliberately does **not** run for non-interactive shells (like
scripts, or the shell cron spawns), because bash checks whether its stdin
is attached to a terminal (an interactive check) before sourcing it — this
is precisely why exporting a variable only in `.bashrc` "disappears" for
cron jobs or scripts, and why config meant for scripts belongs in something
explicitly sourced or in `/etc/environment`-style files instead.


## Cheat sheet

| Concept | Detail |
|---------|--------|
| shell variable | local to the current shell only |
| `export VAR` | promotes it into the environment, inherited by child processes |
| `~/.bashrc` | loaded for interactive non-login shells |
| `~/.bash_profile` / `~/.profile` | loaded for login shells |
| `source file.sh` (or `. file.sh`) | run in the current shell — variables persist |
| `./file.sh` | run in a subshell — variables do NOT persist |
| `${!name}` | indirect expansion — look up variable by a name stored in another variable |

## Exercise

Write `myapp.conf` with a few config variables (`APP_NAME`, `APP_PORT`,
`APP_ENV`) and a script `start.sh` that sources it, uses `require_env` to
validate that all three are set and non-empty, and prints a startup banner
using their values. Test what happens when a variable is missing from the
config file.
