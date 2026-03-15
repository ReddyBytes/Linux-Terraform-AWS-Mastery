# Arrays in Bash

## The Shopping List Analogy

A single variable is like a sticky note — it holds one piece of information. But what if you need to track a list of 10 server hostnames? You would not create 10 separate sticky notes (`SERVER1`, `SERVER2`, `SERVER3`...). You would use a shopping list — one organized structure that holds multiple items, each accessible by its position.

Bash arrays are that shopping list. One variable name, many values.

```
  Single variable            Indexed Array
  +----------------+         +-----------------------------+
  | SERVER="web01" |         | SERVERS=("web01" "web02"   |
  |                |         |          "web03" "db01")    |
  | Holds one item |         |                             |
  +----------------+         | Index:  0      1      2   3 |
                             | Value: web01 web02 web03 db01|
                             +-----------------------------+
```

---

## Indexed Arrays

An indexed array stores values at numbered positions starting at 0.

### Creating Arrays

```bash
# Method 1: Assign all at once
servers=("web01" "web02" "db01" "cache01")

# Method 2: Assign element by element
servers[0]="web01"
servers[1]="web02"
servers[2]="db01"

# Method 3: From command output
log_files=( $(ls /var/log/*.log) )

# Method 4: Capture command output into array (safer — handles spaces)
mapfile -t log_files < <(ls /var/log/*.log)
```

### Accessing Elements

```bash
servers=("web01" "web02" "db01" "cache01")

echo ${servers[0]}    # web01   (first element)
echo ${servers[2]}    # db01    (third element)
echo ${servers[-1]}   # cache01 (last element, Bash 4+)

# All elements
echo ${servers[@]}    # web01 web02 db01 cache01
echo ${servers[*]}    # same, but different quoting behavior (see below)

# Number of elements
echo ${#servers[@]}   # 4

# All indices
echo ${!servers[@]}   # 0 1 2 3
```

### Quoting: `"${arr[@]}"` vs `"${arr[*]}"`

This distinction matters when array elements contain spaces:

```bash
files=("report 2026.pdf" "summary.txt" "data export.csv")

# "${arr[@]}" — each element is a separate quoted word (CORRECT)
for f in "${files[@]}"; do
    echo "Processing: $f"
done
# Output:
# Processing: report 2026.pdf
# Processing: summary.txt
# Processing: data export.csv

# "${arr[*]}" — all elements joined into ONE string (usually wrong)
for f in "${files[*]}"; do
    echo "Processing: $f"
done
# Output (one giant string):
# Processing: report 2026.pdf summary.txt data export.csv
```

**Rule: Always use `"${array[@]}"` when iterating.**

---

## Modifying Arrays

```bash
servers=("web01" "web02" "db01")

# Add element to end
servers+=("cache01")
echo ${servers[@]}    # web01 web02 db01 cache01

# Add multiple elements
servers+=("backup01" "monitor01")

# Remove an element (leaves a gap in indices)
unset servers[1]
echo ${servers[@]}    # web01 db01 cache01

# Remove entire array
unset servers

# Slice: elements 1 and 2 (offset=1, length=2)
echo ${servers[@]:1:2}   # web02 db01
```

---

## Iterating Over Arrays

```bash
#!/usr/bin/env bash
# Deploy script: check health of all servers

servers=("web01.prod" "web02.prod" "web03.prod" "db01.prod")

echo "Checking ${#servers[@]} servers..."
echo ""

for server in "${servers[@]}"; do
    if ping -c 1 -W 2 "$server" &>/dev/null; then
        echo "  [OK]   $server is reachable"
    else
        echo "  [DOWN] $server is NOT responding"
    fi
done
```

Iterate with index (when you need the position):

```bash
environments=("dev" "staging" "production")

for i in "${!environments[@]}"; do
    echo "Environment $i: ${environments[$i]}"
done
# Output:
# Environment 0: dev
# Environment 1: staging
# Environment 2: production
```

---

## Associative Arrays (Dictionaries)

An associative array uses string keys instead of numbers — like a dictionary or a hash map. This requires Bash 4.0+.

```bash
# Must declare explicitly
declare -A server_ips

server_ips["web01"]="10.0.1.10"
server_ips["web02"]="10.0.1.11"
server_ips["db01"]="10.0.2.20"
server_ips["cache01"]="10.0.3.30"

# Or initialize all at once
declare -A server_ips=(
    ["web01"]="10.0.1.10"
    ["web02"]="10.0.1.11"
    ["db01"]="10.0.2.20"
)
```

### Accessing Associative Arrays

```bash
echo ${server_ips["web01"]}    # 10.0.1.10
echo ${server_ips["db01"]}     # 10.0.2.20

# All values
echo ${server_ips[@]}          # 10.0.1.10 10.0.1.11 10.0.2.20

# All keys
echo ${!server_ips[@]}         # web01 web02 db01

# Number of entries
echo ${#server_ips[@]}         # 3
```

### Iterating Over Associative Arrays

```bash
declare -A config=(
    ["app_port"]="8080"
    ["db_host"]="localhost"
    ["log_level"]="INFO"
    ["max_connections"]="100"
)

echo "Application Configuration:"
for key in "${!config[@]}"; do
    echo "  ${key} = ${config[$key]}"
done
```

---

## Real-World Array Patterns

### Pattern 1: Validate list of required environment variables

```bash
#!/usr/bin/env bash
required_vars=("DB_HOST" "DB_PASSWORD" "API_KEY" "AWS_REGION")
missing=()

for var in "${required_vars[@]}"; do
    if [[ -z "${!var}" ]]; then    # ${!var} = indirect expansion
        missing+=("$var")
    fi
done

if [[ ${#missing[@]} -gt 0 ]]; then
    echo "ERROR: Missing required environment variables:"
    for var in "${missing[@]}"; do
        echo "  - $var"
    done
    exit 1
fi

echo "All required variables are set."
```

### Pattern 2: Deploy to multiple environments

```bash
#!/usr/bin/env bash
declare -A environments=(
    ["dev"]="dev.myapp.com"
    ["staging"]="staging.myapp.com"
    ["prod"]="myapp.com"
)

TARGET="${1:-dev}"

if [[ -z "${environments[$TARGET]}" ]]; then
    echo "Unknown environment: $TARGET"
    echo "Valid environments: ${!environments[@]}"
    exit 1
fi

echo "Deploying to: ${environments[$TARGET]}"
```

### Pattern 3: Build a command argument array

```bash
#!/usr/bin/env bash
# Build docker run args dynamically
docker_args=("run" "--rm" "-d")
docker_args+=("--name" "myapp")
docker_args+=("-p" "8080:80")

[[ -n "$ENV_FILE" ]] && docker_args+=("--env-file" "$ENV_FILE")
[[ -n "$NETWORK" ]] && docker_args+=("--network" "$NETWORK")

docker_args+=("myapp:latest")

echo "Running: docker ${docker_args[*]}"
docker "${docker_args[@]}"
```

---

## Summary

> **Indexed arrays** store values at numeric positions (0-based). Create with `arr=("a" "b" "c")`. Access with `${arr[0]}`.
>
> **Associative arrays** use string keys. Declare with `declare -A`. Create with `arr["key"]="value"`. Access with `${arr["key"]}`.
>
> **Always iterate with `"${arr[@]}"`** (double-quoted, `@`) to preserve elements with spaces.
>
> **`${#arr[@]}`** gives the count. **`${!arr[@]}`** gives all keys/indices.
>
> Arrays are essential for managing lists of servers, files, arguments, and configuration in real scripts.

| Operation | Syntax |
|-----------|--------|
| Create indexed | `arr=("a" "b" "c")` |
| Create associative | `declare -A arr=(["k"]="v")` |
| Access element | `${arr[0]}` or `${arr["key"]}` |
| All elements | `${arr[@]}` |
| All keys | `${!arr[@]}` |
| Count | `${#arr[@]}` |
| Append | `arr+=("new")` |
| Delete | `unset arr[index]` |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Variables](./variables.md) &nbsp;|&nbsp; **Next:** [String Operations →](./string_operations.md)

**Related Topics:** [Loops](../03_control_flow/loops.md) · [Functions](../04_functions/functions.md)
