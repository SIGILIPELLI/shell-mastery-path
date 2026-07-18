# 07 · System Administration Scripting

Scripts that manage the system itself — creating users, wiring up
services, controlling systemd units — affect the whole machine, not just
one process, so they deserve extra care. This module covers user/group
management, writing service control scripts, and systemd basics for
running your own scripts as managed, supervised services.

## Creating and managing users

```bash
sudo useradd -m -s /bin/bash deploy            # -m creates a home dir, -s sets the login shell
sudo useradd -r -s /usr/sbin/nologin svcacct     # -r = system account, nologin = no interactive shell

sudo usermod -aG docker deploy                     # -aG appends deploy to the docker group (never plain -G!)
sudo userdel -r olduser                                # -r also removes the home directory
```

`-aG` (append) is important: plain `-G` *replaces* a user's entire group
list, silently dropping every group they weren't just added to.

## Inspecting users and groups

```bash
id deploy                       # uid, gid, and group memberships
getent passwd deploy              # look up a user regardless of backend (files, LDAP, etc.)
getent group docker                 # list members of a group
groups deploy                          # groups deploy belongs to
```

## Scripting user creation idempotently

```bash
ensure_user() {
    local username="$1"
    if id "$username" &>/dev/null; then
        echo "user '$username' already exists — skipping"
    else
        useradd -m -s /bin/bash "$username"
        echo "created user '$username'"
    fi
}

ensure_user "deploy"
ensure_user "deploy"    # safe to run again — no error, no duplicate
```

Idempotent provisioning scripts (safe to re-run any number of times) are
the standard for configuration-management style automation — the same
principle applies to groups, directories, and package installs.

## Password and account state

```bash
sudo passwd -l deploy           # lock the account (disable password login)
sudo passwd -u deploy            # unlock it
sudo chage -l deploy               # show password expiry policy
sudo chage -M 90 deploy              # force password rotation every 90 days
```

## Writing a service control wrapper script

```bash
#!/usr/bin/env bash
# myappctl — start/stop/status wrapper around a long-running process
set -euo pipefail

PID_FILE="/var/run/myapp.pid"
APP_BIN="/opt/myapp/bin/myapp"

start() {
    if [[ -f "$PID_FILE" ]] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
        echo "already running (pid $(cat "$PID_FILE"))"
        exit 0
    fi
    nohup "$APP_BIN" > /var/log/myapp.log 2>&1 &
    echo $! > "$PID_FILE"
    echo "started (pid $!)"
}

stop() {
    if [[ -f "$PID_FILE" ]] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
        kill "$(cat "$PID_FILE")"
        rm -f "$PID_FILE"
        echo "stopped"
    else
        echo "not running"
    fi
}

status() {
    if [[ -f "$PID_FILE" ]] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
        echo "running (pid $(cat "$PID_FILE"))"
    else
        echo "stopped"
    fi
}

case "${1:-}" in
    start) start ;;
    stop) stop ;;
    status) status ;;
    restart) stop; start ;;
    *) echo "Usage: $0 {start|stop|status|restart}" >&2; exit 1 ;;
esac
```

This pid-file pattern is exactly what systemd was built to replace — no
manual `kill -0` checks, no stale pid files after a crash.

## systemd basics: unit files

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
ExecStart=/opt/myapp/bin/myapp
Restart=on-failure
User=myapp
Group=myapp

[Install]
WantedBy=multi-user.target
```

## Controlling a systemd service

```bash
sudo systemctl daemon-reload           # re-read unit files after editing one
sudo systemctl start myapp
sudo systemctl enable myapp              # start automatically on boot
sudo systemctl status myapp
sudo systemctl restart myapp
sudo systemctl stop myapp
journalctl -u myapp -f                     # follow this service's logs live
```

## systemd timers: a cron alternative

```ini
# /etc/systemd/system/myapp-backup.timer
[Unit]
Description=Run myapp backup daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now myapp-backup.timer
systemctl list-timers                        # see all scheduled timers and their next run
```

A `.timer` unit pairs with a matching `myapp-backup.service`
(`Type=oneshot`) — the timer only decides *when* to start it, while
systemd's own logging, restart, and dependency handling give you more than
a raw crontab line does.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `useradd -m -s /bin/bash user` | create a user with a home dir and shell |
| `usermod -aG group user` | add a user to a group (append, don't replace) |
| `userdel -r user` | delete a user and their home dir |
| `id user` / `getent passwd user` | inspect a user's identity |
| `systemctl start/stop/status svc` | control a systemd service |
| `systemctl enable svc` | start automatically on boot |
| `journalctl -u svc -f` | follow a service's logs |
| `*.timer` unit | systemd's cron-like scheduling primitive |

## Exercise

Write `provision_user.sh` that takes a username and a group name as
arguments, creates the user idempotently (skipping if it already exists),
adds them to the group (creating the group first if it doesn't exist), and
locks the password so only key-based or sudo access works. Then write a
systemd unit file for a script of your choice and enable it to start on
boot.
