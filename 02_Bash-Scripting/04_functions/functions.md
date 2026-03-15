# Functions in Bash

## The Department Analogy

A large company does not have one person doing everything. It has departments: the accounting department handles invoices, the HR department handles hiring, the IT department handles infrastructure. When a manager needs payroll run, they call the accounting department — they do not care how the department does it internally. They just call, give the inputs, and receive the result.

Bash functions work the same way. You define a named block of code (a department), give it a clear job, and call it by name whenever you need that job done. The rest of your script does not need to know the internal details.

```
  Without functions:          With functions:
  +----------------------+    +---------------------------+
  | # check server 1     |    | check_server() {          |
  | ping server1         |    |   ping "$1" && echo "OK"  |
  | if failed: alert     |    | }                         |
  |                      |    |                           |
  | # check server 2     |    | check_server web01        |
  | ping server2         |    | check_server web02        |
  | if failed: alert     |    | check_server db01         |
  |                      |    +---------------------------+
  | # check server 3...  |    Write once, reuse anywhere
  +----------------------+
```

---

## Defining Functions

There are two syntaxes — both are valid:

```bash
# Syntax 1: function keyword (more readable)
function greet() {
    echo "Hello, $1!"
}

# Syntax 2: no function keyword (POSIX-compatible)
greet() {
    echo "Hello, $1!"
}
```

Use either style consistently. The `function` keyword makes functions easier to spot when grepping through large scripts.

**Important: Functions must be defined before they are called.**

```bash
#!/usr/bin/env bash

# WRONG: calling before defining
deploy_app       # Error: command not found

deploy_app() {
    echo "Deploying..."
}

# CORRECT: define first, then call
deploy_app() {
    echo "Deploying..."
}

deploy_app       # Works
```

---

## Calling Functions

A function call looks exactly like a command:

```bash
function say_hello() {
    echo "Hello, World!"
}

# Call it
say_hello          # Hello, World!
say_hello          # Call it again — as many times as you want
```

---

## Passing Arguments to Functions

Inside a function, arguments are accessed with `$1`, `$2`, etc. — just like script arguments:

```bash
function deploy() {
    local app_name="$1"
    local environment="$2"
    local version="${3:-latest}"    # default value if not provided

    echo "Deploying $app_name version $version to $environment"
}

# Call with arguments (space-separated, like a command)
deploy "myapp" "production" "v2.1.0"
deploy "api-service" "staging"          # version defaults to "latest"
```

```
  Function definition:      Function call:
  deploy() {                deploy "myapp" "prod" "v2"
    $1 = first arg    <---  "myapp"
    $2 = second arg   <---  "prod"
    $3 = third arg    <---  "v2"
  }
```

Special variables inside functions:
- `$1`, `$2`, `$3` ... — function arguments (NOT the script arguments)
- `$#` — number of arguments passed to this function
- `$@` — all arguments to this function
- `$0` — still the script name (not the function name)

---

## `local` Variables

Variables declared inside a function without `local` are **global** — they leak out and can accidentally overwrite variables in the main script.

Always use `local` for variables inside functions:

```bash
#!/usr/bin/env bash

name="global-script-name"

my_function() {
    local name="local-function-name"    # does NOT affect the global $name
    echo "Inside function: $name"
}

echo "Before call: $name"   # global-script-name
my_function                  # Inside function: local-function-name
echo "After call: $name"    # global-script-name  (unchanged)
```

Without `local`:
```bash
my_function() {
    name="I OVERWROTE THE GLOBAL"    # BAD! No local keyword
    echo "Inside function: $name"
}

echo "Before: $name"    # global-script-name
my_function
echo "After: $name"     # I OVERWROTE THE GLOBAL  (bug!)
```

---

## Return Values

### Method 1: Return codes (`return`)

`return` sets the exit status (`$?`) of the function. This is for success/failure, not for returning data:

```bash
function is_service_running() {
    local service="$1"
    systemctl is-active --quiet "$service"
    return $?    # 0 = running, non-zero = not running
}

if is_service_running "nginx"; then
    echo "nginx is running"
else
    echo "nginx is NOT running"
fi
```

`return 0` = success (true in `if` context)
`return 1` = failure (false in `if` context)

### Method 2: Print and capture (`echo` + command substitution)

For returning actual data, print it and capture with `$(...)`:

```bash
function get_current_version() {
    local app_name="$1"
    # Read version from a file or API
    cat "/opt/${app_name}/version.txt"
}

# Capture the output
current_version=$(get_current_version "myapp")
echo "Current version: $current_version"
```

---

## A Complete Function Library

Here is a real-world pattern: a script with a library of reusable logging and utility functions:

```bash
#!/usr/bin/env bash
# deploy.sh — deployment script with utility functions

# ============================================================
# LOGGING FUNCTIONS
# ============================================================

function log_info()    { echo "[$(date '+%H:%M:%S')] [INFO]  $*"; }
function log_warn()    { echo "[$(date '+%H:%M:%S')] [WARN]  $*" >&2; }
function log_error()   { echo "[$(date '+%H:%M:%S')] [ERROR] $*" >&2; }
function log_success() { echo "[$(date '+%H:%M:%S')] [OK]    $*"; }

# ============================================================
# UTILITY FUNCTIONS
# ============================================================

function require_command() {
    local cmd="$1"
    if ! command -v "$cmd" &>/dev/null; then
        log_error "Required command not found: $cmd"
        exit 1
    fi
    log_info "Found: $cmd at $(command -v "$cmd")"
}

function check_env_var() {
    local var_name="$1"
    if [[ -z "${!var_name}" ]]; then
        log_error "Required environment variable not set: $var_name"
        exit 1
    fi
}

function confirm_action() {
    local message="${1:-Are you sure?}"
    read -rp "$message [y/N] " response
    [[ "$response" =~ ^[Yy]$ ]]
}

# ============================================================
# DEPLOYMENT FUNCTIONS
# ============================================================

function pull_latest_image() {
    local image="$1"
    log_info "Pulling Docker image: $image"
    if docker pull "$image"; then
        log_success "Image pulled: $image"
    else
        log_error "Failed to pull image: $image"
        return 1
    fi
}

function restart_service() {
    local service="$1"
    log_info "Restarting service: $service"
    systemctl restart "$service"
    sleep 3
    if systemctl is-active --quiet "$service"; then
        log_success "Service restarted: $service"
    else
        log_error "Service failed to start: $service"
        return 1
    fi
}

function health_check() {
    local url="$1"
    local max_attempts="${2:-10}"
    local attempt=1

    log_info "Health checking: $url"
    while [[ $attempt -le $max_attempts ]]; do
        if curl -sf "$url" &>/dev/null; then
            log_success "Health check passed after $attempt attempt(s)"
            return 0
        fi
        log_info "Waiting... attempt $attempt/$max_attempts"
        (( attempt++ ))
        sleep 5
    done

    log_error "Health check failed after $max_attempts attempts"
    return 1
}

# ============================================================
# MAIN
# ============================================================

main() {
    local environment="${1:-staging}"
    local version="${2:-latest}"

    log_info "Starting deployment: env=$environment version=$version"

    # Pre-flight checks
    require_command docker
    require_command systemctl
    check_env_var "DOCKER_REGISTRY"
    check_env_var "APP_NAME"

    # Confirm if production
    if [[ "$environment" == "production" ]]; then
        if ! confirm_action "Deploy $version to PRODUCTION?"; then
            log_info "Deployment cancelled."
            exit 0
        fi
    fi

    # Deploy
    pull_latest_image "${DOCKER_REGISTRY}/${APP_NAME}:${version}" || exit 1
    restart_service "${APP_NAME}" || exit 1
    health_check "http://localhost:8080/health" || exit 1

    log_success "Deployment complete: $APP_NAME $version -> $environment"
}

main "$@"
```

---

## Summary

> **Functions** are named, reusable blocks of code. Define with `funcname() { ... }` or `function funcname() { ... }`.
>
> **Always `local`** your variables inside functions to prevent polluting the global scope.
>
> **Arguments** are `$1`, `$2`, `$#`, `$@` — same as script arguments but scoped to the function.
>
> **Return codes** (`return 0/1`) signal success/failure. Use in `if` statements.
>
> **Returning data**: use `echo` inside the function and capture with `result=$(function_name)`.
>
> A good script organizes all logic into functions and calls them from a `main()` function.

| Concept | Syntax |
|---------|--------|
| Define function | `funcname() { ... }` |
| Call function | `funcname arg1 arg2` |
| Function argument | `$1`, `$2`, `$@` |
| Local variable | `local varname="value"` |
| Return code | `return 0` or `return 1` |
| Return data | `echo "$result"` + `$(funcname)` |
| Check success | `if funcname; then` |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Case Statements](../03_control_flow/case_statements.md) &nbsp;|&nbsp; **Next:** [Scope and Return →](./scope_and_return.md)

**Related Topics:** [Scope and Return](./scope_and_return.md) · [Exit Codes](../06_error_handling/exit_codes.md) · [Deployment Scripts](../08_real_world_scripts/deployment_scripts.md)
