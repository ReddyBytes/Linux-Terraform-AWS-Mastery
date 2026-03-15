# Cron Jobs

## The Office Cleaner Analogy

Every office building has a cleaning crew that arrives at exactly 10 PM every weeknight and at 8 AM every Saturday morning. Nobody has to call them, remind them, or supervise them. They arrive, do the work, and leave — like clockwork. If the office manager needs the bins emptied at midnight every day, they arrange that schedule once and it runs forever.

Cron is the Linux scheduler — your building's cleaning crew coordinator. You define a schedule once, attach a command to it, and the system runs that command automatically at the specified times, forever, without any manual intervention.

```
  crontab (schedule table):
  +-----------------------------------------------------------+
  | # minute hour day month weekday command                   |
  |   0      2    *   *     *       /opt/scripts/backup.sh    |
  |   */5    *    *   *     *       /opt/scripts/health_check |
  |   0      0    1   *     *       /opt/scripts/monthly.sh   |
  +-----------------------------------------------------------+
         |
         v
  crond (cron daemon) reads this table and runs commands on schedule
```

---

## Cron Syntax

Each cron job is one line with exactly six fields:

```
  ┌──────────── minute (0-59)
  │ ┌────────── hour (0-23)
  │ │ ┌──────── day of month (1-31)
  │ │ │ ┌────── month (1-12)
  │ │ │ │ ┌──── day of week (0-7, 0 and 7 = Sunday)
  │ │ │ │ │
  * * * * *  command to execute
```

### Special characters

```
  *     every (any value)
  ,     list — 1,3,5 means 1 AND 3 AND 5
  -     range — 1-5 means 1,2,3,4,5
  /     step — */5 means every 5 units
```

---

## Common Schedule Examples

```bash
# Every minute
* * * * *  /opt/scripts/check.sh

# Every 5 minutes
*/5 * * * *  /opt/scripts/health_check.sh

# Every hour (at minute 0)
0 * * * *  /opt/scripts/hourly.sh

# Every day at 2:30 AM
30 2 * * *  /opt/scripts/daily_backup.sh

# Every Monday at 9:00 AM
0 9 * * 1  /opt/scripts/weekly_report.sh

# Every weekday (Mon-Fri) at 8:00 AM
0 8 * * 1-5  /opt/scripts/workday_start.sh

# First day of every month at midnight
0 0 1 * *  /opt/scripts/monthly_cleanup.sh

# Every 15 minutes during business hours on weekdays
*/15 9-17 * * 1-5  /opt/scripts/business_check.sh

# Twice a day (midnight and noon)
0 0,12 * * *  /opt/scripts/twice_daily.sh

# Every Sunday at 3 AM
0 3 * * 0  /opt/scripts/weekly_maintenance.sh
```

---

## Special Cron Strings

These shortcuts replace the five time fields:

```bash
@reboot    # Run once at system startup
@yearly    # Run once a year (same as: 0 0 1 1 *)
@monthly   # Run once a month (same as: 0 0 1 * *)
@weekly    # Run once a week (same as: 0 0 * * 0)
@daily     # Run once a day (same as: 0 0 * * *)
@hourly    # Run once an hour (same as: 0 * * * *)
```

```bash
# Start a monitoring agent after every reboot
@reboot /opt/monitoring/agent start

# Daily database backup at midnight
@daily /opt/scripts/db_backup.sh
```

---

## Managing Crontabs

```bash
# Edit your personal crontab
crontab -e

# View your current crontab
crontab -l

# Remove your entire crontab (be careful!)
crontab -r

# Edit another user's crontab (root only)
crontab -e -u www-data
crontab -l -u deploy

# System-wide crontabs (not user-specific)
# /etc/crontab           — system crontab (has username field)
# /etc/cron.d/           — drop-in cron files
# /etc/cron.daily/       — scripts run daily
# /etc/cron.weekly/      — scripts run weekly
# /etc/cron.monthly/     — scripts run monthly
```

System crontab format (has an extra username field):
```
# /etc/crontab
# minute hour day month weekday USER command
  0      2    *   *     *       root /opt/scripts/backup.sh
  */5    *    *   *     *       www-data /opt/scripts/check.sh
```

---

## The Cron Environment Problem

This is the most common reason cron jobs fail silently: cron runs with a minimal environment, not your full login shell.

```bash
# Your interactive shell PATH:
echo $PATH
# /home/ubuntu/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Cron's default PATH:
# /usr/bin:/bin   (minimal — your custom tools may not be found!)
```

Solutions:

```bash
# Option 1: Use full paths for all commands
0 2 * * * /usr/bin/docker pull myapp:latest && /usr/bin/docker-compose up -d

# Option 2: Set PATH in the crontab header
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 2 * * * docker pull myapp:latest

# Option 3: Source the profile in the script
#!/usr/bin/env bash
source /etc/profile
source ~/.bashrc
# ... rest of script
```

Also: cron does not have access to environment variables set in `.bashrc`. Pass them explicitly:

```bash
# In crontab:
0 2 * * * AWS_PROFILE=production AWS_REGION=us-east-1 /opt/scripts/backup.sh

# Or set in crontab header:
AWS_PROFILE=production
AWS_REGION=us-east-1
0 2 * * * /opt/scripts/backup.sh
```

---

## Logging Cron Output

By default, cron sends output as email to the user. In most server setups, you want file logging instead:

```bash
# Discard all output (silent — dangerous: you lose error messages)
0 2 * * * /opt/scripts/backup.sh > /dev/null 2>&1

# Log stdout to file, discard stderr (bad: errors lost)
0 2 * * * /opt/scripts/backup.sh > /var/log/backup.log

# Log BOTH stdout and stderr to file (good)
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# Log with timestamp in filename (each run gets its own file)
0 2 * * * /opt/scripts/backup.sh > /var/log/backup_$(date +\%Y\%m\%d).log 2>&1

# Use tee to log while also sending to syslog
0 2 * * * /opt/scripts/backup.sh 2>&1 | /usr/bin/logger -t backup_cron
```

Build logging into your script itself (better practice):

```bash
#!/usr/bin/env bash
# backup.sh — always logs to its own file

LOG_FILE="/var/log/myapp/backup_$(date +%Y%m%d).log"
mkdir -p "$(dirname "$LOG_FILE")"

exec >> "$LOG_FILE" 2>&1    # redirect all output to log file
echo "=== Backup started: $(date) ==="

# ... backup logic

echo "=== Backup completed: $(date) ==="
```

Then the crontab is simple:
```bash
0 2 * * * /opt/scripts/backup.sh
```

---

## Complete Crontab Example for a Web Application

```bash
# /etc/cron.d/myapp

# Environment
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash

# Health check every 5 minutes
*/5 * * * *  deploy /opt/myapp/scripts/health_check.sh

# Log rotation: daily at 12:05 AM (5 min after midnight, after logrotate)
5 0 * * *  root logrotate /etc/logrotate.d/myapp

# Database backup: every day at 2:00 AM
0 2 * * *  deploy /opt/myapp/scripts/db_backup.sh >> /var/log/myapp/backup.log 2>&1

# Weekly report: every Sunday at 8:00 AM
0 8 * * 0  deploy /opt/myapp/scripts/weekly_report.sh

# Certificate renewal check: twice daily
0 */12 * * *  root certbot renew --quiet

# Cleanup temp files older than 7 days: daily at 3 AM
0 3 * * *  root find /tmp -name "myapp-*" -mtime +7 -delete

# On reboot: start monitoring agent
@reboot  deploy /opt/monitoring/start_agent.sh
```

---

## Testing Cron Jobs

Before relying on cron, test the command manually as the cron user:

```bash
# Test as yourself first
/opt/scripts/backup.sh

# Then simulate cron's minimal environment
env -i HOME=/root SHELL=/bin/bash PATH=/usr/bin:/bin /opt/scripts/backup.sh

# Or switch to the cron user
sudo -u www-data bash -c '/opt/scripts/backup.sh'
```

Use `cron-apt` or check syslog for cron execution:
```bash
grep CRON /var/log/syslog | tail -20
grep CRON /var/log/cron | tail -20
journalctl -u cron | tail -20
```

---

## Summary

> **Cron** runs commands on a schedule without human intervention — your automation backbone.
>
> **Syntax**: `minute hour day month weekday command` — five time fields then the command.
>
> **`*`** means every, **`,`** means list, **`-`** means range, **`/`** means step.
>
> **`crontab -e`** edits your personal crontab. **`/etc/cron.d/`** holds system job files.
>
> **Cron environment is minimal** — always use full command paths or set PATH in the crontab.
>
> **Always log output** — silent failures in cron are the hardest to debug.
>
> **`@reboot`** runs a command once at system startup. Great for starting services.

| Schedule | Expression |
|----------|-----------|
| Every minute | `* * * * *` |
| Every 5 minutes | `*/5 * * * *` |
| Every hour | `0 * * * *` |
| Daily at 2 AM | `0 2 * * *` |
| Weekdays at 8 AM | `0 8 * * 1-5` |
| First of month | `0 0 1 * *` |
| At startup | `@reboot` |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Debugging](../06_error_handling/debugging.md) &nbsp;|&nbsp; **Next:** [Scheduling →](./scheduling.md)

**Related Topics:** [Scheduling](./scheduling.md) · [System Monitoring](../08_real_world_scripts/system_monitoring.md) · [Backup Scripts](../08_real_world_scripts/backup_scripts.md)
