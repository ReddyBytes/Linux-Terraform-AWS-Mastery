# Scope and Return Values in Bash

## The Office Memo Analogy

Imagine a company with two offices: a large open floor (the global space) and private meeting rooms (function spaces). A whiteboard in the open floor is visible to everyone. Notes written inside a meeting room are private — when the meeting ends and everyone leaves, the notes on that room's whiteboard are erased.

In Bash, variables work the same way. Variables declared outside functions live on the "open floor" whiteboard and are visible everywhere. Variables declared with `local` inside a function live on the meeting room whiteboard — they disappear when the function returns.

```
  Global scope (open floor):
  +--------------------------------------------+
  |  ENVIRONMENT="production"  (visible to ALL) |
  |  VERSION="v2.1.0"          (visible to ALL) |
  |                                             |
  |  +-- Function scope (meeting room) ------+  |
  |  |  local temp_file="/tmp/work.txt"      |  |
  |  |  local count=0                        |  |
  |  |  (only visible INSIDE this function)  |  |
  |  +----------------------------------------+  |
  |  After function returns: temp_file gone   |
  +--------------------------------------------+
```

---

## Variable Scope in Detail

```bash
#!/usr/bin/env bash

# Global variable — visible everywhere
GLOBAL_VAR="I am global"

function scope_demo() {
    # Local variable — only inside this function
    local local_var="I am local"

    # Can read global variables
    echo "Inside function, global: $GLOBAL_VAR"
    echo "Inside function, local:  $local_var"

    # Can modify global variables (without local keyword)
    GLOBAL_VAR="I was changed by the function"
}

echo "Before: $GLOBAL_VAR"     # I am global
scope_demo
echo "After: $GLOBAL_VAR"      # I was changed by the function
echo "Local outside: $local_var"  # (empty — local_var does not exist here)
```

### Why `local` matters in real scripts

```bash
#!/usr/bin/env bash

# Bug scenario without local
count=100    # important script counter

process_items() {
    count=0    # FORGOT local — this overwrites the global count!
    for item in "$@"; do
        echo "Processing: $item"
        (( count++ ))
    done
    echo "Processed $count items"
}

echo "Starting count: $count"   # 100
process_items apple banana cherry
echo "Final count: $count"      # Expected: 100, Got: 3 (BUG!)

# Fixed version with local
process_items() {
    local count=0    # local — does NOT affect global count
    for item in "$@"; do
        echo "Processing: $item"
        (( count++ ))
    done
    echo "Processed $count items"
}
```

---

## Return Values: Three Strategies

Bash functions cannot "return" a value the way Python or JavaScript can. Instead, there are three strategies:

### Strategy 1: Exit/Return Code (success or failure)

```bash
function file_exists() {
    local filepath="$1"
    [[ -f "$filepath" ]]    # returns 0 if true, 1 if false
}

# Use in if statement
if file_exists "/etc/nginx/nginx.conf"; then
    echo "Config found"
else
    echo "Config missing"
fi

# Use with &&
file_exists "/etc/hosts" && echo "hosts file exists"
```

Use this when the function's "return value" is just yes/no, success/failure.

### Strategy 2: `echo` and Command Substitution

```bash
function get_container_ip() {
    local container_name="$1"
    docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$container_name"
}

# Capture the printed value
container_ip=$(get_container_ip "my-app")
echo "Container IP: $container_ip"
```

This is the most common pattern for returning actual data. The function prints to stdout, and the caller captures it with `$(...)`.

```
  function get_version() {       result=$(get_version "app")
    echo "v2.1.0"         ---->  echo "$result"  # v2.1.0
  }
```

**Important caveat:** `echo` inside a function captured with `$(...)` captures ALL stdout. Be careful not to accidentally capture debug messages:

```bash
function get_value() {
    echo "Debug: starting calculation" >&2    # send debug to stderr, not stdout
    echo "the-actual-value"                   # this is what gets captured
}

result=$(get_value)           # captures only "the-actual-value"
echo "Got: $result"
```

### Strategy 3: Output via a Named Variable (nameref / indirect reference)

For returning multiple values without subshell overhead, write to a named variable:

```bash
# Method A: Global variable as output channel
function get_server_info() {
    local host="$1"
    # Write results to predefined output variables
    SERVER_IP=$(dig +short "$host" | head -1)
    SERVER_STATUS=$(curl -so /dev/null -w "%{http_code}" "http://$host/health")
}

get_server_info "myapp.com"
echo "IP: $SERVER_IP  Status: $SERVER_STATUS"

# Method B: Bash 4.3+ nameref — pass variable name as argument
function set_result() {
    local -n result_var="$1"    # nameref: result_var IS the variable named by $1
    local value="$2"
    result_var="$value"         # this sets the CALLER'S variable
}

my_output=""
set_result my_output "computed value"
echo "$my_output"    # computed value
```

---

## Subshell Scope Trap

Functions run in the current shell, but command substitution `$(...)` creates a subshell. Variable changes inside `$(...)` do not persist:

```bash
COUNTER=0

increment() {
    (( COUNTER++ ))
    echo "Inside: $COUNTER"
}

# TRAP: calling via $() creates a subshell
result=$(increment)
echo "Outside: $COUNTER"   # Still 0! The subshell had its own COUNTER.
echo "Captured: $result"   # "Inside: 1" — but the increment was lost

# CORRECT: call directly (no subshell)
increment
echo "Outside: $COUNTER"   # 1 — works correctly
```

This is one of the most common and confusing Bash gotchas. Remember: if you need a function to modify variables, call it directly. If you need it to return data, use `echo` + `$()`.

---

## Returning Multiple Values

```bash
function parse_url() {
    local url="$1"
    # Use regex to extract parts
    if [[ "$url" =~ ^(https?)://([^/:]+):?([0-9]*)(/.*)? ]]; then
        PARSED_SCHEME="${BASH_REMATCH[1]}"
        PARSED_HOST="${BASH_REMATCH[2]}"
        PARSED_PORT="${BASH_REMATCH[3]:-80}"
        PARSED_PATH="${BASH_REMATCH[4]:-/}"
    else
        echo "ERROR: Invalid URL: $url" >&2
        return 1
    fi
}

parse_url "https://myapp.com:8443/api/v1"
echo "Scheme: $PARSED_SCHEME"   # https
echo "Host:   $PARSED_HOST"     # myapp.com
echo "Port:   $PARSED_PORT"     # 8443
echo "Path:   $PARSED_PATH"     # /api/v1
```

---

## Exporting Functions

You can export functions so they are available in child processes and subshells:

```bash
function log() {
    echo "[$(date '+%H:%M:%S')] $*"
}

# Export the function
export -f log

# Now subshells and child scripts can call log()
bash -c 'log "Hello from subshell"'
```

This is useful when your functions need to be available inside `xargs`, `parallel`, or scripts you call from your main script.

---

## Complete Pattern: Function Library

```bash
#!/usr/bin/env bash
# lib/utils.sh — source this in other scripts

# Strict mode
set -euo pipefail

# ---- Logging ---------------------------------------------------
readonly LOG_TIMESTAMP_FORMAT='%Y-%m-%d %H:%M:%S'

log()      { echo "[$(date +"$LOG_TIMESTAMP_FORMAT")] [INFO]  $*"; }
log_warn() { echo "[$(date +"$LOG_TIMESTAMP_FORMAT")] [WARN]  $*" >&2; }
log_err()  { echo "[$(date +"$LOG_TIMESTAMP_FORMAT")] [ERROR] $*" >&2; }

# ---- Validation ------------------------------------------------
require_var() {
    local var="$1"
    [[ -n "${!var}" ]] || { log_err "Required variable not set: $var"; exit 1; }
}

require_cmd() {
    command -v "$1" &>/dev/null || { log_err "Required command not found: $1"; exit 1; }
}

# ---- AWS helpers -----------------------------------------------
get_instance_id() {
    curl -sf "http://169.254.169.254/latest/meta-data/instance-id" 2>/dev/null
}

get_aws_region() {
    aws configure get region 2>/dev/null || echo "${AWS_DEFAULT_REGION:-us-east-1}"
}
```

Usage in another script:
```bash
#!/usr/bin/env bash
source "$(dirname "$0")/lib/utils.sh"

require_cmd aws
require_var AWS_ACCESS_KEY_ID

log "Script starting on $(get_instance_id)"
```

---

## Summary

> **`local`** creates function-scoped variables. Without it, assignments affect global scope.
>
> **Return codes** (`return 0/1`) signal success/failure. Use in `if` and `&&/||` chains.
>
> **`echo` + `$(func)`** is how you return data. Send debug output to `>&2` to avoid polluting the captured value.
>
> **Named output variables** (or nameref `-n`) return multiple values without subshell overhead.
>
> **Subshell trap**: `$(func)` cannot modify caller variables. Call functions directly for side effects.
>
> **`export -f func`** makes functions available to child processes.

| Return strategy | How | Use case |
|----------------|-----|----------|
| Return code | `return 0` or `return 1` | Pass/fail checks |
| stdout capture | `echo value` + `$(func)` | Single return value |
| Named variable | Write to `GLOBAL_VAR` | Multiple values |
| Nameref | `local -n ref="$1"` | Clean multiple values (Bash 4.3+) |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Functions](./functions.md) &nbsp;|&nbsp; **Next:** [User Input →](../05_input_output/user_input.md)

**Related Topics:** [Functions](./functions.md) · [Exit Codes](../06_error_handling/exit_codes.md) · [Variables](../02_variables_and_data/variables.md)
