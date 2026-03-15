# System Monitoring Script

## The Hospital Monitoring Analogy

In a hospital intensive care unit, patients are connected to monitoring equipment at all times. The monitors continuously track vital signs — heart rate, blood pressure, oxygen — and sound an alarm the moment any reading goes outside safe boundaries. A nurse does not sit and stare at every patient; the system alerts them when attention is needed.

Your servers are patients. CPU, memory, and disk usage are vital signs. A monitoring script is the ICU equipment — it checks regularly, logs the readings, and sounds the alarm when thresholds are breached.

```
  Monitoring Script Architecture:

  +------------------+
  | Cron / systemd   |  Triggers script every 5 minutes
  +------------------+
           |
           v
  +------------------+
  | Collect Metrics  |  CPU%, Memory%, Disk%, Process count
  +------------------+
           |
           v
  +------------------+
  | Compare to       |  Is CPU > 80%?
  | Thresholds       |  Is Disk > 90%?
  +------------------+
           |
     +-----+------+
     |            |
     v            v
  [OK]         [ALERT]
  Log only     Log + Notify (email/Slack/PagerDuty)
```

---

## Complete System Monitoring Script

```bash
#!/usr/bin/env bash
# =============================================================
# Script: system_monitor.sh
# Description: Monitor CPU, memory, disk and send alerts
# Usage: ./system_monitor.sh
#        Run via cron: */5 * * * * /opt/scripts/system_monitor.sh
# =============================================================

set -euo pipefail

# ---- Configuration ----
readonly SCRIPT_NAME="system_monitor"
readonly LOG_FILE="/var/log/monitoring/${SCRIPT_NAME}.log"
readonly ALERT_LOG="/var/log/monitoring/${SCRIPT_NAME}_alerts.log"

# Thresholds (percentages)
readonly CPU_THRESHOLD=80
readonly MEMORY_THRESHOLD=85
readonly DISK_THRESHOLD=90
readonly LOAD_THRESHOLD=4     # load average (adjust for CPU count)

# Notification settings
readonly ALERT_EMAIL="${ALERT_EMAIL:-}"          # set in environment or leave empty
readonly SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"       # optional Slack webhook URL
readonly HOSTNAME=$(hostname -f)

# ---- Setup ----
mkdir -p "$(dirname "$LOG_FILE")"

# ---- Logging ----
log() {
    local level="$1"
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] [$HOSTNAME] $*" | tee -a "$LOG_FILE"
}

log_alert() {
    log "ALERT" "$*"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$ALERT_LOG"
}

log_ok() {
    log "OK" "$*"
}

# ---- Notification ----
send_alert() {
    local subject="$1"
    local message="$2"

    log_alert "NOTIFICATION: $subject"

    # Email notification
    if [[ -n "$ALERT_EMAIL" ]] && command -v mail &>/dev/null; then
        echo "$message" | mail -s "[ALERT] $subject" "$ALERT_EMAIL"
        log "INFO" "Alert email sent to: $ALERT_EMAIL"
    fi

    # Slack notification
    if [[ -n "$SLACK_WEBHOOK" ]] && command -v curl &>/dev/null; then
        local payload
        payload=$(printf '{"text": ":rotating_light: *ALERT* [%s]\n%s"}' "$subject" "$message")
        curl -sf -X POST -H 'Content-type: application/json' \
            --data "$payload" "$SLACK_WEBHOOK" &>/dev/null || true
        log "INFO" "Slack alert sent"
    fi
}

# ---- Metric Collection ----
get_cpu_usage() {
    # Get CPU usage percentage (average over 1 second sample)
    top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//' | cut -d. -f1
}

get_memory_usage() {
    # Returns memory usage as percentage
    free | awk '/^Mem:/ {printf "%.0f", ($3/$2)*100}'
}

get_disk_usage() {
    local mount_point="${1:-/}"
    df -h "$mount_point" | awk 'NR==2 {gsub(/%/, "", $5); print $5}'
}

get_load_average() {
    # 1-minute load average
    uptime | awk -F'load average:' '{print $2}' | awk -F, '{gsub(/ /, "", $1); print $1}'
}

get_memory_info() {
    free -h | awk '/^Mem:/ {printf "total=%s used=%s free=%s", $2, $3, $4}'
}

# ---- Checks ----
check_cpu() {
    local usage
    usage=$(get_cpu_usage)

    if (( usage > CPU_THRESHOLD )); then
        log_alert "CPU usage CRITICAL: ${usage}% (threshold: ${CPU_THRESHOLD}%)"
        send_alert "High CPU on $HOSTNAME" "CPU usage is at ${usage}% (threshold: ${CPU_THRESHOLD}%)

Top processes:
$(ps aux --sort=-%cpu | head -6 | awk '{printf "%-30s %s%%\n", $11, $3}')

Hostname: $HOSTNAME
Time: $(date)"
        return 1
    else
        log_ok "CPU: ${usage}% (threshold: ${CPU_THRESHOLD}%)"
        return 0
    fi
}

check_memory() {
    local usage
    usage=$(get_memory_usage)
    local mem_info
    mem_info=$(get_memory_info)

    if (( usage > MEMORY_THRESHOLD )); then
        log_alert "Memory usage CRITICAL: ${usage}% (threshold: ${MEMORY_THRESHOLD}%) [$mem_info]"
        send_alert "High Memory on $HOSTNAME" "Memory usage is at ${usage}% (threshold: ${MEMORY_THRESHOLD}%)

Memory details: $mem_info

Top memory processes:
$(ps aux --sort=-%mem | head -6 | awk '{printf "%-30s %s%%\n", $11, $4}')

Hostname: $HOSTNAME
Time: $(date)"
        return 1
    else
        log_ok "Memory: ${usage}% [$mem_info] (threshold: ${MEMORY_THRESHOLD}%)"
        return 0
    fi
}

check_disk() {
    local issues=0

    # Check all mounted filesystems
    while IFS= read -r line; do
        local mount_point
        local usage
        mount_point=$(echo "$line" | awk '{print $6}')
        usage=$(echo "$line" | awk '{gsub(/%/, "", $5); print $5}')

        # Skip special filesystems
        [[ "$mount_point" =~ ^/(proc|sys|dev|run) ]] && continue
        [[ "$mount_point" =~ tmpfs ]] && continue

        if (( usage > DISK_THRESHOLD )); then
            log_alert "Disk usage CRITICAL on $mount_point: ${usage}% (threshold: ${DISK_THRESHOLD}%)"
            send_alert "High Disk Usage on $HOSTNAME:$mount_point" "Disk usage on $mount_point is at ${usage}%

$(df -h "$mount_point")

Hostname: $HOSTNAME
Time: $(date)"
            (( issues++ )) || true
        else
            log_ok "Disk $mount_point: ${usage}% (threshold: ${DISK_THRESHOLD}%)"
        fi
    done < <(df -h | grep "^/dev/")

    return $issues
}

check_load() {
    local load
    load=$(get_load_average)
    local load_int
    load_int=$(echo "$load" | cut -d. -f1)

    if (( load_int > LOAD_THRESHOLD )); then
        log_alert "Load average CRITICAL: $load (threshold: $LOAD_THRESHOLD)"
        send_alert "High Load on $HOSTNAME" "Load average is $load (threshold: $LOAD_THRESHOLD)

$(uptime)

Hostname: $HOSTNAME
Time: $(date)"
        return 1
    else
        log_ok "Load average: $load (threshold: $LOAD_THRESHOLD)"
        return 0
    fi
}

check_process() {
    local process_name="$1"
    if ! pgrep -x "$process_name" &>/dev/null; then
        log_alert "Process NOT RUNNING: $process_name"
        send_alert "Process Down: $process_name on $HOSTNAME" "The process '$process_name' is not running.

Hostname: $HOSTNAME
Time: $(date)"
        return 1
    else
        local pid
        pid=$(pgrep -x "$process_name" | head -1)
        log_ok "Process $process_name is running (PID: $pid)"
        return 0
    fi
}

# ---- Main ----
main() {
    log "INFO" "=== Monitoring run started ==="

    local issues=0

    check_cpu    || (( issues++ )) || true
    check_memory || (( issues++ )) || true
    check_disk   || (( issues++ )) || true
    check_load   || (( issues++ )) || true

    # Check critical services (customize for your environment)
    # check_process "nginx"   || (( issues++ )) || true
    # check_process "mysqld"  || (( issues++ )) || true

    if [[ $issues -eq 0 ]]; then
        log "INFO" "=== All checks passed ==="
    else
        log "ALERT" "=== $issues issue(s) detected ==="
    fi

    return $issues
}

main "$@"
```

---

## Cron Installation

```bash
# Make executable and install
chmod +x /opt/scripts/system_monitor.sh

# Test manually first
/opt/scripts/system_monitor.sh

# Add to crontab
crontab -e
# Add: */5 * * * * /opt/scripts/system_monitor.sh

# Or install as systemd timer
cat > /etc/systemd/system/system-monitor.service << 'EOF'
[Unit]
Description=System Health Monitor

[Service]
Type=oneshot
ExecStart=/opt/scripts/system_monitor.sh
EOF

cat > /etc/systemd/system/system-monitor.timer << 'EOF'
[Unit]
Description=Run system monitor every 5 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now system-monitor.timer
```

---

## Summary

> A good monitoring script **collects metrics**, **compares to thresholds**, **logs all results**, and **alerts on problems**.
>
> Keep thresholds in **variables at the top** so they are easy to adjust.
>
> Use **functions for each check** — easier to test and maintain.
>
> Log both **normal and alert states** so you can see trends over time.
>
> Support **multiple notification channels** (email, Slack, PagerDuty) through environment variables.
>
> Run via **cron or systemd timer** every 1-5 minutes in production.

| Metric | Tool | Key command |
|--------|------|-------------|
| CPU | `top`, `mpstat` | `top -bn1` |
| Memory | `free` | `free -h` |
| Disk | `df` | `df -h` |
| Load | `uptime` | `uptime` |
| Processes | `pgrep`, `ps` | `pgrep -x nginx` |
| Network | `ss`, `netstat` | `ss -tuln` |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Scheduling](../07_automation/scheduling.md) &nbsp;|&nbsp; **Next:** [Backup Scripts →](./backup_scripts.md)

**Related Topics:** [Cron Jobs](../07_automation/cron_jobs.md) · [Exit Codes](../06_error_handling/exit_codes.md) · [Traps](../06_error_handling/traps.md)
