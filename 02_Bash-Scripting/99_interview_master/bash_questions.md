# Bash Interview Questions

## How to Use This Section

These questions are organized by difficulty. Work through them in order — each level builds on the previous. For each question, try to answer it yourself before reading the answer. For coding questions, actually write the script.

Interview panels for DevOps and SRE roles expect you to be able to: explain concepts clearly, write working code on a whiteboard, debug a broken script, and discuss real-world scenarios.

---

## Beginner Level

### Q1: What is a shell script and why would you use one?

**Answer:** A shell script is a plain text file containing a sequence of shell commands that the system executes in order. You use shell scripts to automate repetitive tasks, ensure consistency (the same steps every time), and document procedures as executable code.

Real-world examples: deployment scripts, backup automation, server setup scripts, monitoring checks, log rotation.

---

### Q2: What is the shebang line and what does it do?

**Answer:** The shebang is the first line of a script: `#!/bin/bash` or `#!/usr/bin/env bash`. It tells the operating system which interpreter to use to execute the rest of the file. Without it, the OS does not know how to run the script as a program.

- `#!/bin/bash` — hardcoded path, reliable on known systems
- `#!/usr/bin/env bash` — searches PATH for bash, more portable

---

### Q3: What is the difference between `>` and `>>`?

**Answer:**
- `>` redirects output to a file and **overwrites** it (creates the file if it does not exist)
- `>>` redirects output to a file and **appends** to it (creates the file if it does not exist)

```bash
echo "first" > file.txt    # file contains: first
echo "second" > file.txt   # file now contains: second (overwritten)
echo "third" >> file.txt   # file now contains: second\nthird (appended)
```

---

### Q4: What does `$?` mean?

**Answer:** `$?` is a special variable that holds the exit code of the most recently executed command. Exit code `0` means success. Any non-zero value means failure. You must read `$?` immediately after the command of interest, because the next command will overwrite it.

```bash
grep "pattern" file.txt
if [[ $? -eq 0 ]]; then
    echo "Found"
fi
```

---

### Q5: What is the difference between single quotes and double quotes in Bash?

**Answer:**
- **Single quotes `'...'`** — everything is literal. No variable expansion, no command substitution.
- **Double quotes `"..."`** — variables (`$var`) and command substitution (`$(cmd)`) are expanded. Spaces and most special characters are protected.

```bash
name="World"
echo 'Hello $name'    # prints: Hello $name  (literal)
echo "Hello $name"    # prints: Hello World  (expanded)
```

---

### Q6: How do you make a script executable, and what are the two ways to run it?

**Answer:**

```bash
# Make executable
chmod +x myscript.sh

# Method 1: Direct execution (requires +x)
./myscript.sh

# Method 2: Explicit interpreter (no +x needed, ignores shebang)
bash myscript.sh
```

---

### Q7: What is the purpose of `set -e` and `set -u`?

**Answer:**
- `set -e` — exit the script immediately if any command returns a non-zero exit code (error). Prevents continuing in a broken state.
- `set -u` — exit the script if you reference a variable that has not been set. Catches typos and missing environment variables.

Most production scripts start with `set -euo pipefail` ("strict mode").

---

### Q8: How do you check if a file exists in a script?

**Answer:**

```bash
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Config file exists"
fi

# Common file tests:
# -f  regular file
# -d  directory
# -e  exists (any type)
# -r  readable
# -w  writable
# -x  executable
# -s  non-empty
```

---

## Intermediate Level

### Q9: What is the difference between `[ ]` and `[[ ]]`?

**Answer:** `[[ ]]` is the modern Bash-specific form. It is safer and more powerful:

| Feature | `[ ]` | `[[ ]]` |
|---------|-------|---------|
| Word splitting of variables | Yes (unsafe) | No (safe) |
| Regex matching | No | Yes (`=~`) |
| Glob patterns | No | Yes (`==`) |
| `&&` and `||` inside | No | Yes |
| POSIX portable | Yes | Bash only |

Always use `[[ ]]` in Bash scripts.

---

### Q10: What is the difference between `$@` and `$*`?

**Answer:** Both expand to all positional parameters, but they differ when quoted:

- `"$@"` — each argument is separately double-quoted. Preserves arguments with spaces. Use this when iterating.
- `"$*"` — all arguments joined into one string separated by `$IFS`. Loses individual argument boundaries.

```bash
args=("hello world" "foo" "bar")

# With "$@": three separate arguments
for a in "${args[@]}"; do echo "Arg: $a"; done
# Arg: hello world
# Arg: foo
# Arg: bar

# With "$*": one string
echo "${args[*]}"
# hello world foo bar
```

---

### Q11: How do you declare and use associative arrays (dictionaries)?

**Answer:**

```bash
# Requires Bash 4+
declare -A config

config["db_host"]="localhost"
config["db_port"]="5432"
config["app_port"]="8080"

# Access value
echo "${config["db_host"]}"

# Iterate
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Check if key exists
if [[ -n "${config["db_host"]+_}" ]]; then
    echo "db_host is set"
fi
```

---

### Q12: Explain the difference between running a script with `./script.sh` vs `source script.sh`.

**Answer:**

- `./script.sh` — creates a **subshell** (child process). The script runs in a separate environment. Variable changes made by the script do NOT affect the calling shell. When the script exits, the child process ends.

- `source script.sh` (or `. script.sh`) — runs in the **current shell**. Variable changes DO persist after the script finishes. Used for loading configuration files, function libraries, and `.bashrc`.

```bash
# test.sh: export MY_VAR="hello"

./test.sh       ; echo $MY_VAR   # (empty — subshell)
source test.sh  ; echo $MY_VAR   # hello — current shell
```

---

### Q13: Write a function that takes a URL and returns the HTTP status code.

**Answer:**

```bash
get_http_status() {
    local url="$1"
    curl -so /dev/null -w "%{http_code}" --connect-timeout 5 "$url" 2>/dev/null
}

status=$(get_http_status "https://myapp.com/health")
if [[ "$status" == "200" ]]; then
    echo "Service is healthy"
else
    echo "Service returned: $status"
fi
```

---

### Q14: What does `2>&1` mean, and when would you use it?

**Answer:** `2>&1` redirects **file descriptor 2** (stderr) to wherever **file descriptor 1** (stdout) currently points. This merges error output with standard output.

```bash
# Without 2>&1: errors go to screen, stdout goes to file
command > output.log

# With 2>&1: both stdout and stderr go to the file
command > output.log 2>&1

# Modern shorthand
command &> output.log

# In a pipeline (capture both)
command 2>&1 | grep "ERROR"
```

---

### Q15: How do you write a retry loop in Bash?

**Answer:**

```bash
retry() {
    local max_attempts="$1"
    shift
    local attempt=1
    local delay=2

    while [[ $attempt -le $max_attempts ]]; do
        if "$@"; then
            return 0
        fi
        echo "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
        sleep $delay
        (( attempt++ ))
        (( delay *= 2 ))   # exponential backoff
    done

    echo "All $max_attempts attempts failed"
    return 1
}

# Usage
retry 5 curl -sf "https://api.service.com/endpoint"
```

---

### Q16: What does `IFS` do and when would you change it?

**Answer:** `IFS` (Internal Field Separator) controls how Bash splits strings into words. Default is space, tab, and newline.

```bash
# Split a CSV line by comma
line="alice,30,engineering"
IFS=',' read -ra fields <<< "$line"
echo "${fields[0]}"   # alice
echo "${fields[1]}"   # 30

# In a loop: temporarily change IFS
old_ifs=$IFS
IFS=$'\n'
for file in $(find /var/log -name "*.log"); do
    echo "$file"
done
IFS=$old_ifs

# In while read: IFS= prevents stripping leading/trailing spaces
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

---

## Advanced Level

### Q17: Explain signal trapping. Write a script that cleans up temp files even if interrupted.

**Answer:**

The `trap` command registers a handler for signals. `EXIT` fires when the script ends for any reason. `INT` catches Ctrl+C.

```bash
#!/usr/bin/env bash
TEMP_DIR=$(mktemp -d)
LOCK_FILE="/var/run/myscript.lock"

cleanup() {
    echo "Cleaning up..."
    rm -rf "$TEMP_DIR"
    rm -f "$LOCK_FILE"
}

trap cleanup EXIT          # runs on any exit
trap 'echo "Interrupted"; exit 130' INT TERM

# Create lock
echo $$ > "$LOCK_FILE"

# Do work in temp dir
cd "$TEMP_DIR"
echo "Working in: $TEMP_DIR"
sleep 100    # cleanup runs if you press Ctrl+C here
```

---

### Q18: What is process substitution and how is it different from a pipe?

**Answer:** Process substitution `<(command)` makes the output of a command appear as a file. This is different from a pipe because:

- A **pipe** runs the receiving command in a subshell — variable changes are lost after the loop
- **Process substitution** with `while ... done < <(command)` runs the while loop in the current shell — variables persist

```bash
# Pipe — count is lost (subshell problem)
count=0
cat file | while read line; do (( count++ )); done
echo $count   # 0!

# Process substitution — count persists
count=0
while read line; do (( count++ )); done < <(cat file)
echo $count   # correct value

# Also useful for diff
diff <(sort file1.txt) <(sort file2.txt)
```

---

### Q19: How would you find and kill all processes matching a pattern?

**Answer:**

```bash
# Method 1: pkill (most convenient)
pkill -f "python app.py"

# Method 2: pgrep + kill (more control)
pids=$(pgrep -f "gunicorn")
if [[ -n "$pids" ]]; then
    echo "Killing PIDs: $pids"
    kill $pids
fi

# Method 3: with graceful shutdown + force kill
pkill -SIGTERM -f "myapp" && sleep 5
pkill -SIGKILL -f "myapp" 2>/dev/null || true

# Method 4: kill all processes matching and wait
pids=( $(pgrep -f "myapp") )
for pid in "${pids[@]}"; do
    kill -TERM "$pid"
done
# Wait for graceful shutdown
for pid in "${pids[@]}"; do
    wait "$pid" 2>/dev/null || true
done
```

---

### Q20: Write a script that parses command-line flags (`--verbose`, `--env staging`, etc.).

**Answer:**

```bash
#!/usr/bin/env bash
VERBOSE=false
ENVIRONMENT="staging"
VERSION=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -e|--env|--environment)
            ENVIRONMENT="$2"
            shift 2
            ;;
        --version)
            VERSION="$2"
            shift 2
            ;;
        -h|--help)
            echo "Usage: $0 [-v] [-e ENV] [--version VER]"
            exit 0
            ;;
        --)
            shift
            break
            ;;
        -*)
            echo "Unknown option: $1"
            exit 1
            ;;
        *)
            break
            ;;
    esac
done

[[ "$VERBOSE" == true ]] && echo "Verbose mode enabled"
echo "Environment: $ENVIRONMENT, Version: ${VERSION:-not set}"
```

---

### Q21: Scenario question — A cron job is failing silently. How would you debug it?

**Answer:**

1. **Check if cron is running**: `systemctl status cron`
2. **Check cron logs**: `grep CRON /var/log/syslog` or `journalctl -u cron`
3. **Capture cron output**: Edit crontab to log output:
   ```
   * * * * * /opt/scripts/job.sh >> /var/log/job.log 2>&1
   ```
4. **Simulate cron environment**: Run the script with cron's minimal environment:
   ```bash
   env -i HOME=/root SHELL=/bin/bash PATH=/usr/bin:/bin /opt/scripts/job.sh
   ```
5. **Add debugging to the script**: Add `set -x` and log to a file at the top
6. **Check permissions**: Confirm the cron user can read and execute the script
7. **Check for PATH issues**: Use full paths for all commands in the script

---

### Q22: What is the difference between `exit` and `return` in Bash?

**Answer:**
- `exit [N]` — terminates the **entire script** (or current subshell) with exit code N
- `return [N]` — exits only the **current function** with return code N (which becomes `$?` at the call site)

```bash
check_service() {
    if systemctl is-active nginx &>/dev/null; then
        return 0   # function success — script continues
    else
        return 1   # function failure — caller decides what to do
    fi
}

if check_service; then
    echo "Nginx is running"
else
    echo "Nginx is down — exiting script"
    exit 1        # terminates the entire script
fi
```

---

### Q23: What does `set -o pipefail` do? Give an example of when it matters.

**Answer:** Without `pipefail`, a pipeline's exit code is the exit code of the **last** command. An error early in the pipeline is hidden.

```bash
# Without pipefail:
cat nonexistent_file.txt | grep "pattern" | wc -l
# cat fails (exit 1), grep exits 0, wc exits 0
# Pipeline exit code = 0 (looks like success!)
echo $?   # 0

# With pipefail:
set -o pipefail
cat nonexistent_file.txt | grep "pattern" | wc -l
# Pipeline exit code = non-zero (cat's failure is propagated)
echo $?   # non-zero
```

This matters critically in deployment scripts where you do not want to continue if any stage fails.

---

## Quick Reference: Common Script Patterns

```bash
# Strict mode
set -euo pipefail

# Get script directory
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Default value for variable
VAR="${VAR:-default_value}"

# Required variable check
: "${REQUIRED_VAR:?ERROR: REQUIRED_VAR must be set}"

# Check if command exists
command -v docker &>/dev/null || { echo "docker not found"; exit 1; }

# Read file line by line (safe for spaces)
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Capture stderr separately
output=$(command 2>/tmp/errors)

# Retry with backoff
for i in 1 2 3 4 5; do
    command && break
    sleep $(( 2 ** i ))
done

# Cleanup on exit
TEMP_FILE=$(mktemp)
trap 'rm -f "$TEMP_FILE"' EXIT
```

---

## Summary

> **Beginner**: Understand the shebang, exit codes, quoting, file tests, and `set -e/-u`.
>
> **Intermediate**: Know arrays, `[[ ]]` vs `[ ]`, functions with `local`, `source` vs `./`, `2>&1`, and retry patterns.
>
> **Advanced**: Explain signal trapping, process substitution, `pipefail`, argument parsing, and be ready for scenario-based debugging questions.
>
> **For every interview**: Practice writing scripts by hand. Read the question carefully — interviewers often test whether you add error handling and use best practices, not just whether the code works in the happy path.

| Topic | Key concept to know |
|-------|---------------------|
| Shebang | `#!/usr/bin/env bash` vs `#!/bin/bash` |
| Exit codes | `$?`, `exit N`, `set -e` |
| Quoting | Single=literal, Double=expanded, always quote `"$var"` |
| Arrays | Indexed vs associative, `"${arr[@]}"` |
| Functions | `local`, return codes, `echo`+`$()` for data |
| Traps | `trap cleanup EXIT`, `trap handler INT TERM` |
| Pipefail | `set -o pipefail` |
| Process sub | `< <(cmd)` avoids subshell problem |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Deployment Scripts](../08_real_world_scripts/deployment_scripts.md) &nbsp;|&nbsp; **Next:** [What is Terraform →](../../04_Terraform/01_introduction/what_is_terraform.md)

**Related Topics:** [Exit Codes](../06_error_handling/exit_codes.md) · [Traps](../06_error_handling/traps.md) · [Functions](../04_functions/functions.md) · [Loops](../03_control_flow/loops.md)
