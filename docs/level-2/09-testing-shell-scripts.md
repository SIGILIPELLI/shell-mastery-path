# 09 · Testing Shell Scripts

Shell scripts accumulate logic just like any other code, and just like any
other code they benefit from automated tests. **Bats** (Bash Automated
Testing System) is the standard framework for this — it lets you write
tests in a syntax that looks like plain bash with a thin TAP-producing
layer on top.

## Installing bats

```bash
# macOS
brew install bats-core

# verify
bats --version
```

## Anatomy of a bats test file

```bash
# greet.sh — the script under test
greet() {
    local name="${1:-World}"
    echo "Hello, $name!"
}
```

```bash
# greet.bats
#!/usr/bin/env bats

setup() {
    load "greet.sh"    # source the script under test before each test
}

@test "greets a given name" {
    run greet "Ada"
    [ "$status" -eq 0 ]
    [ "$output" = "Hello, Ada!" ]
}

@test "greets World by default" {
    run greet
    [ "$output" = "Hello, World!" ]
}
```

```bash
bats greet.bats
```

```text
 ✓ greets a given name
 ✓ greets World by default

2 tests, 0 failures
```

Each `@test "description" { ... }` block is one test case; `run` executes a
command, capturing its exit code into `$status` and its combined output
into `$output` (and each line into the `$lines` array) for assertions.

## Testing exit codes and output together

```bash
# validate.sh
validate_age() {
    local age="$1"
    if [[ ! "$age" =~ ^[0-9]+$ ]]; then
        echo "Error: age must be a number" >&2
        return 1
    fi
    if (( age < 0 || age > 150 )); then
        echo "Error: age out of range" >&2
        return 1
    fi
    echo "valid age: $age"
}
```

```bash
# validate.bats
setup() { load "validate.sh"; }

@test "accepts a valid age" {
    run validate_age 30
    [ "$status" -eq 0 ]
    [ "$output" = "valid age: 30" ]
}

@test "rejects non-numeric input" {
    run validate_age "abc"
    [ "$status" -eq 1 ]
    [[ "$output" == *"must be a number"* ]]
}

@test "rejects an out-of-range age" {
    run validate_age 200
    [ "$status" -eq 1 ]
    [[ "$output" == *"out of range"* ]]
}
```

## Setup, teardown, and temp directories

```bash
setup() {
    TEST_DIR=$(mktemp -d)
    cd "$TEST_DIR" || exit 1
}

teardown() {
    cd - > /dev/null || true
    rm -rf "$TEST_DIR"
}

@test "creates the expected output file" {
    run touch "output.txt"
    [ -f "output.txt" ]
}
```

`setup`/`teardown` run before/after **every** `@test` in the file — using a
fresh `mktemp -d` temp directory per test avoids one test's leftover files
affecting another.

## Testing a real script end-to-end (not just its functions)

```bash
# backup.bats — testing backup.sh from Level 1's capstone
setup() {
    TEST_DIR=$(mktemp -d)
    mkdir -p "$TEST_DIR/source_data"
    echo "sample" > "$TEST_DIR/source_data/file.txt"
}

teardown() {
    rm -rf "$TEST_DIR"
}

@test "backup.sh creates a tar.gz archive" {
    run ./backup.sh "$TEST_DIR/source_data"
    [ "$status" -eq 0 ]
    run bash -c "ls $TEST_DIR/../backups/*.tar.gz 2>/dev/null | wc -l"
}

@test "backup.sh fails on a nonexistent source directory" {
    run ./backup.sh "/nonexistent/path"
    [ "$status" -ne 0 ]
    [[ "$output" == *"does not exist"* ]]
}
```

Testing the whole script via `run ./backup.sh ...` (rather than just its
internal functions) is called a "black box" test — it verifies the actual
contract users depend on, independent of implementation details.

## Bats helper libraries

```bash
# common add-ons (installed alongside bats-core):
#   bats-support  — better failure messages
#   bats-assert   — assert_success, assert_output, assert_line, etc.

load 'bats-support/load'
load 'bats-assert/load'

@test "using bats-assert" {
    run greet "Ada"
    assert_success
    assert_output "Hello, Ada!"
}
```

## Cheat sheet

| Construct | Purpose |
|-----------|---------|
| `@test "name" { ... }` | one test case |
| `run cmd` | execute `cmd`, capturing `$status`/`$output`/`$lines` |
| `setup()` / `teardown()` | run before/after every test in the file |
| `load "file.sh"` | source a script so its functions are testable |
| `[ "$status" -eq 0 ]` | assert a successful exit code |
| `[[ "$output" == *"text"* ]]` | assert output contains a substring |

## Exercise

Write `slugify.bats` (testing the `slugify.sh` function from Level 2's
"String Manipulation" module) with at least four test cases: a normal
sentence, a string with punctuation, an empty string, and a string that's
already a valid slug. Run `bats slugify.bats` and make sure every case
passes.
