# Scheduling Beyond Cron

## The Event Calendar Analogy

A standard wall calendar (cron) is great for recurring appointments — "team meeting every Monday at 9 AM." But life has other scheduling needs: "remind me in 45 minutes to check the oven," "watch this status page and notify me of any change," or "make sure this appointment happens sometime this week, even if my laptop was closed on the scheduled day."

Linux provides several tools beyond cron, each suited to different scheduling needs:
- `at` — run a command once at a specific future time
- `watch` — repeat a command continuously and watch for changes
- `systemd timers` — cron's modern replacement on systemd-based systems
- `anacron` — cron for laptops and desktops that aren't always on

```
  Scheduling tools comparison:

  cron        Recurring, time-based, assumes always-on server
  at          One-shot: "run this once in 2 hours"
  watch       Continuous: "show me this every 2 seconds"
  systemd     Modern cron: better logging, dependencies, boot delay
  anacron     Catch-up cron: "daily" means "daily even if machine was off"
```

---

## `at`: Run Once at a Future Time

The `at` command schedules a command to run once at a specific time. Unlike cron, it is not recurring.

```bash
# Install at if needed
apt-get install at   # Ubuntu/Debian
yum install at       # CentOS/RHEL

# Enable the atd daemon
systemctl enable --now atd
```

### Scheduling jobs with `at`

```bash
# Run in 5 minutes
at now + 5 minutes
# (prompt appears — type commands, then Ctrl+D to submit)
at> /opt/scripts/deploy.sh
at>

# Run at a specific time today
at 14:30
at 2:30pm

# Run tomorrow
at 14:30 tomorrow
at noon tomorrow

# Run on a specific date
at 14:30 Mar 20 2026
at 2pm March 20

# Pass a command directly
echo "/opt/scripts/run_migration.sh" | at midnight
echo "/opt/scripts/notify.sh" | at now + 2 hours
```

### Managing `at` jobs

```bash
# List pending at jobs
atq

# View what a specific job will do
at -c <job_number>

# Remove a pending job
atrm <job_number>

# Example output of atq:
# 5  Sun Mar 15 14:30:00 2026 a ubuntu
# ^  ^                        ^ ^
# |  |                        | +-- user
# |  |                        +-- queue (a=at, b=batch)
# |  +-- scheduled time
# +-- job number
```

### `at` in scripts (fire-and-forget)

```bash
#!/usr/bin/env bash
# maintenance_window.sh — start maintenance, schedule the end

echo "Starting maintenance mode..."
touch /var/www/html/maintenance.flag

# Schedule automatic exit from maintenance in 2 hours
echo "rm /var/www/html/maintenance.flag && systemctl start nginx" | at now + 2 hours

echo "Maintenance window open. Will auto-close in 2 hours."
echo "Job ID: $(atq | tail -1 | cut -f1)"
```

---

## `watch`: Continuous Command Monitoring

`watch` runs a command repeatedly (default: every 2 seconds) and shows the latest output, refreshing the terminal like a dashboard:

```bash
# Basic: watch a command every 2 seconds
watch ls -la /var/log/myapp/

# Custom interval: every 5 seconds
watch -n 5 df -h

# Highlight changes (highlight differences from last run)
watch -d free -h

# Exit when the output changes
watch -g ls /tmp/deploy_complete.flag

# No title bar (cleaner for screenshots)
watch -t date
```

### Practical monitoring use cases

```bash
# Watch disk usage
watch -n 10 df -h

# Watch memory
watch -n 2 free -h

# Watch Docker containers
watch -n 3 docker ps

# Watch a deployment log file grow
watch -n 1 "tail -20 /var/log/deploy.log"

# Watch nginx connection count
watch -n 5 "ss -s | grep estab"

# Watch AWS EC2 instances
watch -n 30 "aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' --output table"
```

### `watch` vs a loop

```bash
# watch: simple, visual, interactive
watch -n 5 "curl -s http://localhost/health"

# Custom loop: more control, better for scripts
while true; do
    status=$(curl -sf http://localhost/health && echo "UP" || echo "DOWN")
    echo "$(date '+%H:%M:%S') Health: $status"
    sleep 5
done
```

---

## systemd Timers: The Modern Cron

On modern Linux systems (Ubuntu 16+, CentOS 7+, Amazon Linux 2), systemd timers are the preferred way to schedule recurring tasks. They offer better logging, dependencies, and flexibility.

### Two-file structure

Every systemd timer requires two files:
1. A **service file** — defines WHAT to run
2. A **timer file** — defines WHEN to run

```bash
# 1. Create the service file
cat > /etc/systemd/system/myapp-backup.service << 'EOF'
[Unit]
Description=MyApp Daily Backup
After=network.target

[Service]
Type=oneshot
User=deploy
ExecStart=/opt/scripts/backup.sh
StandardOutput=journal
StandardError=journal
EOF

# 2. Create the timer file
cat > /etc/systemd/system/myapp-backup.timer << 'EOF'
[Unit]
Description=Run MyApp backup daily at 2 AM
Requires=myapp-backup.service

[Timer]
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=300
Persistent=true

[Install]
WantedBy=timers.target
EOF

# 3. Enable and start
systemctl daemon-reload
systemctl enable --now myapp-backup.timer
```

### Timer schedule syntax

```bash
# systemd timer OnCalendar examples:
OnCalendar=*-*-* 02:00:00    # every day at 2:00 AM
OnCalendar=weekly             # every week
OnCalendar=monthly            # every month
OnCalendar=Mon *-*-* 09:00   # every Monday at 9 AM
OnCalendar=*:0/15             # every 15 minutes
OnCalendar=hourly             # every hour
```

### Managing systemd timers

```bash
# List all timers (active and inactive)
systemctl list-timers --all

# Check timer status
systemctl status myapp-backup.timer
systemctl status myapp-backup.service

# View logs (this is the big advantage over cron)
journalctl -u myapp-backup.service
journalctl -u myapp-backup.service --since "24 hours ago"
journalctl -u myapp-backup.service -n 50

# Run manually (for testing)
systemctl start myapp-backup.service

# Disable
systemctl disable --now myapp-backup.timer
```

### `Persistent=true` — key advantage over cron

```bash
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
# If the machine was off at 2 AM, run the job when it next boots up
# Cron would simply miss the job. systemd catches up.
```

---

## anacron: Cron for Non-Server Systems

`anacron` is designed for laptops and desktop machines that are not always running. It guarantees that daily/weekly/monthly jobs eventually run — even if the machine was off when they were scheduled.

```bash
# Install
apt-get install anacron

# Configuration: /etc/anacrontab
cat /etc/anacrontab
# period  delay  job-identifier  command
# 1       5      cron.daily      run-parts /etc/cron.daily
# 7       10     cron.weekly     run-parts /etc/cron.weekly
# @monthly 15    cron.monthly    run-parts /etc/cron.monthly
```

Fields:
- **period** — days between runs (1=daily, 7=weekly)
- **delay** — minutes to wait after boot before running
- **identifier** — unique name for tracking
- **command** — what to run

```bash
# Custom anacron job: run backup daily, with 10 minute delay after boot
# Add to /etc/anacrontab:
1   10   personal-backup   /home/user/scripts/backup.sh
```

---

## Choosing the Right Tool

```
  Scenario                              Best tool
  ----------------------------------------
  Daily backup on a server              cron or systemd timer
  Daily backup on a laptop              anacron or systemd timer (Persistent=true)
  One-time: restart after 30 min        at
  Monitor disk space every 2 sec        watch
  Run on boot, with service dependency  systemd timer
  Need detailed logs                    systemd timer (journalctl)
  Simple, portable, always works        cron
```

---

## Summary

> **`at`** runs a command once at a future time. Manage with `atq` and `atrm`.
>
> **`watch -n N command`** repeats a command every N seconds with a refreshing display.
>
> **systemd timers** are cron's modern replacement — two files (service + timer), excellent logging via `journalctl`, `Persistent=true` for catch-up.
>
> **anacron** ensures daily/weekly/monthly jobs run even on machines that aren't always on.
>
> For production servers on systemd: prefer systemd timers for new jobs.
>
> For quick, portable cron jobs: traditional crontab is still perfectly valid.

| Tool | Use case | Recurring? |
|------|----------|-----------|
| `cron` | Regular schedules on always-on servers | Yes |
| `at` | One-shot future execution | No |
| `watch` | Real-time terminal monitoring | Continuous |
| systemd timer | Modern recurring with logging | Yes |
| anacron | Recurring on non-always-on machines | Yes |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Cron Jobs](./cron_jobs.md) &nbsp;|&nbsp; **Next:** [System Monitoring →](../08_real_world_scripts/system_monitoring.md)

**Related Topics:** [Cron Jobs](./cron_jobs.md) · [System Monitoring](../08_real_world_scripts/system_monitoring.md) · [Backup Scripts](../08_real_world_scripts/backup_scripts.md)
