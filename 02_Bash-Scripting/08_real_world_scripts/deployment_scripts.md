# Deployment Scripts

## The Surgical Checklist Analogy

Before a surgeon begins an operation, they run through a WHO Surgical Safety Checklist: patient identity confirmed, allergies noted, equipment ready, team briefed. If any item fails, the operation does not start. After the operation, they check again: instruments counted, patient stable.

This discipline — structured pre-checks, execute, post-checks, rollback if something goes wrong — is exactly what a production deployment script must do. Deploying software without this structure is like operating without a checklist: most of the time it works, but when it fails, it fails catastrophically.

```
  Deployment pipeline:

  [Pre-flight]          [Deploy]            [Verify]
  +-----------+    +--------------+    +---------------+
  | Check env |    | Pull code    |    | Health check  |
  | Check tools|-->| Run tests    |--->| Smoke test    |-->SUCCESS
  | Check deps |    | Build image  |    | Log check     |
  | Set lock   |    | Push image   |    +---------------+
  +-----------+    | Update infra |          |
                   +--------------+        FAIL
                                             |
                                             v
                                    [Rollback]
                                    +----------+
                                    | Restore  |
                                    | previous |
                                    | version  |
                                    | Notify   |
                                    +----------+
```

---

## Complete Deployment Script

```bash
#!/usr/bin/env bash
# =============================================================
# Script: deploy.sh
# Description: Deploy application to server with rollback support
# Usage: ./deploy.sh <environment> <version>
#        ./deploy.sh production v2.1.0
#        ./deploy.sh staging latest
# =============================================================

set -uo pipefail
# Note: -e is omitted to allow explicit error handling and rollback

# ---- Configuration ----
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly DEPLOY_LOG="/var/log/deploy/deploy_$(date +%Y%m%d_%H%M%S).log"
readonly LOCK_FILE="/var/run/deploy.lock"

# Application settings
readonly APP_NAME="myapp"
readonly DOCKER_REGISTRY="${DOCKER_REGISTRY:-registry.company.com}"
readonly COMPOSE_FILE="/opt/${APP_NAME}/docker-compose.yml"
readonly HEALTH_CHECK_URL_BASE="http://localhost"
readonly HEALTH_CHECK_PATH="/health"

# Deployment settings
readonly DEPLOY_USER="${DEPLOY_USER:-deploy}"
readonly MAX_HEALTH_RETRIES=20
readonly HEALTH_RETRY_INTERVAL=5   # seconds

# Environment-specific settings
declare -A ENV_PORTS=([dev]="3000" [staging]="8080" [production]="80")
declare -A ENV_REPLICAS=([dev]="1" [staging]="2" [production]="4")

# State
ENVIRONMENT=""
VERSION=""
PREVIOUS_VERSION=""
ROLLBACK_PERFORMED=false
DEPLOY_STARTED=false

# ---- Logging ----
mkdir -p "$(dirname "$DEPLOY_LOG")"

log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $message" | tee -a "$DEPLOY_LOG"
}

log_info()    { log "INFO " "$*"; }
log_warn()    { log "WARN " "$*" >&2; }
log_error()   { log "ERROR" "$*" >&2; }
log_success() { log "OK   " "$*"; }

log_section() {
    log "INFO " "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    log "INFO " "  $*"
    log "INFO " "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
}

# ---- Notification ----
notify() {
    local status="$1"
    local message="$2"
    local slack_webhook="${SLACK_WEBHOOK:-}"

    if [[ -n "$slack_webhook" ]]; then
        local icon=":white_check_mark:"
        [[ "$status" == "FAILED" ]] && icon=":x:"
        curl -sf -X POST -H 'Content-type: application/json' \
            --data "{\"text\": \"$icon *Deploy $status*: $APP_NAME $VERSION -> $ENVIRONMENT\n$message\"}" \
            "$slack_webhook" &>/dev/null || true
    fi

    if [[ -n "${ALERT_EMAIL:-}" ]]; then
        echo "$message" | mail -s "[$status] Deploy $APP_NAME $VERSION to $ENVIRONMENT" \
            "$ALERT_EMAIL" || true
    fi
}

# ---- Cleanup and lock management ----
cleanup() {
    local exit_code=${1:-$?}
    log_info "Cleanup: removing lock file"
    rm -f "$LOCK_FILE"
}

trap 'cleanup $?' EXIT

acquire_lock() {
    if [[ -f "$LOCK_FILE" ]]; then
        local other_pid
        other_pid=$(cat "$LOCK_FILE" 2>/dev/null || echo "unknown")
        log_error "Deployment already in progress (PID: $other_pid)"
        log_error "If this is stale, remove: $LOCK_FILE"
        exit 1
    fi
    echo $$ > "$LOCK_FILE"
    log_info "Lock acquired (PID: $$)"
}

# ---- Argument validation ----
parse_args() {
    if [[ $# -lt 2 ]]; then
        cat << EOF
Usage: $SCRIPT_NAME <environment> <version>

  environment: dev | staging | production
  version:     semantic version (e.g. v2.1.0) or "latest"

Examples:
  $SCRIPT_NAME staging v2.1.0
  $SCRIPT_NAME production v2.0.5
  $SCRIPT_NAME dev latest

EOF
        exit 2
    fi

    ENVIRONMENT="$1"
    VERSION="$2"

    # Validate environment
    if [[ -z "${ENV_PORTS[$ENVIRONMENT]+_}" ]]; then
        log_error "Invalid environment: '$ENVIRONMENT'"
        log_error "Valid environments: ${!ENV_PORTS[*]}"
        exit 2
    fi

    # Validate version format
    if [[ "$VERSION" != "latest" && ! "$VERSION" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
        log_error "Invalid version format: '$VERSION'"
        log_error "Expected: v1.2.3 or 'latest'"
        exit 2
    fi
}

# ---- Pre-flight checks ----
preflight_checks() {
    log_section "Pre-flight checks"

    local failed=0

    # Required commands
    for cmd in docker docker-compose curl git; do
        if command -v "$cmd" &>/dev/null; then
            log_success "Found: $cmd"
        else
            log_error "Required command not found: $cmd"
            (( failed++ )) || true
        fi
    done

    # Required environment variables
    for var in DOCKER_REGISTRY; do
        if [[ -n "${!var:-}" ]]; then
            log_success "Env var set: $var"
        else
            log_error "Required env var not set: $var"
            (( failed++ )) || true
        fi
    done

    # Docker daemon
    if docker info &>/dev/null; then
        log_success "Docker daemon is running"
    else
        log_error "Docker daemon is not running"
        (( failed++ )) || true
    fi

    # Disk space (need 5GB free)
    local avail_gb
    avail_gb=$(df -BG / | awk 'NR==2 {gsub(/G/,""); print $4}')
    if (( avail_gb >= 5 )); then
        log_success "Disk space: ${avail_gb}GB available"
    else
        log_error "Insufficient disk space: ${avail_gb}GB (need 5GB)"
        (( failed++ )) || true
    fi

    # Production requires approval token
    if [[ "$ENVIRONMENT" == "production" ]]; then
        if [[ -z "${DEPLOY_APPROVAL_TOKEN:-}" ]]; then
            log_error "DEPLOY_APPROVAL_TOKEN required for production deployments"
            (( failed++ )) || true
        else
            log_success "Approval token present"
        fi
    fi

    if [[ $failed -gt 0 ]]; then
        log_error "Pre-flight failed: $failed check(s) did not pass"
        exit 1
    fi

    log_success "All pre-flight checks passed"
}

# ---- Capture current version for rollback ----
capture_current_version() {
    log_info "Capturing current version for rollback..."
    PREVIOUS_VERSION=$(docker inspect "${APP_NAME}" 2>/dev/null \
        | grep '"Image"' | head -1 \
        | awk -F'"' '{print $4}' \
        | cut -d: -f2 || echo "")

    if [[ -n "$PREVIOUS_VERSION" ]]; then
        log_info "Current version: $PREVIOUS_VERSION (will use for rollback)"
    else
        log_warn "Could not determine current version — rollback may not be available"
    fi
}

# ---- Pull Docker image ----
pull_image() {
    log_section "Pulling Docker image"

    local image="${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}"
    log_info "Pulling: $image"

    local attempt=1
    local max_attempts=3

    while [[ $attempt -le $max_attempts ]]; do
        if docker pull "$image"; then
            log_success "Image pulled: $image"
            return 0
        fi
        log_warn "Pull attempt $attempt/$max_attempts failed"
        (( attempt++ ))
        sleep 5
    done

    log_error "Failed to pull image after $max_attempts attempts: $image"
    return 1
}

# ---- Run tests ----
run_tests() {
    log_section "Running smoke tests"

    local image="${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}"

    log_info "Running test suite in container..."
    if docker run --rm \
        -e ENVIRONMENT=test \
        "$image" \
        /app/scripts/run_tests.sh; then
        log_success "All tests passed"
    else
        log_error "Tests FAILED — aborting deployment"
        return 1
    fi
}

# ---- Deploy ----
deploy_application() {
    log_section "Deploying application"
    DEPLOY_STARTED=true

    local port="${ENV_PORTS[$ENVIRONMENT]}"
    local replicas="${ENV_REPLICAS[$ENVIRONMENT]}"
    local image="${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}"

    export APP_IMAGE="$image"
    export APP_PORT="$port"
    export APP_REPLICAS="$replicas"

    log_info "Deploying $image (port=$port, replicas=$replicas)"

    if [[ -f "$COMPOSE_FILE" ]]; then
        docker-compose -f "$COMPOSE_FILE" up -d --no-deps --scale "app=$replicas" app
    else
        # Simple docker run if no compose file
        docker stop "${APP_NAME}-old" &>/dev/null || true
        docker rename "${APP_NAME}" "${APP_NAME}-old" &>/dev/null || true
        docker run -d \
            --name "$APP_NAME" \
            --restart unless-stopped \
            -p "${port}:8080" \
            -e ENVIRONMENT="$ENVIRONMENT" \
            "$image"
    fi

    log_success "Container(s) started"
}

# ---- Health check ----
wait_for_health() {
    log_section "Health check"

    local port="${ENV_PORTS[$ENVIRONMENT]}"
    local url="${HEALTH_CHECK_URL_BASE}:${port}${HEALTH_CHECK_PATH}"
    local attempt=1

    log_info "Waiting for $url to become healthy..."

    while [[ $attempt -le $MAX_HEALTH_RETRIES ]]; do
        local http_code
        http_code=$(curl -so /dev/null -w "%{http_code}" --connect-timeout 5 "$url" 2>/dev/null || echo "000")

        if [[ "$http_code" == "200" ]]; then
            log_success "Health check passed (attempt $attempt) — HTTP $http_code"
            return 0
        fi

        log_info "Health check attempt $attempt/$MAX_HEALTH_RETRIES: HTTP $http_code"
        (( attempt++ ))
        sleep $HEALTH_RETRY_INTERVAL
    done

    log_error "Health check FAILED after $MAX_HEALTH_RETRIES attempts"
    return 1
}

# ---- Rollback ----
rollback() {
    log_section "ROLLBACK"
    ROLLBACK_PERFORMED=true

    if [[ -z "$PREVIOUS_VERSION" ]]; then
        log_error "No previous version available for rollback"
        return 1
    fi

    log_warn "Rolling back to: $PREVIOUS_VERSION"

    local prev_image="${DOCKER_REGISTRY}/${APP_NAME}:${PREVIOUS_VERSION}"

    docker stop "$APP_NAME" &>/dev/null || true
    docker rm "$APP_NAME" &>/dev/null || true

    if docker run -d \
        --name "$APP_NAME" \
        --restart unless-stopped \
        -p "${ENV_PORTS[$ENVIRONMENT]}:8080" \
        -e ENVIRONMENT="$ENVIRONMENT" \
        "$prev_image"; then
        log_success "Rolled back to: $PREVIOUS_VERSION"
    else
        log_error "Rollback FAILED — manual intervention required!"
    fi
}

# ---- Main ----
main() {
    parse_args "$@"

    log_section "Deployment started"
    log_info "App:         $APP_NAME"
    log_info "Version:     $VERSION"
    log_info "Environment: $ENVIRONMENT"
    log_info "Started by:  ${SUDO_USER:-$USER}"
    log_info "Log file:    $DEPLOY_LOG"

    acquire_lock
    preflight_checks
    capture_current_version

    # Confirm production deployments
    if [[ "$ENVIRONMENT" == "production" ]] && [[ -t 0 ]]; then
        echo ""
        log_warn "You are deploying to PRODUCTION."
        read -rp "Type 'deploy' to confirm: " confirmation
        if [[ "$confirmation" != "deploy" ]]; then
            log_info "Deployment cancelled by user"
            exit 0
        fi
    fi

    # Execute deployment
    if ! pull_image; then
        log_error "Image pull failed — aborting"
        notify "FAILED" "Image pull failed for $VERSION"
        exit 1
    fi

    if ! run_tests; then
        log_error "Tests failed — aborting"
        notify "FAILED" "Tests failed for $VERSION"
        exit 1
    fi

    deploy_application

    if ! wait_for_health; then
        log_error "Health check failed — initiating rollback"
        rollback
        notify "FAILED" "Deployment failed — rolled back to $PREVIOUS_VERSION"
        exit 1
    fi

    # Cleanup old container
    docker rm "${APP_NAME}-old" &>/dev/null || true

    local duration=$SECONDS
    log_section "Deployment SUCCESSFUL"
    log_success "Version $VERSION is live on $ENVIRONMENT"
    log_info "Duration: ${duration}s"

    notify "SUCCEEDED" "Successfully deployed $VERSION to $ENVIRONMENT in ${duration}s"
}

main "$@"
```

---

## Summary

> A production deployment script must: **validate inputs**, **check pre-conditions**, **capture current state for rollback**, **deploy**, **health check**, and **rollback on failure**.
>
> **Acquire a lock file** to prevent concurrent deployments.
>
> **Always capture the previous version** before deploying so rollback is possible.
>
> **Health checks with retries** are critical — allow time for the application to start up.
>
> **Log everything** to a timestamped file so post-mortems have complete information.
>
> **Notify your team** on both success and failure via Slack, email, or PagerDuty.

| Phase | What happens |
|-------|-------------|
| Pre-flight | Validate tools, env vars, disk space |
| Image pull | Pull new Docker image with retries |
| Test | Run tests inside new image |
| Deploy | Start new containers |
| Health check | Poll `/health` endpoint until ready |
| Rollback | Restore previous image on failure |
| Notify | Send Slack/email with result |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Backup Scripts](./backup_scripts.md) &nbsp;|&nbsp; **Next:** [Bash Interview Questions →](../99_interview_master/bash_questions.md)

**Related Topics:** [Exit Codes](../06_error_handling/exit_codes.md) · [Traps](../06_error_handling/traps.md) · [Functions](../04_functions/functions.md)
