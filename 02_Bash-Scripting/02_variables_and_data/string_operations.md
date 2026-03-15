# String Operations in Bash

## The Word Processor Analogy

Think of Bash string manipulation as having a built-in word processor. When you type a document, you can find and replace words, count characters, extract a paragraph, convert everything to uppercase, or trim whitespace from the edges. Bash offers all of these operations — without needing external tools like `awk` or `sed` — directly through parameter expansion syntax.

Learning string operations is what separates a basic scripter from someone who can write efficient, production-quality scripts.

```
  Original string: "  Hello, World!  "
                    ^              ^
                    leading     trailing
                    whitespace  whitespace

  Operations available:
  +--------------+-------------------------+
  | Length       | ${#string}           = 18|
  | Uppercase    | ${string^^}          = "  HELLO, WORLD!  " |
  | Lowercase    | ${string,,}          = "  hello, world!  " |
  | Substring    | ${string:2:5}        = "Hello"             |
  | Replace      | ${string/World/Bash} = "  Hello, Bash!  "  |
  | Strip        | ${string##*,}        = " World!  "         |
  +--------------+--------------------------------------------+
```

---

## String Length

```bash
name="deploy-production-v2"
echo ${#name}          # 21

# Practical: validate input is not empty
if [[ ${#name} -eq 0 ]]; then
    echo "ERROR: name cannot be empty"
    exit 1
fi

# Validate max length
if [[ ${#name} -gt 50 ]]; then
    echo "ERROR: name too long (max 50 chars)"
    exit 1
fi
```

---

## Substrings

```bash
# ${string:offset:length}
full="us-east-1-web-prod-01"

echo ${full:0:9}      # us-east-1   (from position 0, take 9 chars)
echo ${full:10}       # web-prod-01 (from position 10 to end)
echo ${full: -8}      # prod-01     (last 8 chars, note the space)
echo ${full: -8:4}    # prod        (last 8 chars, take 4)
```

Real-world use — extract AWS region from an ARN:

```bash
arn="arn:aws:s3:::my-bucket"
# ARN format: arn:partition:service:region:account:resource

IFS=':' read -ra arn_parts <<< "$arn"
region="${arn_parts[3]}"
service="${arn_parts[2]}"
echo "Service: $service, Region: ${region:-global}"
```

---

## String Replacement

```bash
# ${string/pattern/replacement}  — replace FIRST match
# ${string//pattern/replacement} — replace ALL matches

version="app-v1.2.3-beta"

echo ${version/beta/stable}        # app-v1.2.3-stable (first match)
echo ${version//-/_}               # app_v1.2.3_beta   (all hyphens to underscore)

# Replace at start: ${string/#pattern/replacement}
path="/old/path/to/file"
echo ${path/#\/old/\/new}          # /new/path/to/file

# Replace at end: ${string/%pattern/replacement}
filename="deploy.sh.bak"
echo ${filename/%.bak/}            # deploy.sh  (remove .bak suffix)
```

---

## Removing Prefixes and Suffixes

This is one of the most powerful and frequently used Bash string operations:

```bash
# ${string#pattern}  — remove SHORTEST prefix match
# ${string##pattern} — remove LONGEST prefix match
# ${string%pattern}  — remove SHORTEST suffix match
# ${string%%pattern} — remove LONGEST suffix match

filepath="/var/log/nginx/access.log"

# Get just the filename (remove everything up to last /)
filename="${filepath##*/}"
echo $filename    # access.log

# Get just the directory (remove everything from last / onward)
directory="${filepath%/*}"
echo $directory   # /var/log/nginx

# Remove file extension
basename="${filename%.*}"
echo $basename    # access

# Get file extension
extension="${filename##*.}"
echo $extension   # log
```

These patterns are so useful they deserve their own diagram:

```
  filepath = "/var/log/nginx/access.log"

  ${filepath##*/}   removes  "/var/log/nginx/"   →  "access.log"
  ${filepath%/*}    removes  "/access.log"        →  "/var/log/nginx"
  ${filename%.*}    removes  ".log"               →  "access"
  ${filename##*.}   removes  "access."            →  "log"

  # = remove from LEFT (prefix)
  % = remove from RIGHT (suffix)
  single char = shortest match
  double char = longest match
```

---

## Case Conversion (Bash 4+)

```bash
name="Hello World"

echo ${name^^}     # HELLO WORLD   (all uppercase)
echo ${name,,}     # hello world   (all lowercase)
echo ${name^}      # Hello World   (capitalize first char)
echo ${name,}      # hello World   (lowercase first char)

# Practical: normalize user input
read -p "Environment (dev/staging/prod): " env
env="${env,,}"    # convert to lowercase regardless of what user typed

case "$env" in
    dev|development)   deploy_dev   ;;
    staging|stage)     deploy_stage ;;
    prod|production)   deploy_prod  ;;
    *)                 echo "Unknown environment: $env" ; exit 1 ;;
esac
```

---

## String Tests and Comparisons with `[[ ]]`

The `[[ ]]` construct (double brackets) is the modern Bash way to test strings. It is safer and more capable than the old `[ ]`:

```bash
name="production"

# Equality
[[ "$name" == "production" ]] && echo "is production"

# Inequality
[[ "$name" != "development" ]] && echo "not dev"

# Empty / not empty
[[ -z "$name" ]] && echo "name is empty"
[[ -n "$name" ]] && echo "name is not empty"

# Contains (pattern matching — not regex)
[[ "$name" == *"prod"* ]] && echo "contains prod"

# Starts with
[[ "$name" == prod* ]] && echo "starts with prod"

# Ends with
[[ "$name" == *tion ]] && echo "ends with tion"
```

---

## Regex Matching with `[[ =~ ]]`

For more powerful matching, use the regex operator `=~`:

```bash
version="v2.14.3"

# Check version format: v followed by digits.digits.digits
if [[ "$version" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid semantic version: $version"
else
    echo "Invalid version format"
fi

# Extract matched groups using BASH_REMATCH
ip="192.168.1.100"
if [[ "$ip" =~ ^([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
    echo "Full match: ${BASH_REMATCH[0]}"   # 192.168.1.100
    echo "Octet 1: ${BASH_REMATCH[1]}"      # 192
    echo "Octet 2: ${BASH_REMATCH[2]}"      # 168
fi

# Validate email address (basic)
email="devops@company.com"
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi
```

---

## Trimming Whitespace

Bash has no built-in trim function, but parameter expansion handles it:

```bash
# Trim leading whitespace
ltrim() { echo "${1#"${1%%[! ]*}"}"; }

# Trim trailing whitespace
rtrim() { echo "${1%"${1##*[! ]}"}"; }

# Trim both sides (using sed for clarity in scripts)
trim() { echo "$1" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'; }

# Or with pure Bash (works for simple cases)
text="  hello world  "
trimmed="${text#"${text%%[![:space:]]*}"}"    # leading
trimmed="${trimmed%"${trimmed##*[![:space:]]}"}"  # trailing
echo "'$trimmed'"   # 'hello world'
```

---

## Practical String Script: Config File Parser

```bash
#!/usr/bin/env bash
# parse_config.sh — Parse KEY=VALUE config file, ignoring comments

parse_config() {
    local config_file="$1"
    declare -gA CONFIG

    while IFS='=' read -r key value; do
        # Skip comments and blank lines
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${key// /}" ]] && continue

        # Trim whitespace from key and value
        key="${key// /}"
        value="${value#"${value%%[![:space:]]*}"}"
        value="${value%"${value##*[![:space:]]}"}"

        CONFIG["$key"]="$value"
    done < "$config_file"
}

parse_config "app.conf"

echo "Database: ${CONFIG[DB_HOST]}:${CONFIG[DB_PORT]}"
echo "App Port: ${CONFIG[APP_PORT]}"
```

---

## Summary

> Bash string operations use **parameter expansion** — no external tools needed.
>
> **`${#str}`** — length. **`${str:offset:len}`** — substring.
>
> **`${str/old/new}`** — replace first. **`${str//old/new}`** — replace all.
>
> **`${str##*/}`** — remove longest prefix. **`${str%/*}`** — remove shortest suffix.
>
> **`${str^^}`** — uppercase. **`${str,,}`** — lowercase. (Bash 4+)
>
> **`[[ "$str" =~ regex ]]`** — regex match. Captured groups in `$BASH_REMATCH`.

| Operation | Syntax | Example |
|-----------|--------|---------|
| Length | `${#str}` | `${#name}` → `4` |
| Substring | `${str:2:5}` | `${path:0:4}` |
| Replace first | `${str/old/new}` | |
| Replace all | `${str//old/new}` | |
| Remove prefix | `${str##pattern}` | `${path##*/}` |
| Remove suffix | `${str%%pattern}` | `${file%.*}` |
| Uppercase | `${str^^}` | |
| Lowercase | `${str,,}` | |
| Regex match | `[[ $str =~ regex ]]` | |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Arrays](./arrays.md) &nbsp;|&nbsp; **Next:** [Conditionals →](../03_control_flow/conditionals.md)

**Related Topics:** [Variables](./variables.md) · [User Input](../05_input_output/user_input.md) · [File Operations](../05_input_output/file_operations.md)
