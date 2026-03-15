# Conditionals in Bash

## The Traffic Light Analogy

Every driver approaching an intersection makes a decision: Is the light red? Stop. Is it yellow? Slow down. Is it green? Go. The driver does not drive the same way regardless of the light — they check a condition and choose a path.

Bash conditionals work exactly the same way. Your script reaches a fork in the road, checks a condition, and follows the appropriate path. Without conditionals, every script would be a straight line that could not react to the real world.

```
  Script execution flow:

  [Start]
     |
     v
  [Check condition]
     |
     +---> TRUE  ---> [Do this block] --+
     |                                  |
     +---> FALSE ---> [Do that block] --+
                                        |
                                        v
                                    [Continue]
```

---

## Basic `if` Statement

```bash
#!/usr/bin/env bash

disk_usage=85   # percentage

if [[ $disk_usage -gt 90 ]]; then
    echo "CRITICAL: Disk almost full!"
fi
```

The structure is:
```
if [[ condition ]]; then
    # commands to run if condition is TRUE
fi
```

---

## `if / else`

```bash
if [[ $disk_usage -gt 90 ]]; then
    echo "CRITICAL: Disk almost full!"
else
    echo "Disk usage is acceptable."
fi
```

---

## `if / elif / else`

```bash
if [[ $disk_usage -gt 90 ]]; then
    echo "CRITICAL: disk at ${disk_usage}% — take action NOW"
elif [[ $disk_usage -gt 80 ]]; then
    echo "WARNING: disk at ${disk_usage}% — monitor closely"
elif [[ $disk_usage -gt 70 ]]; then
    echo "INFO: disk at ${disk_usage}% — getting full"
else
    echo "OK: disk at ${disk_usage}%"
fi
```

---

## `[[ ]]` vs `[ ]` — Use the Modern Form

You will see both in scripts. Here is why `[[ ]]` is the modern, preferred form:

```
  Old form: [ ]              New form: [[ ]]
  +-----------------------+  +--------------------------+
  | POSIX compatible      |  | Bash-specific            |
  | No regex support      |  | Regex with =~            |
  | Word splitting occurs |  | No word splitting        |
  | Quoting is critical   |  | Safer without quotes     |
  | No && || inside       |  | && || work inside        |
  | No pattern matching   |  | Pattern matching with *  |
  +-----------------------+  +--------------------------+

  Use [[ ]] for all new Bash scripts.
  Use [ ] only if you need POSIX sh compatibility.
```

Example showing why `[[ ]]` is safer:

```bash
name=""

# With [ ] — causes syntax error when var is empty!
if [ $name == "admin" ]; then   # expands to: if [ == "admin" ]  ERROR

# With [[ ]] — handles empty variables safely
if [[ $name == "admin" ]]; then   # safe even without quotes
```

---

## Numeric Comparison Operators

```bash
a=10
b=20

[[ $a -eq $b ]]   # equal to              (==)
[[ $a -ne $b ]]   # not equal to          (!=)
[[ $a -lt $b ]]   # less than             (<)
[[ $a -le $b ]]   # less than or equal    (<=)
[[ $a -gt $b ]]   # greater than          (>)
[[ $a -ge $b ]]   # greater than or equal (>=)
```

Or use arithmetic context for more natural syntax:

```bash
if (( a > b )); then
    echo "$a is greater"
fi

if (( a == 0 )); then
    echo "zero"
fi
```

---

## String Comparison Operators

```bash
env="production"

[[ "$env" == "production" ]]    # exact match
[[ "$env" != "development" ]]   # not equal
[[ -z "$env" ]]                 # is empty (zero length)
[[ -n "$env" ]]                 # is not empty (non-zero length)
[[ "$env" < "staging" ]]        # lexicographic comparison
[[ "$env" > "dev" ]]            # lexicographic comparison

# Pattern matching (glob-style, not regex)
[[ "$env" == prod* ]]           # starts with "prod"
[[ "$env" == *tion ]]           # ends with "tion"
[[ "$env" == *oduc* ]]          # contains "oduc"

# Regex matching
[[ "$env" =~ ^prod(uction)?$ ]] # matches "prod" or "production"
```

---

## File Test Operators

These are invaluable in deployment and automation scripts:

```bash
filepath="/etc/nginx/nginx.conf"
dirpath="/var/log/app"

# File existence and type
[[ -e "$filepath" ]]   # exists (file or directory)
[[ -f "$filepath" ]]   # exists and is a regular file
[[ -d "$dirpath" ]]    # exists and is a directory
[[ -L "$filepath" ]]   # exists and is a symlink
[[ -p "$filepath" ]]   # exists and is a named pipe

# Permissions
[[ -r "$filepath" ]]   # readable
[[ -w "$filepath" ]]   # writable
[[ -x "$filepath" ]]   # executable

# File content
[[ -s "$filepath" ]]   # exists and is NOT empty (size > 0)

# Comparison
[[ "file1" -nt "file2" ]]   # file1 is newer than file2
[[ "file1" -ot "file2" ]]   # file1 is older than file2
```

Practical deployment use:

```bash
#!/usr/bin/env bash
CONFIG="/etc/myapp/config.yml"
LOG_DIR="/var/log/myapp"

# Check config exists before starting
if [[ ! -f "$CONFIG" ]]; then
    echo "ERROR: Config file not found: $CONFIG"
    exit 1
fi

# Create log directory if missing
if [[ ! -d "$LOG_DIR" ]]; then
    echo "Creating log directory: $LOG_DIR"
    mkdir -p "$LOG_DIR"
fi

# Check config is readable
if [[ ! -r "$CONFIG" ]]; then
    echo "ERROR: Cannot read config: $CONFIG"
    exit 1
fi

echo "Pre-flight checks passed."
```

---

## Combining Conditions

```bash
# AND: both must be true
if [[ -f "$file" && -r "$file" ]]; then
    echo "File exists and is readable"
fi

# OR: at least one must be true
if [[ "$env" == "staging" || "$env" == "production" ]]; then
    echo "Deploying to a real environment"
fi

# NOT: invert the condition
if [[ ! -d "$dir" ]]; then
    mkdir -p "$dir"
fi

# Complex combination
if [[ -f "$config" && -n "$API_KEY" && "$ENV" != "test" ]]; then
    echo "Ready to deploy"
fi
```

---

## One-liner Shortcuts

```bash
# Run command only if condition is true (AND shortcut)
[[ -d "$LOG_DIR" ]] || mkdir -p "$LOG_DIR"
[[ -n "$USER" ]] && echo "Logged in as $USER"

# Equivalents using if:
# if [[ ! -d "$LOG_DIR" ]]; then mkdir -p "$LOG_DIR"; fi
# if [[ -n "$USER" ]]; then echo "Logged in as $USER"; fi
```

---

## Real-World Example: Pre-Deployment Checks

```bash
#!/usr/bin/env bash
# pre_deploy_check.sh — validate environment before deploying

ENVIRONMENT="${1:-}"
VERSION="${2:-}"

# Validate required arguments
if [[ -z "$ENVIRONMENT" || -z "$VERSION" ]]; then
    echo "Usage: $0 <environment> <version>"
    echo "  environment: dev | staging | production"
    echo "  version:     e.g. v2.1.0"
    exit 1
fi

# Validate environment name
if [[ "$ENVIRONMENT" != "dev" && "$ENVIRONMENT" != "staging" && "$ENVIRONMENT" != "production" ]]; then
    echo "ERROR: Invalid environment '$ENVIRONMENT'"
    exit 1
fi

# Validate version format
if [[ ! "$VERSION" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "ERROR: Invalid version '$VERSION' — must match v1.2.3"
    exit 1
fi

# Check required tools
for tool in docker git aws; do
    if ! command -v "$tool" &>/dev/null; then
        echo "ERROR: Required tool '$tool' not found in PATH"
        exit 1
    fi
done

# Extra checks for production
if [[ "$ENVIRONMENT" == "production" ]]; then
    if [[ -z "$DEPLOY_APPROVAL_TOKEN" ]]; then
        echo "ERROR: DEPLOY_APPROVAL_TOKEN required for production"
        exit 1
    fi
    echo "WARNING: You are about to deploy to PRODUCTION!"
    read -p "Type 'yes' to confirm: " confirm
    if [[ "$confirm" != "yes" ]]; then
        echo "Deployment cancelled."
        exit 0
    fi
fi

echo "All checks passed. Proceeding with deployment..."
```

---

## Summary

> **`if / elif / else / fi`** is the core decision structure. All conditions sit inside `[[ ]]`.
>
> **Always use `[[ ]]`** (double brackets) over `[ ]` for Bash scripts. It is safer and more powerful.
>
> **Numeric operators**: `-eq -ne -lt -le -gt -ge` or `(( ))` with `== != < <= > >=`.
>
> **String operators**: `== != -z -n` plus glob patterns and `=~` for regex.
>
> **File operators**: `-f -d -e -r -w -x -s` for file existence, type, and permissions.
>
> **Combine conditions** with `&&` (AND), `||` (OR), `!` (NOT).

| Test | Meaning |
|------|---------|
| `-f file` | Is a regular file |
| `-d dir` | Is a directory |
| `-e path` | Exists (any type) |
| `-r file` | Is readable |
| `-x file` | Is executable |
| `-z "$str"` | String is empty |
| `-n "$str"` | String is not empty |
| `$a -gt $b` | a is greater than b |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← String Operations](../02_variables_and_data/string_operations.md) &nbsp;|&nbsp; **Next:** [Loops →](./loops.md)

**Related Topics:** [Case Statements](./case_statements.md) · [Exit Codes](../06_error_handling/exit_codes.md) · [User Input](../05_input_output/user_input.md)
