# Case Statements in Bash

## The Postal Sorting Analogy

Walk into a postal sorting facility and you will see parcels arriving on a conveyor belt. A sorter examines each package's destination and routes it: package to New York goes to bin A, package to London goes to bin B, package to Tokyo goes to bin C. If the destination is unknown, it goes to the "undeliverable" pile.

The `case` statement in Bash is that sorter. One item (a variable) comes in, and Bash routes it to the matching code block. It is a cleaner, more readable alternative to long `if/elif/elif/elif` chains when you are checking one variable against many possible values.

```
  if/elif chain:              case statement:
  +--------------------+      +------------------------+
  | if [[ x == "a" ]] |      | case "$x" in           |
  | elif [[ x == "b" ]]|     |   "a") do_a ;;         |
  | elif [[ x == "c" ]]|     |   "b") do_b ;;         |
  | elif [[ x == "d" ]]|     |   "c") do_c ;;         |
  | elif ...           |      |   "d") do_d ;;         |
  | fi                 |      |   *)   fallback ;;     |
  +--------------------+      | esac                   |
  Hard to read at 5+ checks   +------------------------+
                              Clean at any size
```

---

## Basic Case Statement Syntax

```bash
case "$variable" in
    pattern1)
        # commands for pattern1
        ;;
    pattern2)
        # commands for pattern2
        ;;
    *)
        # default — matches anything not caught above
        ;;
esac
```

Key syntax rules:
- `case` ... `in` opens the block
- Each pattern ends with `)`
- Each block ends with `;;`
- `*)` is the catch-all default (like `else`)
- `esac` closes the block (`case` spelled backwards)

---

## Simple Example: Environment Selection

```bash
#!/usr/bin/env bash

ENVIRONMENT="${1:-}"

case "$ENVIRONMENT" in
    dev|development)
        DB_HOST="localhost"
        APP_PORT=3000
        DEBUG=true
        ;;
    staging|stage)
        DB_HOST="staging-db.internal"
        APP_PORT=8080
        DEBUG=false
        ;;
    prod|production)
        DB_HOST="prod-db.internal"
        APP_PORT=80
        DEBUG=false
        ;;
    *)
        echo "ERROR: Unknown environment '$ENVIRONMENT'"
        echo "Usage: $0 [dev|staging|prod]"
        exit 1
        ;;
esac

echo "Starting app: host=$DB_HOST port=$APP_PORT debug=$DEBUG"
```

The `|` (pipe) inside a pattern acts as OR — `dev|development` matches either "dev" or "development".

---

## Pattern Matching in `case`

`case` uses glob-style patterns, not regex:

```bash
filename="backup-2026-03-15.tar.gz"

case "$filename" in
    *.tar.gz | *.tgz)
        echo "Compressed tarball: $filename"
        tar -xzf "$filename"
        ;;
    *.tar.bz2)
        echo "Bzip2 tarball: $filename"
        tar -xjf "$filename"
        ;;
    *.zip)
        echo "ZIP archive: $filename"
        unzip "$filename"
        ;;
    *.log)
        echo "Log file: $filename"
        ;;
    backup-????)
        echo "Matches backup with 4-char code"
        ;;
    *)
        echo "Unknown file type: $filename"
        ;;
esac
```

Pattern syntax:
```
  *        matches any string (zero or more chars)
  ?        matches any single character
  [abc]    matches a, b, or c
  [a-z]    matches any lowercase letter
  pattern1 | pattern2    matches either pattern
```

---

## Fall-through with `;&` and `;;&`

By default, `;;` exits the case block after the first match. Two alternatives allow fall-through:

```bash
value="b"

case "$value" in
    a)
        echo "matched a"
        ;;&    # continue checking patterns below (not just next)
    b)
        echo "matched b"
        ;;     # stop here
    c)
        echo "matched c"
        ;;
esac
# Output: matched b

# Using ;& — falls through to NEXT block unconditionally
case "$value" in
    b)
        echo "matched b"
        ;&    # unconditionally fall into next block
    c)
        echo "also ran c block"
        ;;
esac
# Output: matched b
#         also ran c block
```

In practice, `;;` covers 99% of real scripts. `;&` and `;;&` are niche but good to know.

---

## Practical Use: Interactive Menu Script

```bash
#!/usr/bin/env bash
# server_menu.sh — Admin menu for server management

show_menu() {
    echo ""
    echo "============================================"
    echo "          SERVER MANAGEMENT MENU"
    echo "============================================"
    echo "  1) Check server status"
    echo "  2) Restart application"
    echo "  3) View recent logs"
    echo "  4) Run health check"
    echo "  5) Deploy latest version"
    echo "  q) Quit"
    echo "============================================"
    echo -n "Choose an option: "
}

while true; do
    show_menu
    read -r choice

    case "$choice" in
        1)
            echo "Checking server status..."
            systemctl status myapp
            ;;
        2)
            echo "Restarting application..."
            systemctl restart myapp && echo "Restarted successfully"
            ;;
        3)
            echo "Last 50 log lines:"
            journalctl -u myapp -n 50 --no-pager
            ;;
        4)
            echo "Running health check..."
            curl -sf "http://localhost:8080/health" && echo "HEALTHY" || echo "UNHEALTHY"
            ;;
        5)
            read -p "Enter version to deploy (e.g. v2.1.0): " version
            ./deploy.sh production "$version"
            ;;
        q|Q|quit|exit)
            echo "Goodbye."
            exit 0
            ;;
        "")
            # User pressed Enter with no input — do nothing
            ;;
        *)
            echo "Invalid option: '$choice'"
            ;;
    esac
done
```

---

## Practical Use: Argument Parsing

```bash
#!/usr/bin/env bash
# backup.sh — with command-line argument parsing via case

show_help() {
    cat << EOF
Usage: $0 [OPTIONS] <target>

OPTIONS:
  -h, --help        Show this help message
  -v, --verbose     Enable verbose output
  -d, --dry-run     Show what would be done without doing it
  -e, --encrypt     Encrypt the backup
  -r, --remote HOST Upload to remote host

EXAMPLES:
  $0 /var/www/html
  $0 --verbose --encrypt /home/user
  $0 --dry-run --remote backup.server.com /data
EOF
}

VERBOSE=false
DRY_RUN=false
ENCRYPT=false
REMOTE_HOST=""

# Parse arguments
while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--verbose)
            VERBOSE=true
            shift    # consume this argument
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        -e|--encrypt)
            ENCRYPT=true
            shift
            ;;
        -r|--remote)
            REMOTE_HOST="$2"
            shift 2    # consume flag AND its value
            ;;
        -*)
            echo "ERROR: Unknown option '$1'"
            show_help
            exit 1
            ;;
        *)
            TARGET="$1"
            shift
            ;;
    esac
done

echo "Settings: verbose=$VERBOSE dry-run=$DRY_RUN encrypt=$ENCRYPT remote=$REMOTE_HOST"
echo "Target: $TARGET"
```

---

## Practical Use: Log Level Handler

```bash
#!/usr/bin/env bash

LOG_LEVEL="${LOG_LEVEL:-INFO}"

log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    case "$level" in
        DEBUG)   [[ "$LOG_LEVEL" == "DEBUG" ]] || return 0 ;;
        INFO)    [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO)$ ]] || return 0 ;;
        WARN)    [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO|WARN)$ ]] || return 0 ;;
        ERROR)   ;;    # always log errors
        *)       echo "Unknown log level: $level" ; return 1 ;;
    esac

    echo "[$timestamp] [$level] $message"
}

log DEBUG "Connecting to database..."
log INFO  "Application starting on port 8080"
log WARN  "Memory usage at 78%"
log ERROR "Failed to connect to cache server"
```

---

## Summary

> **`case`** provides clean multi-branch logic when testing one value against many patterns.
>
> Patterns use **glob syntax**: `*` (any string), `?` (any char), `[abc]` (character class).
>
> **`|` between patterns** means OR: `dev|development)` matches either.
>
> **`;;`** ends a case block and exits the `case` statement (use this 99% of the time).
>
> **`*)`** is the catch-all default that matches anything not matched above.
>
> Common real-world uses: environment selection, interactive menus, argument parsing, log levels.

| Syntax | Meaning |
|--------|---------|
| `pattern)` | Match this pattern |
| `;;` | End block, exit case |
| `;&` | Fall through to next block |
| `;;&` | Continue checking patterns |
| `p1\|p2)` | Match p1 OR p2 |
| `*)` | Default (catch-all) |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Loops](./loops.md) &nbsp;|&nbsp; **Next:** [Functions →](../04_functions/functions.md)

**Related Topics:** [Conditionals](./conditionals.md) · [User Input](../05_input_output/user_input.md)
