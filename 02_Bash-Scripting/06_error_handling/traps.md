# Traps and Signal Handling in Bash

## The Fire Alarm Analogy

A responsible building manager does not just set up offices and walk away. They install fire alarms. When a fire alarm goes off (a signal), a specific response kicks in automatically: alert the fire department, open emergency exits, shut down the elevators. This happens regardless of what work was in progress at the time.

The `trap` command is your script's emergency response system. You register a handler — a function or command — that automatically runs when a specific signal or event occurs. Whether the script finishes normally, crashes, or gets interrupted with Ctrl+C, your cleanup code runs.

```
  Normal script flow:
  [start] -> [work] -> [work] -> [finish] -> EXIT

  Interrupted script flow:
  [start] -> [work] -> [Ctrl+C!]
                            |
                            v
                      TRAP fires!
                            |
                            v
                      [cleanup code runs]
                            |
                            v
                          EXIT

  Either way: cleanup code runs.
```

---

## Basic `trap` Syntax

```bash
trap 'commands' SIGNAL [SIGNAL...]
```

- `'commands'` — shell code to run when the signal is received
- `SIGNAL` — the signal name(s) to catch

```bash
# Run cleanup when script exits (any reason)
trap 'echo "Script is exiting!"' EXIT

# Run cleanup when user presses Ctrl+C
trap 'echo "Caught Ctrl+C!"' INT

# Multiple signals
trap 'cleanup' EXIT INT TERM
```

---

## The Most Important Trap: `EXIT`

The `EXIT` pseudo-signal fires when the script exits for any reason: normal completion, `exit` command, or fatal error (with `set -e`). This makes it perfect for cleanup:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Create a temp directory
TEMP_DIR=$(mktemp -d)
echo "Working in: $TEMP_DIR"

# Register cleanup — this runs no matter how the script exits
cleanup() {
    echo "Cleaning up temp directory: $TEMP_DIR"
    rm -rf "$TEMP_DIR"
}
trap cleanup EXIT

# Do work in the temp directory
cd "$TEMP_DIR"
git clone https://github.com/myorg/myapp.git .
docker build -t myapp:latest .

echo "Build complete"
# cleanup() runs automatically here — whether build succeeded or failed
```

---

## Signals to Know

```
  Signal    Number    Default action    Meaning
  --------  ------    --------------    -------
  HUP       1         Terminate         Terminal closed
  INT       2         Terminate         Ctrl+C (interrupt)
  QUIT      3         Core dump         Ctrl+\ (quit)
  TERM      15        Terminate         Polite termination (kill PID)
  KILL      9         Terminate         Cannot be caught or ignored!
  EXIT      (special) N/A               Script exits (any reason)
  ERR       (special) N/A               Any command returns non-zero
  DEBUG     (special) N/A               Before each command
```

---

## Cleaning Up Temp Files

The most common use of `trap`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Temp files and directories
TEMP_FILE=$(mktemp /tmp/deploy-XXXXXXXX)
TEMP_DIR=$(mktemp -d /tmp/build-XXXXXXXX)
LOCK_FILE="/var/run/deploy.lock"

# Cleanup function
cleanup() {
    local exit_code=$?
    echo ""
    echo "Cleaning up..."
    rm -f  "$TEMP_FILE"
    rm -rf "$TEMP_DIR"
    rm -f  "$LOCK_FILE"

    if [[ $exit_code -ne 0 ]]; then
        echo "Script exited with error code: $exit_code"
    fi
}

trap cleanup EXIT

# Create a lock file to prevent concurrent runs
if [[ -f "$LOCK_FILE" ]]; then
    echo "ERROR: Another deployment is already running (lock: $LOCK_FILE)"
    exit 1
fi
touch "$LOCK_FILE"

# Do the real work...
echo "Deploying..."
# ... deployment steps
echo "Done."
```

---

## Catching Ctrl+C (SIGINT)

```bash
#!/usr/bin/env bash

interrupted=false

handle_interrupt() {
    echo ""
    echo "Caught Ctrl+C — stopping gracefully..."
    interrupted=true
}

trap handle_interrupt INT

servers=("web01" "web02" "web03" "web04" "web05")

for server in "${servers[@]}"; do
    if [[ "$interrupted" == true ]]; then
        echo "Interrupted — processed ${#processed[@]} servers"
        exit 130    # 128 + 2 (SIGINT) = 130 (conventional exit code)
    fi
    echo "Processing: $server"
    sleep 2    # simulate work
done

echo "All servers processed."
```

---

## `trap '' SIGNAL`: Ignoring Signals

You can tell the script to ignore certain signals during a critical section:

```bash
#!/usr/bin/env bash

echo "Starting critical database migration..."

# Ignore Ctrl+C and TERM during the critical section
trap '' INT TERM

# Critical section — DO NOT INTERRUPT
echo "Running migration... (cannot be interrupted)"
mysql -u root mydb < migration.sql
echo "Migration complete."

# Restore normal signal handling
trap - INT TERM

echo "You can now interrupt safely."
sleep 100
```

---

## `trap` with `ERR`: Error Detection

The `ERR` pseudo-signal fires when any command fails (similar to `set -e` but you control the response):

```bash
#!/usr/bin/env bash
set -uo pipefail    # Note: no -e here since ERR trap handles errors

DEPLOYMENT_STEP=""

on_error() {
    local exit_code=$?
    local line_number=$1
    echo ""
    echo "============================="
    echo "ERROR DETECTED"
    echo "============================="
    echo "Exit code:   $exit_code"
    echo "Line number: $line_number"
    echo "Step:        $DEPLOYMENT_STEP"
    echo ""
    echo "Initiating rollback..."
    rollback
    exit $exit_code
}

# $LINENO is passed to the handler
trap 'on_error $LINENO' ERR

rollback() {
    echo "Rolling back deployment..."
    # ... rollback logic
}

DEPLOYMENT_STEP="pulling image"
docker pull myapp:v2.1.0

DEPLOYMENT_STEP="stopping old container"
docker stop myapp-old

DEPLOYMENT_STEP="starting new container"
docker run -d --name myapp myapp:v2.1.0

DEPLOYMENT_STEP="health check"
curl -sf http://localhost/health

echo "Deployment successful!"
```

---

## Complete Production Pattern

```bash
#!/usr/bin/env bash
# deploy.sh — production deployment with full trap handling

set -euo pipefail

# ---- State tracking ----
readonly LOCK_FILE="/var/run/app_deploy.lock"
readonly LOG_FILE="/var/log/deploy/deploy_$(date +%Y%m%d_%H%M%S).log"
TEMP_DIR=""
ROLLBACK_NEEDED=false

# ---- Cleanup and handlers ----
cleanup() {
    local exit_code=${1:-$?}
    echo "Cleanup: removing temp files and lock"
    [[ -n "$TEMP_DIR" ]] && rm -rf "$TEMP_DIR"
    rm -f "$LOCK_FILE"
}

on_interrupt() {
    echo ""
    echo "Deployment interrupted by user"
    if [[ "$ROLLBACK_NEEDED" == true ]]; then
        echo "Rollback needed — initiating..."
        rollback
    fi
    cleanup 130
    exit 130
}

on_error() {
    echo "Deployment failed on line $1 (exit code $2)"
    if [[ "$ROLLBACK_NEEDED" == true ]]; then
        rollback
    fi
    cleanup "$2"
    exit "$2"
}

rollback() {
    echo "Rolling back to previous version..."
    # ... rollback logic
}

# ---- Register all traps ----
trap 'cleanup $?' EXIT
trap 'on_interrupt' INT TERM
trap 'on_error $LINENO $?' ERR

# ---- Prevent concurrent runs ----
if [[ -f "$LOCK_FILE" ]]; then
    echo "ERROR: Deployment already running (PID: $(cat "$LOCK_FILE"))"
    exit 1
fi
echo $$ > "$LOCK_FILE"

# ---- Actual deployment ----
mkdir -p "$(dirname "$LOG_FILE")"
TEMP_DIR=$(mktemp -d)

echo "Deployment started: $(date)" | tee -a "$LOG_FILE"

ROLLBACK_NEEDED=true

docker pull myapp:latest
docker-compose up -d --no-deps app

ROLLBACK_NEEDED=false

echo "Deployment successful: $(date)" | tee -a "$LOG_FILE"
```

---

## Summary

> **`trap 'commands' SIGNAL`** registers code to run when a signal is received.
>
> **`EXIT`** is the most important — fires when the script exits for any reason. Use it for cleanup.
>
> **`INT`** catches Ctrl+C (SIGINT). Handle it to stop gracefully instead of mid-operation.
>
> **`TERM`** catches `kill PID`. Handle it so long-running scripts shut down cleanly.
>
> **`ERR`** fires when a command fails. Use with `$LINENO` and `$?` to build detailed error reports.
>
> **`trap '' SIGNAL`** ignores a signal — use during critical sections that must not be interrupted.
>
> **`trap - SIGNAL`** restores default signal handling.

| Trap | Fires when |
|------|-----------|
| `EXIT` | Script exits (any reason) |
| `INT` | Ctrl+C pressed |
| `TERM` | `kill PID` received |
| `ERR` | Any command returns non-zero |
| `DEBUG` | Before each command |
| `HUP` | Terminal closed |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Exit Codes](./exit_codes.md) &nbsp;|&nbsp; **Next:** [Debugging →](./debugging.md)

**Related Topics:** [Exit Codes](./exit_codes.md) · [Deployment Scripts](../08_real_world_scripts/deployment_scripts.md) · [Functions](../04_functions/functions.md)
