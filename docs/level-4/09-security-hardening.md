# 09 · Security Hardening for Production Scripts

Production scripts run unattended, often with elevated access to real
data. This module turns Level 3's security considerations into a concrete
production checklist: where secrets belong, how to run with the least
privilege a script actually needs, and a hardening audit you can run
against any script before it ships.

## Where secrets do NOT belong

```bash
# BAD — visible in shell history and any `ps aux` listing while it runs
mysql -u root -pSuperSecret123 mydb

# BAD — committed to version control forever, even if later removed
API_KEY="sk_live_abc123"
```

## Environment variables: better, but still leak further than you'd expect

```bash
export API_KEY="sk_live_abc123"
some_command      # API_KEY is visible to some_command and anything IT spawns,
                   # and readable via /proc/<pid>/environ by the same user or root
```

## Secret files: the standard production pattern

```bash
install -m 600 -o svcacct -g svcacct api_key.secret /run/secrets/api_key   # owner-only, no group/world access

read_secret() {
    local path="$1"
    [[ -f "$path" ]] || { echo "secret file '$path' not found" >&2; return 1; }
    local perms
    perms=$(stat -c '%a' "$path" 2>/dev/null || stat -f '%Lp' "$path")
    if [[ "$perms" != "600" && "$perms" != "400" ]]; then
        echo "refusing to read '$path' — permissions are $perms, expected 600/400" >&2
        return 1
    fi
    cat "$path"
}

api_key=$(read_secret /run/secrets/api_key)
```

## Secrets managers (Vault-style, tool-agnostic)

```bash
# most secrets managers boil down to: fetch at runtime, never persist longer than needed
api_key=$(vault kv get -field=api_key secret/myapp)      # HashiCorp Vault CLI

# or a cloud secrets manager:
api_key=$(aws secretsmanager get-secret-value --secret-id myapp/api_key --query SecretString --output text)

curl -H "Authorization: Bearer ${api_key}" https://api.example.com
unset api_key    # scrub it from this shell's variable table once you're done
```

## Keeping secrets out of logs and traces

```bash
set +x    # NEVER trace around secret-handling code — `-x` prints argument values verbatim

redact() {
    # strip anything that looks like a bearer token or "token=..." before logging a line
    echo "$1" | sed -E 's/(Bearer|token=)[A-Za-z0-9._-]+/\1[REDACTED]/g'
}
```

## Least-privilege execution: dedicated service accounts

```bash
sudo useradd -r -s /usr/sbin/nologin deploy_svc
sudo install -o deploy_svc -g deploy_svc -m 700 -d /opt/myapp

# run the script as that account instead of root, even during manual testing
sudo -u deploy_svc /opt/myapp/bin/deploy.sh
```

## Dropping privileges inside a script that must start as root

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ "$(id -u)" -eq 0 ]]; then
    # exec replaces this process — everything after this line runs as deploy_svc, not root
    exec setpriv --reuid=deploy_svc --regid=deploy_svc --init-groups "$0" "$@"
fi

echo "running as: $(whoami)"
```

## systemd sandboxing directives (defense in depth for services)

```ini
[Service]
User=deploy_svc
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/opt/myapp/data
CapabilityBoundingSet=
```

Each directive removes something the service doesn't need:
`NoNewPrivileges` blocks escalation via setuid binaries, `ProtectSystem=strict`
makes the whole filesystem read-only except explicit `ReadWritePaths`, and
an empty `CapabilityBoundingSet` strips every Linux capability — so even a
fully compromised process is boxed in.

## A hardening checklist script

```bash
#!/usr/bin/env bash
# harden_check.sh — audit a script for common production hardening gaps
set -euo pipefail

target="$1"
issues=0

check() {
    local description="$1"; shift
    if "$@"; then
        echo "  [ok]   $description"
    else
        echo "  [FAIL] $description"
        issues=$((issues + 1))
    fi
}

has_strict_mode()      { grep -q 'set -euo pipefail' "$target"; }
not_world_writable() {
    local perms
    perms=$(stat -c '%a' "$target" 2>/dev/null || stat -f '%Lp' "$target")
    [[ "$perms" != *2 ]]
}
no_hardcoded_secrets()  { ! grep -qiE '(password|api_key)=[^$]' "$target"; }
no_eval_calls()             { ! grep -q 'eval ' "$target"; }

echo "hardening check: $target"
check "has 'set -euo pipefail'"                has_strict_mode
check "not world-writable"                        not_world_writable
check "no hardcoded password=/api_key="  no_hardcoded_secrets
check "no bare eval calls"                        no_eval_calls

echo "$issues issue(s) found"
exit "$issues"
```

Notice the checklist script itself follows its own rules — no `eval`, no
hardcoded secrets, `set -euo pipefail` at the top — a hardening tool that
fails its own audit isn't trustworthy.

## Cheat sheet

| Practice | Why |
|----------|-----|
| never hardcode secrets in source | leaks via version control, `ps`, logs |
| secret files, mode 600, owned by the service account | standard least-exposure pattern |
| secrets managers (Vault, cloud KMS) | short-lived, centrally audited credentials |
| `set +x` around secret handling | prevents `-x` tracing from leaking values |
| dedicated non-root service account | limits blast radius of a compromised script |
| `setpriv`/`sudo -u` to drop privileges | run as root only as long as strictly necessary |
| systemd sandboxing directives | defense in depth even if the script is compromised |
| a hardening-check script in CI | catches regressions automatically, not just at review time |

## Exercise

Run (or adapt) a hardening-check script like the one above against Level
1's `backup.sh` and Level 3's `taskctl`. Fix any findings — in particular,
confirm neither script would leak a secret if invoked with `bash -x`, and
add a dedicated `backup_svc` system user that the backup script could run
as instead of your personal account.
