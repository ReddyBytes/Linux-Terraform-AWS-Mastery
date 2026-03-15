# Backup Scripts

## The Safety Deposit Box Analogy

A prudent person does not keep their only copy of important documents at home. They make copies and store them somewhere else — a safety deposit box at a bank, a fireproof safe offsite. The 3-2-1 backup rule captures this: **3 copies**, on **2 different media types**, with **1 copy offsite**.

Automated backup scripts implement this discipline. They run on a schedule, copy your data to one or more locations, compress it to save space, rotate old backups to manage storage, and log everything so you know the backup worked when you need to restore.

```
  Backup strategy (3-2-1 rule):

  Production Server         Backup Destinations
  +----------------+        +------------------+
  | /var/www/html  | -----> | Local: /backups/ |  (copy 1, medium 1)
  | /etc/          |        | Remote: NAS/SSH  |  (copy 2, medium 2)
  | /var/lib/mysql |        | Cloud: S3        |  (copy 3, offsite)
  +----------------+        +------------------+

  Rotation: keep daily(7), weekly(4), monthly(12)
```

---

## Complete Backup Script

```bash
#!/usr/bin/env bash
# =============================================================
# Script: backup.sh
# Description: Full application backup with rotation and S3 upload
# Usage: ./backup.sh [--dry-run] [--no-s3]
# Schedule: 0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
# =============================================================

set -euo pipefail

# ---- Configuration ----
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
readonly DATE=$(date '+%Y%m%d')
readonly HOSTNAME=$(hostname -s)
readonly LOG_FILE="/var/log/backup/backup_${DATE}.log"

# Source directories to back up
BACKUP_SOURCES=(
    "/var/www/html"
    "/etc/nginx"
    "/etc/myapp"
)

# Backup destinations
LOCAL_BACKUP_DIR="/var/backups/myapp"
REMOTE_BACKUP_USER="backup"
REMOTE_BACKUP_HOST="backup.internal"
REMOTE_BACKUP_DIR="/backups/${HOSTNAME}"

# S3 settings
S3_BUCKET="${S3_BACKUP_BUCKET:-}"
S3_PREFIX="backups/${HOSTNAME}"

# Retention settings
KEEP_DAILY=7
KEEP_WEEKLY=4
KEEP_MONTHLY=12

# Flags
DRY_RUN=false
SKIP_S3=false

# ---- Parse arguments ----
for arg in "$@"; do
    case "$arg" in
        --dry-run)  DRY_RUN=true ;;
        --no-s3)    SKIP_S3=true ;;
        --help)
            echo "Usage: $0 [--dry-run] [--no-s3]"
            exit 0
            ;;
        *)
            echo "Unknown option: $arg"
            exit 1
            ;;
    esac
done

# ---- Logging ----
mkdir -p "$(dirname "$LOG_FILE")"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

log_section() {
    log "============================================"
    log "  $*"
    log "============================================"
}

# ---- Dry run wrapper ----
run_cmd() {
    if [[ "$DRY_RUN" == true ]]; then
        log "[DRY-RUN] Would run: $*"
    else
        "$@"
    fi
}

# ---- Pre-flight checks ----
preflight_checks() {
    log_section "Pre-flight checks"

    # Check sources exist
    for src in "${BACKUP_SOURCES[@]}"; do
        if [[ ! -d "$src" && ! -f "$src" ]]; then
            log "WARNING: Source not found, skipping: $src"
        else
            log "Source OK: $src"
        fi
    done

    # Check available disk space (need at least 2GB)
    local available_gb
    available_gb=$(df -BG "$LOCAL_BACKUP_DIR" 2>/dev/null | awk 'NR==2 {gsub(/G/, "", $4); print $4}' || echo "0")
    if (( available_gb < 2 )); then
        log "ERROR: Insufficient disk space for backup: ${available_gb}GB available"
        exit 1
    fi
    log "Disk space available: ${available_gb}GB"

    # Check rsync is available
    command -v rsync &>/dev/null || { log "ERROR: rsync not found"; exit 1; }
}

# ---- Database backup ----
backup_database() {
    log_section "Database backup"

    local db_backup_file="${LOCAL_BACKUP_DIR}/db_${TIMESTAMP}.sql.gz"

    if ! command -v mysqldump &>/dev/null; then
        log "mysqldump not found — skipping database backup"
        return 0
    fi

    log "Dumping database..."
    run_cmd bash -c "mysqldump \
        --single-transaction \
        --quick \
        --lock-tables=false \
        --all-databases \
        2>>'$LOG_FILE' | gzip > '$db_backup_file'"

    if [[ "$DRY_RUN" == false && -f "$db_backup_file" ]]; then
        local size
        size=$(du -sh "$db_backup_file" | cut -f1)
        log "Database backup complete: $db_backup_file ($size)"
    fi
}

# ---- File backup using rsync ----
backup_files() {
    log_section "File backup (rsync)"

    local backup_dest="${LOCAL_BACKUP_DIR}/files_${TIMESTAMP}"

    # Use hard-link based incremental backup (rsync --link-dest)
    local latest_backup
    latest_backup=$(ls -1d "${LOCAL_BACKUP_DIR}/files_"* 2>/dev/null | sort | tail -1 || echo "")

    local rsync_opts=(-avz --delete --stats)
    if [[ -n "$latest_backup" && -d "$latest_backup" ]]; then
        rsync_opts+=(--link-dest="$latest_backup")
        log "Using previous backup for hard-links: $latest_backup"
    fi

    run_cmd mkdir -p "$backup_dest"

    for src in "${BACKUP_SOURCES[@]}"; do
        [[ -d "$src" || -f "$src" ]] || continue
        log "Backing up: $src"
        run_cmd rsync "${rsync_opts[@]}" "$src" "$backup_dest/" 2>>"$LOG_FILE" || {
            log "WARNING: rsync failed for $src (exit code $?)"
        }
    done

    log "File backup complete: $backup_dest"
    echo "$backup_dest"    # return path for use by caller
}

# ---- Compress backup ----
compress_backup() {
    local backup_dir="$1"
    local archive="${backup_dir}.tar.gz"

    log "Compressing: $backup_dir -> $archive"
    run_cmd tar -czf "$archive" -C "$(dirname "$backup_dir")" "$(basename "$backup_dir")"

    if [[ "$DRY_RUN" == false ]]; then
        run_cmd rm -rf "$backup_dir"
        local size
        size=$(du -sh "$archive" | cut -f1)
        log "Compressed backup: $archive ($size)"
    fi

    echo "$archive"
}

# ---- Remote sync ----
sync_to_remote() {
    local local_dir="$1"

    if [[ -z "$REMOTE_BACKUP_HOST" ]]; then
        log "No remote host configured — skipping remote sync"
        return 0
    fi

    log_section "Remote sync to $REMOTE_BACKUP_HOST"
    run_cmd rsync -avz --delete \
        "${local_dir}/" \
        "${REMOTE_BACKUP_USER}@${REMOTE_BACKUP_HOST}:${REMOTE_BACKUP_DIR}/" \
        2>>"$LOG_FILE" || {
        log "WARNING: Remote sync failed (continuing)"
    }
}

# ---- S3 upload ----
upload_to_s3() {
    local file="$1"

    if [[ "$SKIP_S3" == true || -z "$S3_BUCKET" ]]; then
        log "S3 upload skipped"
        return 0
    fi

    if ! command -v aws &>/dev/null; then
        log "WARNING: aws CLI not found — skipping S3 upload"
        return 0
    fi

    log_section "S3 upload"
    local s3_key="${S3_PREFIX}/$(basename "$file")"
    log "Uploading to s3://${S3_BUCKET}/${s3_key}"
    run_cmd aws s3 cp "$file" "s3://${S3_BUCKET}/${s3_key}" \
        --storage-class STANDARD_IA \
        2>>"$LOG_FILE" || {
        log "WARNING: S3 upload failed (continuing)"
    }
}

# ---- Rotation: remove old backups ----
rotate_backups() {
    log_section "Backup rotation"

    # Keep last N daily backups
    local daily_backups
    mapfile -t daily_backups < <(ls -1t "${LOCAL_BACKUP_DIR}"/files_*.tar.gz 2>/dev/null || true)

    local count=${#daily_backups[@]}
    log "Found $count backup(s). Keeping last $KEEP_DAILY."

    if (( count > KEEP_DAILY )); then
        local to_delete=("${daily_backups[@]:$KEEP_DAILY}")
        for old in "${to_delete[@]}"; do
            log "Removing old backup: $old"
            run_cmd rm -f "$old"
        done
    fi

    # Clean up database backups older than 30 days
    log "Removing database backups older than 30 days"
    run_cmd find "$LOCAL_BACKUP_DIR" -name "db_*.sql.gz" -mtime +30 -exec rm -f {} \;
}

# ---- Verify backup ----
verify_backup() {
    local archive="$1"

    log_section "Backup verification"
    if [[ "$DRY_RUN" == true ]]; then
        log "[DRY-RUN] Skipping verification"
        return 0
    fi

    log "Testing archive integrity: $archive"
    if tar -tzf "$archive" &>/dev/null; then
        local file_count
        file_count=$(tar -tzf "$archive" | wc -l)
        log "Verification passed: $file_count files in archive"
        return 0
    else
        log "ERROR: Archive is corrupt or incomplete: $archive"
        return 1
    fi
}

# ---- Summary report ----
send_summary() {
    local status="$1"
    local duration="$2"

    log_section "Backup summary"
    log "Status:   $status"
    log "Duration: ${duration}s"
    log "Log:      $LOG_FILE"

    if [[ -n "${ALERT_EMAIL:-}" ]] && command -v mail &>/dev/null; then
        mail -s "[Backup] $status — $HOSTNAME — $(date '+%Y-%m-%d')" "$ALERT_EMAIL" < "$LOG_FILE"
    fi
}

# ---- Main ----
main() {
    local start_time=$SECONDS

    log_section "Backup started: $HOSTNAME — $TIMESTAMP"
    [[ "$DRY_RUN" == true ]] && log "*** DRY RUN MODE — no changes will be made ***"

    mkdir -p "$LOCAL_BACKUP_DIR"

    # Run backup steps
    preflight_checks
    backup_database
    local files_dir
    files_dir=$(backup_files)

    local archive=""
    if [[ "$DRY_RUN" == false ]]; then
        archive=$(compress_backup "$files_dir")
        verify_backup "$archive"
    fi

    sync_to_remote "$LOCAL_BACKUP_DIR"
    [[ -n "$archive" ]] && upload_to_s3 "$archive"
    rotate_backups

    local duration=$(( SECONDS - start_time ))
    send_summary "SUCCESS" "$duration"
    log_section "Backup complete in ${duration}s"
}

main "$@"
```

---

## Summary

> A production backup script needs: **source definition**, **multiple destinations**, **compression**, **rotation**, **verification**, and **logging**.
>
> **`rsync --link-dest`** creates incremental backups using hard links — only changed files take new space.
>
> **`tar -czf`** compresses directories into portable archives. **`tar -tzf`** verifies them.
>
> **Rotation** prevents backup storage from growing indefinitely. Keep a sensible number of daily, weekly, and monthly copies.
>
> **Always verify** the archive after creating it — a backup you cannot restore is not a backup.
>
> **`--dry-run`** mode is invaluable for testing and debugging backup scripts safely.

| Step | Tool | Purpose |
|------|------|---------|
| Copy files | `rsync` | Efficient, incremental sync |
| Compress | `tar -czf` | Reduce storage size |
| Verify | `tar -tzf` | Confirm archive integrity |
| Rotate | `find -mtime` | Remove old backups |
| Remote sync | `rsync` over SSH | Second copy offsite |
| Cloud backup | `aws s3 cp` | Third copy in cloud |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← System Monitoring](./system_monitoring.md) &nbsp;|&nbsp; **Next:** [Deployment Scripts →](./deployment_scripts.md)

**Related Topics:** [Cron Jobs](../07_automation/cron_jobs.md) · [File Operations](../05_input_output/file_operations.md) · [Exit Codes](../06_error_handling/exit_codes.md)
