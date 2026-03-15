# Loops in Bash

## The Assembly Line Analogy

Picture a car factory assembly line. The same sequence of operations — weld, paint, install engine, inspect — happens to every car that comes down the line. The factory does not write a separate set of instructions for each car. The instructions are written once, and the process repeats for every car automatically.

Bash loops are that assembly line. You write the instructions once and the loop repeats them for every item in a list, for every line in a file, or until a condition changes. This is the foundation of automation.

```
  Without loop:           With loop:
  +-----------------+     +---------------------------+
  | deploy web01    |     | servers=("web01" "web02"  |
  | deploy web02    |     |          "web03")          |
  | deploy web03    |     | for server in "${servers[@]}"; do |
  | (copy-paste x10)|     |     deploy "$server"      |
  +-----------------+     | done                      |
                          +---------------------------+
```

---

## The `for` Loop

### Looping over a list

```bash
# Loop over a literal list
for color in red green blue yellow; do
    echo "Color: $color"
done

# Loop over an array
servers=("web01" "web02" "db01" "cache01")
for server in "${servers[@]}"; do
    echo "Checking: $server"
    ping -c 1 "$server" &>/dev/null && echo "  UP" || echo "  DOWN"
done

# Loop over command output
for user in $(cat /etc/passwd | cut -d: -f1); do
    echo "User: $user"
done

# Better: use while+read for command output (handles spaces in names)
while IFS= read -r user; do
    echo "User: $user"
done < <(cut -d: -f1 /etc/passwd)
```

### C-style `for` loop

```bash
# Traditional C-style loop
for (( i = 0; i < 5; i++ )); do
    echo "Iteration: $i"
done

# Count down
for (( i = 10; i > 0; i-- )); do
    echo "Countdown: $i"
done
echo "Blast off!"
```

### Using `seq`

```bash
# seq generates number sequences
for i in $(seq 1 5); do
    echo "Step $i of 5"
done

# seq with step value (count by 2)
for i in $(seq 0 2 10); do
    echo "$i"    # 0 2 4 6 8 10
done

# seq with padding (zero-pad to 3 digits)
for i in $(seq -w 1 10); do
    echo "server-${i}"   # server-01, server-02, ..., server-10
done
```

### Looping over files

```bash
# Loop over all .log files in a directory
for logfile in /var/log/*.log; do
    size=$(du -sh "$logfile" | cut -f1)
    echo "$logfile  [$size]"
done

# Recursive glob (Bash 4+ with globstar)
shopt -s globstar
for file in /app/src/**/*.py; do
    echo "Found Python file: $file"
done
```

---

## The `while` Loop

The `while` loop runs as long as a condition is true. It is used when you do not know in advance how many iterations you need.

```bash
# Basic while loop
count=1
while [[ $count -le 5 ]]; do
    echo "Count: $count"
    (( count++ ))
done

# Read file line by line (the idiomatic Bash pattern)
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Loop until command succeeds (retry pattern)
attempt=1
max_attempts=5

while ! curl -sf "https://myapp.com/health" &>/dev/null; do
    if [[ $attempt -ge $max_attempts ]]; then
        echo "ERROR: Service did not start after $max_attempts attempts"
        exit 1
    fi
    echo "Waiting for service... attempt $attempt/$max_attempts"
    (( attempt++ ))
    sleep 10
done
echo "Service is up!"
```

### Infinite loop

```bash
# Monitoring loop — runs forever until killed
while true; do
    cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
    echo "$(date): CPU usage: ${cpu}%"
    sleep 60
done
```

---

## The `until` Loop

`until` is the opposite of `while` — it runs until the condition becomes true (i.e., while it is false):

```bash
# until: run until condition is TRUE (while condition is FALSE)
count=0
until [[ $count -ge 5 ]]; do
    echo "Count: $count"
    (( count++ ))
done

# Wait for a port to open (useful after starting a service)
until nc -z localhost 8080 2>/dev/null; do
    echo "Waiting for port 8080..."
    sleep 2
done
echo "Port 8080 is now open"
```

```
  while condition:          until condition:
  +------------------+      +------------------+
  | Run while TRUE   |      | Run while FALSE  |
  | Stop when FALSE  |      | Stop when TRUE   |
  +------------------+      +------------------+
  Like: "keep going         Like: "keep trying
  as long as healthy"       until success"
```

---

## `break` and `continue`

### `break` — exit the loop immediately

```bash
# Find the first failing server, then stop
servers=("web01" "web02" "web03" "web04")

for server in "${servers[@]}"; do
    if ! ping -c 1 -W 2 "$server" &>/dev/null; then
        echo "ALERT: $server is down — stopping scan"
        break    # exit the for loop entirely
    fi
    echo "$server: OK"
done

# Break with exit code (break from nested loops)
for region in us-east-1 eu-west-1 ap-south-1; do
    for service in ec2 rds s3; do
        if some_check_fails "$region" "$service"; then
            break 2    # break out of BOTH loops (level 2)
        fi
    done
done
```

### `continue` — skip to next iteration

```bash
log_files=("app.log" "app.log.1" "app.log.2.gz" "error.log" "debug.log")

for file in "${log_files[@]}"; do
    # Skip compressed files
    if [[ "$file" == *.gz ]]; then
        echo "Skipping compressed: $file"
        continue    # jump to next iteration
    fi
    echo "Processing: $file"
    grep "ERROR" "$file" >> /tmp/errors.txt
done
```

---

## Real-World Loop Patterns

### Pattern 1: Deploy to multiple servers

```bash
#!/usr/bin/env bash
# deploy_multi.sh — deploy to all servers in parallel

servers=("web01.prod" "web02.prod" "web03.prod")
VERSION="${1:-latest}"
FAILED=()

for server in "${servers[@]}"; do
    echo "Deploying $VERSION to $server..."
    if ssh "ubuntu@$server" "cd /app && docker pull myapp:$VERSION && docker-compose up -d"; then
        echo "  [OK] $server"
    else
        echo "  [FAIL] $server"
        FAILED+=("$server")
    fi
done

if [[ ${#FAILED[@]} -gt 0 ]]; then
    echo ""
    echo "FAILED servers: ${FAILED[*]}"
    exit 1
fi

echo "Deployment complete."
```

### Pattern 2: Retry with exponential backoff

```bash
retry_with_backoff() {
    local max_attempts=5
    local attempt=1
    local delay=1

    while [[ $attempt -le $max_attempts ]]; do
        if "$@"; then
            return 0    # success
        fi
        echo "Attempt $attempt failed. Retrying in ${delay}s..."
        sleep $delay
        (( attempt++ ))
        (( delay *= 2 ))    # double the delay each time: 1, 2, 4, 8, 16
    done

    echo "ERROR: All $max_attempts attempts failed"
    return 1
}

# Usage
retry_with_backoff curl -sf "https://api.myservice.com/deploy" -d "$payload"
```

### Pattern 3: Process CSV file

```bash
#!/usr/bin/env bash
# Process a CSV of server configs

while IFS=',' read -r hostname ip_address role; do
    # Skip header line
    [[ "$hostname" == "hostname" ]] && continue
    echo "Configuring $hostname ($ip_address) as $role"
    # ... do work
done < servers.csv
```

---

## Summary

> **`for item in list`** — iterate over a fixed list (arrays, file globs, command output).
>
> **`for ((i=0; i<n; i++))`** — C-style loop for numeric ranges.
>
> **`while condition`** — loop while condition is true. Use for retry loops, infinite monitors, reading files.
>
> **`until condition`** — loop until condition becomes true. Cleaner than `while ! condition`.
>
> **`break`** — exit the loop entirely. **`break 2`** — exit two levels of nested loops.
>
> **`continue`** — skip current iteration and go to the next.

| Loop type | Best for |
|-----------|----------|
| `for item in list` | Arrays, file lists, fixed sets |
| `for ((i=0; i<n; i++))` | Numeric ranges, indexes |
| `seq` | Number sequences with step/padding |
| `while read -r line` | Reading files line by line |
| `while ! command` | Retry until success |
| `while true` | Infinite monitoring loops |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Conditionals](./conditionals.md) &nbsp;|&nbsp; **Next:** [Case Statements →](./case_statements.md)

**Related Topics:** [Arrays](../02_variables_and_data/arrays.md) · [File Operations](../05_input_output/file_operations.md) · [Automation](../07_automation/cron_jobs.md)
