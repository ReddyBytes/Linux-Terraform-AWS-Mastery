# Exit Codes in Bash

## The Traffic Light Inspection Analogy

Imagine a safety inspector checking a building. When they finish their inspection, they do not just disappear — they file a report with a status code: 0 means "passed, no issues", 1 means "minor violations", 2 means "critical failure, building must close". The next person in the chain (the city office) reads that status code and decides what to do.

Every command in Linux works the same way. When a command finishes, it leaves behind an exit code — a number that tells the caller whether it succeeded or failed. Your scripts can read these codes and make intelligent decisions: retry, alert, skip, or stop.

```
  Command runs
       |
       v
  Command finishes
       |
       +-- Exit code 0 -----> SUCCESS (true in if context)
       |
       +-- Exit code 1-255 -> FAILURE (false in if context)
                                ^
                                |
                     Different values mean different failures
                     (1=general error, 2=misuse, 126=not executable, etc.)
```

---

## `$?`: Reading the Exit Code

The special variable `$?` always contains the exit code of the most recently run command:

```bash
ls /etc/hosts
echo "Exit code: $?"      # 0 (success — file exists)

ls /nonexistent_file
echo "Exit code: $?"      # 2 (failure — file not found)

grep "nonexistent" /etc/hosts
echo "Exit code: $?"      # 1 (grep returns 1 when no match found)
```

**Important: Read `$?` immediately.** Every command resets it:

```bash
# WRONG: $? is the exit code of echo, not grep
grep "pattern" file
echo "Done"
if [[ $? -eq 0 ]]; then ...   # always 0 — that's echo's exit code!

# CORRECT: check $? immediately after the command
grep "pattern" file
if [[ $? -eq 0 ]]; then ...   # correct — grep's exit code

# BETTER: use the command directly in if
if grep "pattern" file; then ...
```

---

## Common Exit Code Values

```
  0     Success
  1     General error (catch-all)
  2     Misuse of shell builtin
  126   Command found but not executable
  127   Command not found
  128   Invalid exit argument
  128+N Fatal signal N received (e.g. 130 = killed by Ctrl+C = SIGINT=2)
  130   Script terminated by Ctrl+C
  255   Exit status out of range
```

Define meaningful exit codes in your scripts:

```bash
#!/usr/bin/env bash
# Exit code constants — self-documenting
readonly EXIT_SUCCESS=0
readonly EXIT_GENERAL_ERROR=1
readonly EXIT_INVALID_ARGS=2
readonly EXIT_CONFIG_NOT_FOUND=3
readonly EXIT_SERVICE_FAILED=4
readonly EXIT_HEALTH_CHECK_FAILED=5

if [[ -z "$1" ]]; then
    echo "ERROR: No argument provided"
    exit $EXIT_INVALID_ARGS
fi

if [[ ! -f "$CONFIG" ]]; then
    echo "ERROR: Config not found"
    exit $EXIT_CONFIG_NOT_FOUND
fi
```

---

## `exit N`: Setting Your Script's Exit Code

```bash
#!/usr/bin/env bash

deploy() {
    # ... deployment logic
    if some_step_fails; then
        echo "Deployment failed"
        exit 1    # exit the ENTIRE script with code 1
    fi
}

echo "Starting"
deploy
echo "This runs only if deploy() does not call exit"
exit 0    # explicit success (optional at end of script)
```

When a script exits without an explicit `exit`, it exits with the exit code of the last command that ran.

---

## `set -e`: Exit on Error

Without `set -e`, a script keeps running even after a command fails:

```bash
#!/usr/bin/env bash
# Without set -e:
rm /important/file       # fails silently
cp /src /dest            # continues running on the wrong system state
deploy_application       # may deploy corrupted state
```

With `set -e`, any command that returns a non-zero exit code causes the entire script to stop immediately:

```bash
#!/usr/bin/env bash
set -e    # exit immediately on any error

git pull origin main          # stops here if git fails
docker build -t myapp:latest  # stops here if build fails
docker push myapp:latest      # stops here if push fails
ssh prod-server "docker pull && docker-compose up -d"
echo "Deployment complete"
```

### Exceptions to `set -e`

Some commands are allowed to fail. Use `|| true` to suppress the exit:

```bash
set -e

# These are expected to sometimes return non-zero
grep "optional-pattern" /var/log/app.log || true   # grep returns 1 if no match
rm -f /tmp/oldfile || true                          # rm returns error if file missing

# Or use if/else (also exempt from set -e)
if ! command_that_might_fail; then
    echo "It failed, that's OK"
fi
```

---

## `set -o pipefail`: Catch Errors in Pipelines

`set -e` alone does not catch errors inside pipelines:

```bash
set -e
# This does NOT exit with set -e alone!
cat nonexistent_file | grep "pattern" | wc -l
# cat fails (exit 1), but grep returns 0, so the pipeline exit code is 0
```

`set -o pipefail` makes a pipeline fail if ANY command in it fails:

```bash
set -e -o pipefail
# Now this WILL cause exit:
cat nonexistent_file | grep "pattern" | wc -l
# cat fails -> entire pipeline returns non-zero -> script exits
```

---

## `set -u`: Exit on Undefined Variable

```bash
set -u    # exit on undefined variable

echo $UNDEFINED_VAR    # would normally print blank
                        # with set -u: error and exit
```

Use `${VAR:-default}` to provide defaults when using `set -u`:

```bash
set -u
ENVIRONMENT="${ENVIRONMENT:-development}"    # safe default
VERSION="${1:-latest}"                        # safe default for arg
```

---

## The Safety Trio

Most production Bash scripts start with this:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- `e` — exit on error
- `u` — exit on undefined variable
- `o pipefail` — exit if any pipe command fails

This is often called "strict mode" and catches the most common classes of bugs before they cause real damage.

---

## Checking Command Success Patterns

```bash
#!/usr/bin/env bash
set -euo pipefail

# Pattern 1: if/then
if ! docker pull "myapp:$VERSION"; then
    echo "ERROR: Failed to pull image"
    exit 1
fi

# Pattern 2: || (or) shorthand
docker pull "myapp:$VERSION" || { echo "Pull failed"; exit 1; }

# Pattern 3: check $? explicitly
docker push "myapp:$VERSION"
if [[ $? -ne 0 ]]; then
    echo "Push failed with exit code $?"
    exit 1
fi

# Pattern 4: trap ERR for automatic error handling (see traps section)
trap 'echo "ERROR: Command failed on line $LINENO"' ERR
```

---

## Real-World Example: Deployment with Exit Code Handling

```bash
#!/usr/bin/env bash
set -euo pipefail

# Exit code definitions
readonly E_SUCCESS=0
readonly E_BUILD_FAILED=10
readonly E_PUSH_FAILED=11
readonly E_DEPLOY_FAILED=12
readonly E_HEALTHCHECK_FAILED=13

ENVIRONMENT="${1:-staging}"
VERSION="${2:-}"

[[ -z "$VERSION" ]] && { echo "Usage: $0 <env> <version>"; exit 2; }

echo "=== Building ==="
if ! docker build -t "myapp:$VERSION" .; then
    echo "Build failed"
    exit $E_BUILD_FAILED
fi

echo "=== Pushing ==="
if ! docker push "myapp:$VERSION"; then
    echo "Push failed"
    exit $E_PUSH_FAILED
fi

echo "=== Deploying ==="
if ! ssh "deploy@$ENVIRONMENT-server" "docker pull myapp:$VERSION && docker-compose up -d"; then
    echo "Deploy failed"
    exit $E_DEPLOY_FAILED
fi

echo "=== Health Check ==="
sleep 10
if ! curl -sf "https://$ENVIRONMENT.myapp.com/health" &>/dev/null; then
    echo "Health check failed — initiating rollback"
    ssh "deploy@$ENVIRONMENT-server" "docker-compose rollback"
    exit $E_HEALTHCHECK_FAILED
fi

echo "Deployment successful!"
exit $E_SUCCESS
```

---

## Summary

> **Exit code 0** = success. **Non-zero** = failure. Every command sets `$?`.
>
> **`$?`** contains the last command's exit code. Read it immediately.
>
> **`exit N`** terminates your script with exit code N.
>
> **`set -e`** makes the script exit immediately if any command fails.
>
> **`set -o pipefail`** makes a pipeline fail if any stage fails.
>
> **`set -u`** makes the script exit if you use an unset variable.
>
> **Use `set -euo pipefail`** at the top of every production script.

| Setting | Effect |
|---------|--------|
| `set -e` | Exit on any error |
| `set -u` | Exit on undefined variable |
| `set -o pipefail` | Exit if any pipe command fails |
| `set -x` | Trace mode (print each command) |
| `exit 0` | Exit with success |
| `exit 1` | Exit with failure |
| `$?` | Last exit code |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Pipes and Redirection](../05_input_output/pipes_and_redirection.md) &nbsp;|&nbsp; **Next:** [Traps →](./traps.md)

**Related Topics:** [Traps](./traps.md) · [Debugging](./debugging.md) · [Deployment Scripts](../08_real_world_scripts/deployment_scripts.md)
