# User Input in Bash

## The Bank Teller Analogy

A bank teller does not begin a transaction by assuming they know what you want. They ask: "How can I help you today?" They listen for your answer, verify your identity, and only then proceed. If you take too long to respond, the transaction times out.

The `read` command is your script's bank teller. It pauses execution, asks a question, waits for the user to type an answer, and stores it in a variable. You can prompt with a message, hide sensitive input like passwords, and set a timeout for unattended scenarios.

```
  Script execution:
  +--------------+
  |  echo "msg"  |  --> prints to screen
  |  read name   |  --> PAUSES, waits for user to type
  |              |  <-- user types and presses Enter
  |  echo $name  |  --> uses what user typed
  +--------------+
```

---

## Basic `read`

```bash
#!/usr/bin/env bash

# Simple read — waits for input, stores in variable
echo "Enter your name:"
read name
echo "Hello, $name!"

# Read multiple variables at once
echo "Enter first and last name:"
read first_name last_name
echo "Hello, $first_name $last_name!"

# If user types more words than variables, the last variable gets the rest
echo "Enter city and country:"
read city country
# Input: "New York United States"
# city="New"  country="York United States"
```

---

## `read -p`: Inline Prompt

Instead of a separate `echo`, you can include the prompt directly:

```bash
read -p "Enter environment (dev/staging/prod): " environment
echo "Deploying to: $environment"

read -p "Enter your username: " username
read -p "Enter server hostname: " hostname

echo "Will SSH as $username to $hostname"
```

---

## `read -s`: Silent Input (Passwords)

The `-s` flag suppresses echoing — the user types but nothing appears on screen:

```bash
read -p "Database password: " -s db_password
echo ""    # print newline since -s suppresses it

echo "Connecting to database with provided credentials..."
# Never print $db_password to the screen!

# Two-factor: ask twice and compare
read -p "New password: " -s password1
echo ""
read -p "Confirm password: " -s password2
echo ""

if [[ "$password1" != "$password2" ]]; then
    echo "ERROR: Passwords do not match"
    exit 1
fi
echo "Password set successfully"
```

---

## `read -t`: Timeout

In automated or attended scripts, you may not want to wait forever:

```bash
# Timeout after 10 seconds
if read -t 10 -p "Continue? [Y/n]: " response; then
    echo "Got response: $response"
else
    echo ""
    echo "No response — assuming 'yes' (timeout)"
    response="y"
fi

# Practical: auto-proceed in CI but wait in interactive mode
if [[ -t 0 ]]; then
    # stdin is a terminal — ask the user
    read -t 30 -p "Deploy to production? [y/N]: " -r confirm
else
    # stdin is not a terminal (CI pipeline) — auto-continue
    confirm="y"
fi
```

---

## `read -r`: Raw Mode

Always use `-r` unless you specifically want backslash processing:

```bash
# Without -r: backslash is an escape character
read path
# If user types: /home/user\nfiles
# $path becomes: /home/usernfiles  (backslash-n becomes newline)

# With -r: backslash is treated literally
read -r path
# If user types: /home/user\nfiles
# $path becomes: /home/user\nfiles  (preserved exactly)

# Best practice: always use -r
read -rp "Enter file path: " filepath
```

---

## `read -a`: Read Into Array

```bash
read -ra servers <<< "web01 web02 web03 db01"
echo "Servers: ${servers[@]}"
echo "Count: ${#servers[@]}"

# Or interactively
read -rp "Enter server names (space-separated): " -a server_list
for server in "${server_list[@]}"; do
    echo "Pinging: $server"
done
```

---

## Reading from Arguments vs `read`

Scripts can receive input in two ways:

```bash
#!/usr/bin/env bash
# Method 1: Command-line arguments (non-interactive)
#   ./deploy.sh production v2.1.0

ENVIRONMENT="${1:-}"
VERSION="${2:-}"

# Method 2: Interactive prompts (when args not provided)
if [[ -z "$ENVIRONMENT" ]]; then
    read -rp "Environment (dev/staging/prod): " ENVIRONMENT
fi

if [[ -z "$VERSION" ]]; then
    read -rp "Version to deploy: " VERSION
fi

echo "Deploying $VERSION to $ENVIRONMENT"
```

This pattern lets your script work both interactively and in automated pipelines.

---

## Input Validation

Never trust user input. Always validate:

```bash
#!/usr/bin/env bash

# ---- Validate non-empty input ----
while true; do
    read -rp "Enter environment: " env
    if [[ -n "$env" ]]; then
        break
    fi
    echo "ERROR: Environment cannot be empty"
done

# ---- Validate against allowed values ----
valid_envs=("dev" "staging" "production")

read -rp "Environment: " env
valid=false
for valid_env in "${valid_envs[@]}"; do
    if [[ "$env" == "$valid_env" ]]; then
        valid=true
        break
    fi
done

if [[ "$valid" == false ]]; then
    echo "ERROR: '$env' is not a valid environment"
    exit 1
fi

# ---- Validate format with regex ----
read -rp "Enter version (e.g. v2.1.0): " version
if [[ ! "$version" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "ERROR: Version must match format v1.2.3"
    exit 1
fi

# ---- Validate numeric input ----
read -rp "Enter port number: " port
if [[ ! "$port" =~ ^[0-9]+$ ]] || (( port < 1 || port > 65535 )); then
    echo "ERROR: Invalid port number"
    exit 1
fi

echo "Valid input received"
```

---

## Real-World Example: Interactive Deployment Wizard

```bash
#!/usr/bin/env bash
# deploy_wizard.sh — guided deployment with input validation

set -euo pipefail

function prompt_with_default() {
    local prompt="$1"
    local default="$2"
    local result
    read -rp "$prompt [$default]: " result
    echo "${result:-$default}"
}

function prompt_yes_no() {
    local prompt="$1"
    local default="${2:-n}"
    local response
    while true; do
        read -rp "$prompt [y/n] (default: $default): " response
        response="${response:-$default}"
        case "${response,,}" in
            y|yes) return 0 ;;
            n|no)  return 1 ;;
            *) echo "Please answer y or n" ;;
        esac
    done
}

# ---- Gather deployment parameters ----
echo "============================================"
echo "       DEPLOYMENT WIZARD"
echo "============================================"
echo ""

ENVIRONMENT=$(prompt_with_default "Environment" "staging")
VERSION=$(prompt_with_default "Version" "latest")
REGION=$(prompt_with_default "AWS Region" "us-east-1")

read -rp "Database password: " -s DB_PASSWORD
echo ""

if [[ -z "$DB_PASSWORD" ]]; then
    echo "ERROR: Database password cannot be empty"
    exit 1
fi

echo ""
echo "Summary:"
echo "  Environment: $ENVIRONMENT"
echo "  Version:     $VERSION"
echo "  Region:      $REGION"
echo ""

if ! prompt_yes_no "Proceed with deployment?"; then
    echo "Deployment cancelled."
    exit 0
fi

echo "Starting deployment..."
# ... rest of deployment logic
```

---

## Summary

> **`read var`** — waits for user input, stores in `$var`.
>
> **`read -p "prompt" var`** — includes an inline prompt message.
>
> **`read -s`** — silent mode for passwords; nothing echoes to screen.
>
> **`read -t seconds`** — timeout; if no input, `read` returns non-zero.
>
> **`read -r`** — raw mode; treats backslashes literally. Use this by default.
>
> **`read -a array`** — splits input into an array.
>
> **Always validate input**: check for empty values, allowed values, and correct format.

| Flag | Effect | Use case |
|------|--------|----------|
| `-p "text"` | Inline prompt | Interactive questions |
| `-s` | Silent (no echo) | Passwords, secrets |
| `-t N` | Timeout after N seconds | Auto-continue in CI |
| `-r` | Raw (no backslash escape) | File paths, general input |
| `-a` | Read into array | Space-separated lists |
| `-n N` | Read exactly N chars | Single key press |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Scope and Return](../04_functions/scope_and_return.md) &nbsp;|&nbsp; **Next:** [File Operations →](./file_operations.md)

**Related Topics:** [Conditionals](../03_control_flow/conditionals.md) · [Case Statements](../03_control_flow/case_statements.md) · [Variables](../02_variables_and_data/variables.md)
