# Variables and Data in Bash

## The Labeled Box Analogy

Think of a warehouse full of shelves. Every item on a shelf sits in a labeled box. When a warehouse worker needs the invoice number, they do not memorize it — they grab the box labeled "INVOICE_NUMBER" and read what is inside.

Variables in Bash are exactly those labeled boxes. You put a value in (assignment), give it a name (the label), and retrieve it later (expansion). The shell manages the shelf space for you.

```
  Variable Assignment           Variable Expansion
  +-------------------+         +-------------------+
  | NAME="deploy-01"  |         | echo $NAME        |
  |    ^      ^       |         |       ^           |
  |    |      |       |   --->  | Shell opens the   |
  |  label  value     |         | box labeled NAME  |
  +-------------------+         | prints: deploy-01 |
                                 +-------------------+
```

---

## Declaring Variables

In Bash, you assign a variable with `=`. There must be **no spaces** around the `=`:

```bash
# Correct
server_name="web-prod-01"
port=8080
is_running=true

# WRONG — spaces cause errors
server_name = "web-prod-01"   # Error: 'server_name' is not a command
```

To use (expand) a variable, prefix it with `$`:

```bash
server_name="web-prod-01"
echo $server_name           # web-prod-01
echo "Server: $server_name" # Server: web-prod-01
echo "Server: ${server_name}"  # Same — braces are explicit, safer
```

**Always use double quotes around variable expansions** to prevent word splitting and globbing:

```bash
filename="my file.txt"       # filename has a space
rm $filename                 # WRONG: tries to delete "my" and "file.txt"
rm "$filename"               # CORRECT: treats as one argument
```

---

## Variable Naming Rules

```
  VALID names          INVALID names
  +--------------+     +--------------+
  | myvar        |     | 1stvar       |  (starts with number)
  | my_var       |     | my-var       |  (hyphens not allowed)
  | MY_VAR       |     | my var       |  (spaces not allowed)
  | _private     |     | my.var       |  (dots not allowed)
  | deploy2prod  |     |              |
  +--------------+     +--------------+

  Convention: lowercase for local vars, UPPER_CASE for environment vars
```

---

## Local vs Global (Environment) Variables

By default, a variable you create exists only in the **current shell session**. It is a "local" shell variable. Child processes (scripts you call, commands you run) cannot see it.

```bash
MY_VAR="hello"
bash -c 'echo $MY_VAR'   # prints nothing — child bash cannot see it
```

To share a variable with child processes, **export** it:

```bash
export MY_VAR="hello"
bash -c 'echo $MY_VAR'   # now prints: hello
```

An exported variable becomes an **environment variable** — it travels with the process into any child it spawns.

```
  Shell Session
  +-------------------------------------+
  |  shell variable: MY_VAR="hello"     |
  |  (only visible here)                |
  |                                     |
  |  export MY_VAR                      |
  |                                     |
  |  environment variable: MY_VAR       |
  |  (visible to all child processes)   |
  |                                     |
  |  +-- child process (bash script) -- |
  |  |   can read MY_VAR               ||
  |  +--------------------------------- |
  +-------------------------------------+
```

---

## Built-in Environment Variables

Linux automatically provides a set of environment variables in every session:

```bash
echo $HOME      # /home/ubuntu        — your home directory
echo $USER      # ubuntu              — current username
echo $PATH      # /usr/bin:/bin:...   — where shell looks for commands
echo $SHELL     # /bin/bash           — your default shell
echo $PWD       # /home/ubuntu/app    — current working directory
echo $HOSTNAME  # web-prod-01         — machine hostname
echo $LANG      # en_US.UTF-8         — locale setting
```

Practical use in a deployment script:

```bash
#!/usr/bin/env bash
LOG_DIR="$HOME/logs"
APP_DIR="$HOME/app"
echo "Deploying to: $APP_DIR on $HOSTNAME"
```

---

## `export` and `readonly`

```bash
# export: make variable available to child processes
export DATABASE_URL="postgres://localhost/mydb"

# readonly: prevent the variable from being changed
readonly MAX_RETRIES=3
MAX_RETRIES=5    # Error: MAX_RETRIES: readonly variable

# Combine: export and readonly
export readonly API_KEY="abc123"

# See all exported variables
env | grep MY_
printenv DATABASE_URL
```

---

## Special Variables

Bash provides automatic special variables that carry script metadata:

```bash
#!/usr/bin/env bash
# Assume you run: ./deploy.sh production v2.1.0

echo $0     # ./deploy.sh    — name of the script
echo $1     # production     — first argument
echo $2     # v2.1.0         — second argument
echo $#     # 2              — number of arguments
echo $@     # production v2.1.0  — all arguments (each quoted separately)
echo $*     # production v2.1.0  — all arguments as one string
echo $$     # 12345          — PID of current script
echo $!     # 12346          — PID of last background process
echo $?     # 0              — exit code of last command
```

### Using `$@` in a real script

```bash
#!/usr/bin/env bash
# Usage: ./deploy.sh env1 env2 env3

echo "Deploying to $# environments:"
for env in "$@"; do
    echo "  -> $env"
done
```

### Checking the exit code with `$?`

```bash
cp important.txt /backup/
if [ $? -eq 0 ]; then
    echo "Backup succeeded"
else
    echo "Backup FAILED"
fi
```

---

## Arithmetic with Variables

Bash variables are strings by default. For math, use `$(( ))`:

```bash
count=10
new_count=$(( count + 5 ))
echo $new_count    # 15

# Increment
count=$(( count + 1 ))
(( count++ ))       # shorter form

# All arithmetic operators
echo $(( 10 + 3 ))  # 13
echo $(( 10 - 3 ))  # 7
echo $(( 10 * 3 ))  # 30
echo $(( 10 / 3 ))  # 3 (integer division)
echo $(( 10 % 3 ))  # 1 (remainder)
```

---

## Default Values

You can provide fallbacks when a variable might be empty:

```bash
# ${var:-default}  — use default if var is unset or empty
ENVIRONMENT="${1:-development}"
echo "Deploying to: $ENVIRONMENT"

# ${var:=default}  — assign default if var is unset or empty
: "${LOG_LEVEL:=INFO}"
echo "Log level: $LOG_LEVEL"

# ${var:?error_message}  — error and exit if var is unset
DB_PASSWORD="${DB_PASSWORD:?ERROR: DB_PASSWORD must be set}"
```

Real deployment script usage:

```bash
#!/usr/bin/env bash
ENVIRONMENT="${1:-staging}"
VERSION="${2:-latest}"
REGION="${AWS_REGION:-us-east-1}"

echo "Deploying version $VERSION to $ENVIRONMENT in $REGION"
```

---

## Summary

> **Variables** are named storage boxes for values. Assignment uses `=` (no spaces). Expansion uses `$name` or `${name}`.
>
> **Always quote** variable expansions in strings: `"$var"` not `$var`.
>
> **`export VAR`** makes a variable visible to child processes (environment variable).
>
> **`readonly VAR`** prevents accidental changes.
>
> **Special variables**: `$0` script name, `$1`-`$9` arguments, `$#` arg count, `$@` all args, `$?` last exit code.
>
> **Default values**: `${VAR:-default}` is your safety net for unset variables.

| Variable | Meaning |
|----------|---------|
| `$0` | Script name |
| `$1`, `$2` | Positional arguments |
| `$#` | Number of arguments |
| `$@` | All arguments |
| `$?` | Last exit code |
| `$$` | Current process ID |
| `$HOME` | Home directory |
| `$USER` | Current username |
| `$PATH` | Command search path |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Shebang and Execution](../01_shell_basics/shebang_and_execution.md) &nbsp;|&nbsp; **Next:** [Arrays →](./arrays.md)

**Related Topics:** [String Operations](./string_operations.md) · [Functions](../04_functions/functions.md) · [Exit Codes](../06_error_handling/exit_codes.md)
