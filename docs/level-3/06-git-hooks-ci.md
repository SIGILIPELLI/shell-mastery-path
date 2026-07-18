# 06 · Integrating with Git Hooks & CI Scripts

Shell scripts are the glue behind most git workflows and CI pipelines. This
module covers writing **git hooks** (scripts that run automatically at
points in the git lifecycle) and shell steps for CI systems like GitHub
Actions.

## Git hooks: where they live

```bash
ls .git/hooks/
# lots of *.sample files — hooks are disabled until you remove the .sample suffix
```

| Hook | Runs when |
|------|-----------|
| `pre-commit` | before a commit is created (can abort it) |
| `commit-msg` | after the commit message is written (can validate/reject it) |
| `pre-push` | before `git push` sends anything to the remote |
| `post-checkout` | after `git checkout`/`git switch` |
| `post-merge` | after a successful `git merge` (including `git pull`) |

## A pre-commit hook: block commits with debug statements

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
set -euo pipefail

staged_files=$(git diff --cached --name-only --diff-filter=ACM -- '*.sh')

if [[ -z "$staged_files" ]]; then
    exit 0     # no shell files staged — nothing to check
fi

found_issue=0
while IFS= read -r file; do
    if grep -nE "^\s*(echo\s+\"?DEBUG|set -x\b)" "$file" > /dev/null; then
        echo "pre-commit: '$file' contains leftover debug output — remove before committing" >&2
        found_issue=1
    fi
done <<< "$staged_files"

exit "$found_issue"
```

```bash
chmod +x .git/hooks/pre-commit    # hooks must be executable to run
```

A non-zero exit from `pre-commit` **aborts the commit** — this pattern
catches an entire class of "oops, forgot to remove debug code" mistakes
before they ever reach the repository.

## A pre-commit hook: run shellcheck on staged scripts

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
set -euo pipefail

staged_sh_files=$(git diff --cached --name-only --diff-filter=ACM -- '*.sh')
[[ -z "$staged_sh_files" ]] && exit 0

failed=0
while IFS= read -r file; do
    if ! shellcheck "$file"; then
        failed=1
    fi
done <<< "$staged_sh_files"

if [[ "$failed" -eq 1 ]]; then
    echo "pre-commit: shellcheck failed — fix the warnings above, or 'git commit --no-verify' to bypass" >&2
    exit 1
fi
```

## A commit-msg hook: enforce a message format

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg
set -euo pipefail

commit_msg_file="$1"
commit_msg=$(head -1 "$commit_msg_file")

if [[ ! "$commit_msg" =~ ^(feat|fix|docs|refactor|test|chore)(\(.+\))?:\ .+ ]]; then
    echo "commit-msg: message must follow 'type: description' (e.g. 'fix: correct off-by-one')" >&2
    echo "Got: $commit_msg" >&2
    exit 1
fi
```

`commit-msg` hooks receive the path to a temp file holding the message as
`$1` — this is how you validate conventional-commit style formatting
automatically.

## Sharing hooks across a team

Hooks in `.git/hooks/` are **not** version-controlled by default. The
standard fix is `core.hooksPath`:

```bash
mkdir -p .githooks
# move your hook scripts into .githooks/, commit them normally

git config core.hooksPath .githooks
```

Every contributor who runs `git config core.hooksPath .githooks` (often
via a one-time `make setup` or `./scripts/bootstrap.sh`) then shares the
exact same hooks, tracked in version control like any other code.

## Shell steps in CI (GitHub Actions example)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install shellcheck
        run: sudo apt-get update && sudo apt-get install -y shellcheck

      - name: Lint all shell scripts
        run: |
          set -euo pipefail
          find . -name "*.sh" -print0 | xargs -0 shellcheck

      - name: Install bats
        run: sudo apt-get install -y bats

      - name: Run tests
        run: bats tests/*.bats
```

CI runners execute each `run:` block as a shell script — the same
`set -euo pipefail` discipline from Level 2 applies directly, and a failing
step (non-zero exit) fails the whole pipeline, exactly like a local script.

## A pre-push hook: prevent pushing directly to main

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push
set -euo pipefail

protected_branch="main"
current_branch=$(git rev-parse --abbrev-ref HEAD)

if [[ "$current_branch" == "$protected_branch" ]]; then
    echo "pre-push: direct pushes to '$protected_branch' are blocked — open a PR instead" >&2
    exit 1
fi
```

## Cheat sheet

| Hook/tool | Purpose |
|-----------|---------|
| `.git/hooks/pre-commit` | validate/lint before a commit is created |
| `.git/hooks/commit-msg` | validate the commit message text |
| `.git/hooks/pre-push` | validate before pushing to a remote |
| `core.hooksPath` | point git at a version-controlled hooks directory |
| `git diff --cached --name-only` | list staged files (for a pre-commit hook) |
| CI `run:` step | just a shell script — same conventions apply |

## Exercise

Set up `.githooks/pre-commit` in a test repo that runs `shellcheck` against
every staged `.sh` file and blocks the commit if any fail, wire it up with
`git config core.hooksPath .githooks`, and confirm it actually blocks a
commit containing a deliberately unquoted variable. Then write a matching
GitHub Actions workflow that runs the same `shellcheck` check in CI.
