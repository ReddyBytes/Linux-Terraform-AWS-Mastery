# Pipes and Redirection

## The Plumbing Analogy

In a building, water flows through pipes from one place to another. Output from the tank flows into the filter, from the filter into the softener, from the softener to the tap. No single component needs to know the full path — each piece just receives input and sends output.

Linux shell pipes work identically. The output of one command flows directly into the input of the next, forming a processing pipeline. Redirection is the valve that controls where output flows — to a file, to the screen, or away entirely.

```
  Pipeline flow:

  [command1] --stdout--> | --stdin--> [command2] --stdout--> | --stdin--> [command3]

  Example:
  cat access.log | grep "ERROR" | awk '{print $1}' | sort | uniq -c | sort -rn
      |                |               |               |       |           |
      read file    filter lines    extract field     sort  dedupe      sort count
```

---

## The Pipe `|`

The pipe connects stdout of one command to stdin of the next:

```bash
# Find all ERROR lines in log, count unique IPs
cat /var/log/nginx/access.log | grep "ERROR" | awk '{print $1}' | sort -u | wc -l

# List running Docker containers sorted by name
docker ps --format "{{.Names}}" | sort

# Show top 10 memory-consuming processes
ps aux | sort -k 4 -rn | head -10

# Find all Python files and count total lines
find /app -name "*.py" | xargs wc -l | tail -1

# Show disk usage sorted by size (largest first)
du -sh /var/* 2>/dev/null | sort -rh | head -10
```

---

## Output Redirection `>` and `>>`

```bash
# > creates or overwrites
echo "Deployment started at $(date)" > /var/log/deploy.log

# >> appends to existing file
echo "Step 1: Image pulled" >> /var/log/deploy.log
echo "Step 2: Container started" >> /var/log/deploy.log

# Overwrite an entire command's output to a file
ls -la /app > /tmp/app_contents.txt

# Append multiple lines using a loop
for server in web01 web02 web03; do
    echo "$(date): Deploying to $server" >> /var/log/deploy.log
done
```

---

## Standard Streams

Every process has three standard streams:

```
  stdin  (fd 0) — input  (keyboard by default)
  stdout (fd 1) — output (screen by default)
  stderr (fd 2) — errors (screen by default)

  +----------+
  |          | <-- stdin (0)
  | command  |
  |          | --> stdout (1) --> screen
  |          | --> stderr (2) --> screen
  +----------+
```

---

## Redirecting stderr `2>`

```bash
# Redirect stderr to a file (keep stdout on screen)
docker pull myimage:latest 2> /tmp/pull_errors.txt

# Redirect stdout to one file, stderr to another
make build > /tmp/build_output.txt 2> /tmp/build_errors.txt

# Redirect stderr to stdout (merge them)
make build 2>&1

# Merge both into one file
make build > /tmp/build.log 2>&1
# Or modern shorthand (Bash 4+):
make build &> /tmp/build.log
```

The order matters with `2>&1`:
```bash
# CORRECT: redirect stdout to file, THEN redirect stderr to where stdout now points
command > file 2>&1

# WRONG: redirects stderr to screen (old stdout), THEN redirects stdout to file
command 2>&1 > file
```

---

## `/dev/null`: The Black Hole

`/dev/null` is a special file that discards anything written to it:

```bash
# Suppress stdout (only show errors)
command > /dev/null

# Suppress stderr (only show output)
command 2> /dev/null

# Suppress ALL output (silent execution)
command &> /dev/null
command > /dev/null 2>&1    # equivalent, older syntax

# Common use: test if command succeeds without showing output
if command -v docker &>/dev/null; then
    echo "Docker is installed"
fi

if ping -c 1 8.8.8.8 &>/dev/null; then
    echo "Internet is reachable"
fi
```

---

## Input Redirection `<`

```bash
# Feed file content as stdin
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# Sort a file
sort < /var/log/app.log > /var/log/app_sorted.log

# Pass file to command that reads stdin
mysql -u root mydb < schema.sql
```

---

## `tee`: Write to File AND Screen

`tee` splits stdout — it writes to a file while still passing data through to stdout:

```bash
# Show output on screen AND save to log
./deploy.sh | tee /var/log/deploy.log

# Append to log
./deploy.sh | tee -a /var/log/deploy.log

# In a pipeline — tee in the middle
cat access.log | tee /tmp/raw_access.log | grep "ERROR" | tee /tmp/errors.log | wc -l

# Include stderr in the tee
./deploy.sh 2>&1 | tee /var/log/deploy.log
```

```
  command  --stdout-->  tee  --stdout--> screen
                         |
                         +--> file (also written here)
```

---

## Here Documents `<< 'EOF'`

A here document lets you write multi-line strings inline in a script:

```bash
# Write a config file inline
cat > /etc/myapp/nginx.conf << 'EOF'
server {
    listen 80;
    server_name myapp.com;
    root /var/www/html;

    location / {
        proxy_pass http://localhost:8080;
    }
}
EOF

# Send an email (if mail is configured)
mail -s "Deployment Complete" ops@company.com << 'EOF'
The deployment to production has completed successfully.

Version: v2.1.0
Environment: production
Time: 2026-03-15 14:30:00 UTC
EOF

# Using variables inside here doc (no single quotes around EOF)
VERSION="v2.1.0"
cat > /tmp/release_notes.txt << EOF
Release Notes for $VERSION
Deployed by: $USER
Date: $(date)
EOF
```

**Quoting rules:**
- `<< 'EOF'` (quoted) — no variable/command expansion inside
- `<< EOF` (unquoted) — variables and `$(...)` are expanded inside

---

## Here Strings `<<<`

A here string feeds a single string as stdin:

```bash
# Feed a string to a command
grep "ERROR" <<< "This is an ERROR message"

# Read from a string into variables
read -r first last <<< "John Smith"
echo "$first $last"    # John Smith

# Parse a string with read
IFS=',' read -r host port db <<< "localhost,5432,mydb"
echo "Host: $host, Port: $port, DB: $db"
```

---

## Process Substitution `<()`

Process substitution lets you use command output as if it were a file:

```bash
# Compare output of two commands (diff needs files)
diff <(sort file1.txt) <(sort file2.txt)

# Read from two commands simultaneously
while IFS= read -r line; do
    echo "Line: $line"
done < <(find /var/log -name "*.log" -newer /tmp/timestamp)

# Pass command output to a command that reads a file
wc -l < <(grep "ERROR" /var/log/app.log)

# Useful for while+read without a subshell:
total=0
while IFS= read -r line; do
    (( total++ ))
done < <(cat /var/log/app.log)
echo "Total: $total"    # Works! No subshell means total persists
```

The difference between `|` and `< <()`:
```bash
# With pipe: while runs in a subshell — count is lost after loop
cat file | while read line; do (( count++ )); done
echo $count    # 0  (count died with subshell)

# With process substitution: while runs in current shell
while read line; do (( count++ )); done < <(cat file)
echo $count    # correct value
```

---

## Real-World Pipeline: Log Processing

```bash
#!/usr/bin/env bash
# daily_log_report.sh — Generate daily error report from application logs

LOG_FILE="/var/log/myapp/app.log"
REPORT="/tmp/daily_report_$(date +%Y%m%d).txt"
TODAY=$(date '+%Y-%m-%d')

{
    echo "Daily Error Report - $TODAY"
    echo "================================"
    echo ""

    echo "Total log entries today:"
    grep "^$TODAY" "$LOG_FILE" | wc -l

    echo ""
    echo "Error breakdown:"
    grep "^$TODAY" "$LOG_FILE" \
        | grep "\[ERROR\]\|\[WARN\]\|\[CRITICAL\]" \
        | awk '{print $3}' \
        | sort | uniq -c | sort -rn

    echo ""
    echo "Top 5 error messages:"
    grep "^$TODAY" "$LOG_FILE" \
        | grep "\[ERROR\]" \
        | sed 's/.*\[ERROR\] //' \
        | sort | uniq -c | sort -rn | head -5

} | tee "$REPORT"

echo ""
echo "Full report saved to: $REPORT"
```

---

## Summary

> **`|`** connects stdout of one command to stdin of the next. Chain as many as needed.
>
> **`>`** redirects stdout to a file (overwrites). **`>>`** appends.
>
> **`2>`** redirects stderr. **`2>&1`** merges stderr into stdout.
>
> **`/dev/null`** discards output. Use `&>/dev/null` to silence a command completely.
>
> **`tee`** splits stdout to screen and file simultaneously. Essential for logging.
>
> **`<< 'EOF'`** is a here document — embed multi-line text in a script.
>
> **`< <(command)`** is process substitution — use command output as a file input without creating a subshell.

| Syntax | Effect |
|--------|--------|
| `cmd1 \| cmd2` | Pipe stdout of cmd1 to stdin of cmd2 |
| `> file` | Redirect stdout to file (overwrite) |
| `>> file` | Redirect stdout to file (append) |
| `2> file` | Redirect stderr to file |
| `2>&1` | Redirect stderr to stdout |
| `&> file` | Redirect both stdout+stderr to file |
| `> /dev/null` | Discard stdout |
| `tee file` | Write to file and stdout |
| `<< 'EOF'` | Here document |
| `< <(cmd)` | Process substitution (file-like input) |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← File Operations](./file_operations.md) &nbsp;|&nbsp; **Next:** [Exit Codes →](../06_error_handling/exit_codes.md)

**Related Topics:** [File Operations](./file_operations.md) · [Loops](../03_control_flow/loops.md) · [System Monitoring](../08_real_world_scripts/system_monitoring.md)
