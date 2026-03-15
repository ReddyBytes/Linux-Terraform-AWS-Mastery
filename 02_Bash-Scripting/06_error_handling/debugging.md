# Debugging Bash Scripts

## The Black Box Recorder Analogy

When an airplane crashes, investigators retrieve the flight data recorder — the black box — which captured every instrument reading, every control input, and every system event during the flight. Without it, there is only the wreckage and guesswork.

`set -x` is the black box for your Bash script. It records every command being executed, every variable being expanded, every decision being made, and prints it to your terminal in real time. When your script fails mysteriously, `set -x` turns "the script just stopped" into "here is exactly which command failed, with these exact values."

```
  Without debug mode:            With set -x (trace mode):
  +--------------------+         +------------------------------------+
  | ./deploy.sh        |         | + VERSION=v2.1.0                   |
  |                    |         | + docker pull myapp:v2.1.0         |
  | ...                |         | ++ docker pull myapp:v2.1.0        |
  | Deployment failed! |         | Error: image not found             |
  |                    |         | + echo "Pull failed"               |
  | (No idea why)      |         | Pull failed                        |
  +--------------------+         | (Exact line, exact values shown)   |
                                  +------------------------------------+
```

---

## `set -x`: Trace Mode

`set -x` prints each command before executing it, with a `+` prefix. Variable expansions are shown with their actual values:

```bash
#!/usr/bin/env bash
set -x    # Enable trace mode

name="world"
echo "Hello, $name!"
```

Output:
```
+ name=world
+ echo 'Hello, world!'
Hello, world!
```

### Enable/disable for a section

```bash
#!/usr/bin/env bash

echo "Normal execution (no trace)"
some_command

set -x    # Start tracing
echo "This section is traced"
problem_command
another_command
set +x    # Stop tracing

echo "Back to normal"
```

### Run a script with trace mode without editing it

```bash
bash -x deploy.sh
bash -x deploy.sh production v2.1.0

# Or set via environment variable
BASHOPTS=xtrace bash deploy.sh
```

---

## `set -e`: Exit on Error

Without `set -e`, errors are silently ignored:

```bash
#!/usr/bin/env bash
# Without set -e:
git pull origin main     # fails: not a git repo
docker build -t myapp .  # still runs, but working directory is wrong
docker push myapp        # still runs, pushing a broken image

# With set -e:
set -e
git pull origin main     # fails and STOPS HERE
docker build -t myapp .  # never reached
```

---

## `set -u`: Catch Undefined Variables

```bash
#!/usr/bin/env bash
set -u

# Without set -u: $TYPO silently expands to empty string
# rm -rf "/${TYPO}data"   ->  rm -rf "/data"  (DISASTER!)

# With set -u:
rm -rf "/${TYPO}data"
# bash: TYPO: unbound variable  (SAVED!)
```

Always provide defaults for variables that might legitimately be unset:

```bash
set -u
# Safe patterns with set -u:
ENVIRONMENT="${ENVIRONMENT:-staging}"
VERSION="${1:-latest}"
DEBUG="${DEBUG:-false}"
```

---

## `set -o pipefail`

Without `pipefail`, a failed command inside a pipe is hidden:

```bash
# grep returns 1 if no match, but cat returns 0, so pipeline = 0
cat nonexistent | grep "pattern"
echo $?    # 0 — looks like success!

set -o pipefail
cat nonexistent | grep "pattern"
echo $?    # non-zero — the failure is visible
```

---

## The Full Debug Header

Use this at the top of every script you are debugging:

```bash
#!/usr/bin/env bash
set -euxo pipefail
# e = exit on error
# u = error on undefined variable
# x = trace mode (print each command)
# o pipefail = fail if any pipe command fails
```

Remove `-x` for production but keep `-euo pipefail`.

---

## `PS4`: Customizing the Trace Prefix

The `+` prefix in `set -x` output can be improved to show the filename and line number:

```bash
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

Output with custom `PS4`:
```
+(deploy.sh:42): main(): docker pull myapp:latest
+(deploy.sh:43): main(): docker-compose up -d
```

Now you know exactly which file, which line number, and which function — invaluable for large scripts.

---

## `bash -n`: Syntax Check Without Running

Check for syntax errors without actually executing the script:

```bash
bash -n deploy.sh
# If no output: syntax is OK
# If errors shown: fix them before running

# Always run this before deploying a script to production:
bash -n "$script" && echo "Syntax OK" || echo "Syntax ERROR"
```

---

## Strategic `echo` Debugging

Simple but effective for quick debugging:

```bash
#!/usr/bin/env bash

DEBUG="${DEBUG:-false}"

debug() {
    if [[ "$DEBUG" == "true" ]]; then
        echo "[DEBUG] $*" >&2
    fi
}

VERSION="${1:-latest}"
debug "Script started with VERSION=$VERSION"

docker pull "myapp:$VERSION"
debug "Docker pull completed with exit code: $?"

# Run with debug output:
# DEBUG=true ./deploy.sh v2.1.0
```

---

## Common Bash Mistakes and Fixes

### Mistake 1: Spaces around `=`

```bash
# WRONG
name = "hello"    # bash thinks 'name' is a command

# CORRECT
name="hello"
```

### Mistake 2: Unquoted variables with spaces

```bash
# WRONG
filename="my report.pdf"
rm $filename        # tries to delete "my" and "report.pdf"

# CORRECT
rm "$filename"      # treats as one argument
```

### Mistake 3: Using `[ ]` instead of `[[ ]]`

```bash
# WRONG — can fail when variable is empty
if [ $name == "admin" ]; then

# CORRECT
if [[ "$name" == "admin" ]]; then
```

### Mistake 4: Forgetting that pipes create subshells

```bash
# WRONG — count is 0 because pipe creates subshell
count=0
cat file | while read line; do (( count++ )); done
echo $count    # 0!

# CORRECT — use process substitution
count=0
while read line; do (( count++ )); done < <(cat file)
echo $count    # correct
```

### Mistake 5: Not checking if command exists

```bash
# WRONG
jq .version config.json    # fails if jq not installed

# CORRECT
if ! command -v jq &>/dev/null; then
    echo "ERROR: jq is required but not installed"
    exit 1
fi
jq .version config.json
```

### Mistake 6: `cd` without error checking

```bash
# WRONG — if cd fails, commands run in wrong directory
cd /app
rm -rf *    # might rm from current dir if /app doesn't exist!

# CORRECT
cd /app || { echo "ERROR: Cannot cd to /app"; exit 1; }
rm -rf *
```

---

## Debugging Tools Reference

```bash
# Show each command as it executes
set -x
bash -x script.sh

# Exit on error
set -e

# Exit on undefined variable
set -u

# Fail pipeline if any stage fails
set -o pipefail

# Syntax check (no execution)
bash -n script.sh

# Check for common mistakes with shellcheck (external tool)
shellcheck script.sh

# Print line number in error messages
export PS4='+(${BASH_SOURCE}:${LINENO}): '

# Show script version info
bash --version
```

### ShellCheck — the static analyzer

Install ShellCheck and run it on your scripts before deploying:

```bash
# Ubuntu/Debian
apt-get install shellcheck

# macOS
brew install shellcheck

# Check a script
shellcheck deploy.sh
shellcheck *.sh
```

ShellCheck catches dozens of common bugs and anti-patterns automatically.

---

## Real-World Debug Wrapper

```bash
#!/usr/bin/env bash
# debug_wrapper.sh — run any script with full debug output and logging

SCRIPT="$1"
shift

LOG_FILE="/tmp/debug_$(basename "$SCRIPT")_$(date +%Y%m%d_%H%M%S).log"

echo "Running: $SCRIPT $*"
echo "Debug log: $LOG_FILE"
echo ""

export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
bash -x "$SCRIPT" "$@" 2>&1 | tee "$LOG_FILE"

echo ""
echo "Exit code: $?"
echo "Full log saved to: $LOG_FILE"
```

Usage:
```bash
./debug_wrapper.sh deploy.sh production v2.1.0
```

---

## Summary

> **`set -x`** prints every command before executing it (trace mode). Essential for debugging.
>
> **`bash -x script.sh`** enables trace mode without editing the script.
>
> **`set -e`** exits immediately when any command fails.
>
> **`set -u`** exits when an undefined variable is used — catches typos.
>
> **`set -o pipefail`** catches failures inside pipes.
>
> **`bash -n script.sh`** syntax-checks without running.
>
> **`PS4`** customizes the trace prefix — add filename and line number for better output.
>
> **ShellCheck** is a static analysis tool that catches common bugs automatically.

| Debug tool | Usage |
|-----------|-------|
| `set -x` | Trace every command |
| `set -e` | Exit on error |
| `set -u` | Exit on undefined var |
| `set -o pipefail` | Fail on pipe error |
| `bash -n script` | Syntax check |
| `bash -x script` | Trace without editing |
| `shellcheck script` | Static analysis |
| Custom `PS4` | Better trace output |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Traps](./traps.md) &nbsp;|&nbsp; **Next:** [Cron Jobs →](../07_automation/cron_jobs.md)

**Related Topics:** [Exit Codes](./exit_codes.md) · [Traps](./traps.md) · [Functions](../04_functions/functions.md)
