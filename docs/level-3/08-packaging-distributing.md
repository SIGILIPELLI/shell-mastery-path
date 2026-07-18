# 08 · Packaging & Distributing Scripts

A script that only runs on your machine, from the one directory you wrote it
in, isn't finished. This module covers making a CLI tool installable
system-wide, documenting it with a man page, and versioning it so users know
what they're running.

## Making a script installable

```bash
#!/usr/bin/env bash
# mytool — lives at bin/mytool in the project repo
set -euo pipefail
echo "mytool v1.0.0"
```

```bash
chmod +x bin/mytool

# put it somewhere already on $PATH so it can be run from anywhere
sudo cp bin/mytool /usr/local/bin/mytool

# or, without root, a per-user bin directory
mkdir -p "$HOME/.local/bin"
cp bin/mytool "$HOME/.local/bin/mytool"
# then make sure ~/.local/bin is on PATH (add to .bashrc/.zshrc if not):
export PATH="$HOME/.local/bin:$PATH"
```

```bash
which mytool        # confirm it resolves
mytool               # runs from any directory now
```

## A standard project layout

```text
mytool/
    bin/mytool              # the executable entry point
    lib/mytool/*.sh          # sourced helper functions (if the tool is big)
    man/mytool.1              # man page (see below)
    install.sh                  # copies bin/ + man/ into place
    CHANGELOG.md                 # version history
    VERSION                       # single source of truth for the version
```

```bash
# install.sh — a simple installer
#!/usr/bin/env bash
set -euo pipefail
PREFIX="${PREFIX:-/usr/local}"

install -Dm755 bin/mytool "$PREFIX/bin/mytool"
install -Dm644 man/mytool.1 "$PREFIX/share/man/man1/mytool.1"

echo "installed mytool to $PREFIX/bin/mytool"
echo "run 'man mytool' to view the manual"
```

`install(1)` sets permissions and creates parent directories in one step —
more reliable than a manual `mkdir -p && cp && chmod`.

## Writing a man page

Man pages use the `troff`/`groff` markup format. A minimal one:

```text
.TH MYTOOL 1 "2026-07-18" "mytool 1.0.0" "User Commands"
.SH NAME
mytool \- do a useful thing from the command line
.SH SYNOPSIS
.B mytool
[\fB\-v\fR] [\fB\-\-help\fR] \fIARGUMENT\fR
.SH DESCRIPTION
.B mytool
processes ARGUMENT and prints a result. Designed to be used in pipelines
and scripts.
.SH OPTIONS
.TP
.B \-v, \-\-verbose
Print extra diagnostic information to stderr.
.TP
.B \-\-help
Show usage and exit.
.SH EXIT STATUS
0 on success, 1 on invalid usage, 2 on a processing error.
.SH AUTHOR
Written by you.
```

```bash
man ./man/mytool.1              # preview it locally before installing
man mytool                       # after install.sh has copied it into place
```

## Self-documenting `--help`

Every real CLI tool should answer its own `--help` without needing the man
page installed:

```bash
usage() {
    cat <<'EOF'
Usage: mytool [OPTIONS] ARGUMENT

Process ARGUMENT and print a result.

Options:
  -v, --verbose   print extra diagnostic information
  -h, --help      show this help and exit

Exit status:
  0  success
  1  invalid usage
  2  processing error
EOF
}

case "${1:-}" in
    -h|--help) usage; exit 0 ;;
esac
```

## Versioning conventions

Semantic versioning (`MAJOR.MINOR.PATCH`) communicates the impact of a
release at a glance: bump `MAJOR` for breaking changes, `MINOR` for
backward-compatible features, `PATCH` for bug fixes.

```bash
# VERSION file, read by the script itself and by install.sh
echo "1.2.0" > VERSION
```

```bash
# embed it in the script so `mytool --version` works without extra files
VERSION="1.2.0"

case "${1:-}" in
    --version) echo "mytool $VERSION"; exit 0 ;;
esac
```

```markdown
# CHANGELOG.md
## [1.2.0] - 2026-07-18
### Added
- `--json` output flag.

## [1.1.0] - 2026-06-01
### Fixed
- Crash when ARGUMENT contained spaces.
```

## Distributing via a package manager

For wider distribution, a Homebrew formula lets macOS/Linuxbrew users
install with one command:

```ruby
# mytool.rb — a minimal Homebrew formula
class Mytool < Formula
  desc "Do a useful thing from the command line"
  homepage "https://github.com/you/mytool"
  url "https://github.com/you/mytool/archive/refs/tags/v1.2.0.tar.gz"
  sha256 "REPLACE_WITH_REAL_SHA256"
  license "MIT"

  def install
    bin.install "bin/mytool"
    man1.install "man/mytool.1"
  end

  test do
    system "#{bin}/mytool", "--version"
  end
end
```

```bash
brew install --build-from-source ./mytool.rb        # test locally
brew tap you/tap && brew install mytool               # after publishing the tap
```

## Cheat sheet

| Task | Command |
|------|---------|
| Install a script to `$PATH` | `install -Dm755 bin/tool /usr/local/bin/tool` |
| Install a man page | `install -Dm644 man/tool.1 /usr/local/share/man/man1/tool.1` |
| Preview a man page before installing | `man ./man/tool.1` |
| Check where a command resolves | `which tool` / `type tool` |
| Read `VERSION` in a script | `VERSION=$(cat VERSION)` |

## Exercise

Take the `backup.sh` script from Level 1's capstone project. Turn it into an
installable tool called `bkup`: give it a `--help` and `--version` flag, a
`VERSION` file, a one-page man page describing its usage and exit codes,
and an `install.sh` that installs both into `/usr/local`. Verify with
`man bkup` and `bkup --version` after installing.
