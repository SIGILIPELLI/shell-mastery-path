# 01 · Setup & First Script

## Which shell?

Almost every Linux distro and macOS ships with **Bash**. Check what you have:

```bash
echo $SHELL          # /bin/bash, /bin/zsh, etc. — your login shell
bash --version        # confirm bash itself is installed
```

macOS has used `zsh` as the default login shell since Catalina, but `bash` is
still installed and this whole site targets Bash specifically (noting POSIX
`sh` differences where they matter, since some systems' `/bin/sh` is a much
more limited shell like `dash`).

## The terminal, briefly

A few things you'll use constantly:

```bash
pwd                # print working directory — where am I?
ls                  # list files in the current directory
ls -la              # list all files (incl. hidden), long format
cd some/path        # change directory
cd ..               # go up one level
cd -                # jump back to the previous directory
clear               # clear the screen
```

## Your first script

Create a file called `hello.sh`:

```bash
#!/usr/bin/env bash
# hello.sh — prints a greeting

echo "Hello, world!"
```

### The shebang line

`#!/usr/bin/env bash` is the **shebang**. It must be the very first line of
the file (no blank line before it) and tells the OS which interpreter to use
when the script is executed directly.

| Shebang | Meaning |
|---------|---------|
| `#!/bin/bash` | run with bash at this exact path |
| `#!/usr/bin/env bash` | find `bash` via `PATH` — more portable across systems |
| `#!/bin/sh` | run with the system's POSIX shell (may be `dash`, not `bash`) |

`#!/usr/bin/env bash` is the recommended default, since `bash` doesn't always
live at `/bin/bash` (some systems, like Nix-based ones, put it elsewhere).

### Making it executable

```bash
chmod +x hello.sh    # add the executable permission bit
ls -l hello.sh
# -rwxr-xr-x  1 you  staff  45 Jul 18 10:00 hello.sh
#  ^^^ the three x's mean owner/group/other can execute this file
```

### Running it

```bash
./hello.sh            # run it directly (needs +x and the shebang)
# Hello, world!

bash hello.sh         # or explicitly hand it to bash — works even without +x
sh hello.sh           # run under the system's sh instead
```

`./hello.sh` looks for `hello.sh` in the current directory (`.`) — plain
`hello.sh` won't work unless the current directory is on your `PATH`, which it
normally isn't (for good reason — security).

## Comments

```bash
# A comment runs to the end of the line.
echo "this runs"   # comments can also follow real code
```

There is no multi-line comment syntax in Bash; every commented line needs its
own `#`.

## A slightly bigger first script

```bash
#!/usr/bin/env bash
# greet.sh — greets whoever is running the script

echo "What is your name?"
read -r name          # read a line of input into the variable `name`
echo "Hello, $name! Welcome to shell scripting."
```

```bash
chmod +x greet.sh
./greet.sh
# What is your name?
# Ada
# Hello, Ada! Welcome to shell scripting.
```

`read -r` reads a line from standard input into a variable; `-r` disables
backslash escaping so backslashes in the input are kept literally — almost
always what you want.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `pwd` | print current directory |
| `ls -la` | list all files, long format |
| `cd path` | change directory |
| `chmod +x file` | make a file executable |
| `./script.sh` | run a script in the current directory |
| `bash script.sh` | run a script with bash explicitly |
| `read -r var` | read a line of input into `var` |

## Exercise

Write `intro.sh` that prints your name, today's date (use the `date` command),
and the current working directory (`pwd`), each on its own line. Make it
executable and run it with `./intro.sh`.
