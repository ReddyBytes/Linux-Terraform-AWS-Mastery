# Shebang Lines and Script Execution

## The Language Translator Analogy

Imagine you walk into the United Nations and hand someone a document written in Japanese. Before they can act on it, they need to know: "Which interpreter do I call for this?" The cover page of the document says "Interpreter: Japanese — Room 4B." They call the Japanese interpreter, who reads every word aloud in the correct language.

The shebang line is that cover page instruction. When Linux receives your script file, the shebang tells it exactly which program (interpreter) to hand the file to.

```
  Linux OS receives file "deploy.sh"
         |
         v
  Reads first line: #!/bin/bash
         |
         +-- "Ah, use /bin/bash as the interpreter"
         |
         v
  /bin/bash reads and executes the rest of the file
```

---

## Two Shebang Styles

You will see two common shebang formats in the wild:

### Style 1: Hardcoded path
```bash
#!/bin/bash
```

This says: "Use the Bash binary located at exactly `/bin/bash`."

- Works on most Linux systems (Ubuntu, CentOS, Amazon Linux)
- May fail on systems where Bash is installed elsewhere (some macOS setups, NixOS)
- Predictable and explicit

### Style 2: Using `env`
```bash
#!/usr/bin/env bash
```

This says: "Search the user's `PATH` for `bash` and use whatever version is found first."

- More portable — works even if Bash is at `/usr/local/bin/bash` or `/opt/homebrew/bin/bash`
- Preferred for scripts that must run on macOS, Linux, and other Unix-like systems
- The DevOps community increasingly prefers this style

```
  #!/bin/bash                    #!/usr/bin/env bash
  +-------------------+          +-------------------------+
  | Hardcoded path    |          | Searches PATH           |
  | Fast, predictable |          | Portable, flexible      |
  | Best for servers  |          | Best for shared scripts |
  +-------------------+          +-------------------------+
```

### Recommendation

Use `#!/usr/bin/env bash` for scripts you share or open-source. Use `#!/bin/bash` for scripts locked to a known Linux server environment where you control the setup.

---

## Other Common Shebangs

```bash
#!/usr/bin/env python3    # Python script
#!/usr/bin/env node       # Node.js script
#!/usr/bin/env ruby       # Ruby script
#!/bin/sh                 # POSIX sh (most compatible, fewest features)
#!/usr/bin/env zsh        # Zsh script
```

---

## `chmod +x`: Granting Execute Permission

Linux tracks three types of permission for three types of audience:

```
  Permission string: -rwxr-xr-x
                      ^^^  ^  ^
                      |    |  |
                      |    |  +-- Others (everyone else)
                      |    +-- Group (users in same group)
                      +-- Owner (you)

  r = read    (4)
  w = write   (2)
  x = execute (1)
```

When you create a new file, it has no execute bit:
```
  -rw-r--r-- deploy.sh
```

After `chmod +x`:
```
  -rwxr-xr-x deploy.sh
```

You can also use numeric mode:
```bash
chmod 755 deploy.sh    # rwxr-xr-x  (owner: full, group/others: read+execute)
chmod 700 deploy.sh    # rwx------  (owner only — good for scripts with passwords)
```

---

## Three Ways to Run a Script

### Method 1: Direct execution with `./`

```bash
chmod +x deploy.sh
./deploy.sh
```

- The `./` means "in the current directory"
- Linux does NOT search the current directory for executables by default (a security feature)
- The shebang line determines which interpreter runs the script
- Requires execute permission

### Method 2: Explicit interpreter

```bash
bash deploy.sh
```

- Tells Linux directly: "use bash to run this file"
- Does NOT require execute permission
- The shebang is ignored — bash is already specified
- Useful for debugging or running scripts you cannot `chmod`

```bash
# You can also pass options this way:
bash -x deploy.sh    # run with trace mode (shows each command)
bash -n deploy.sh    # syntax check without running
```

### Method 3: Sourcing with `.` or `source`

```bash
source deploy.sh
# or equivalently:
. deploy.sh
```

This is fundamentally different from the other two methods:

```
  Method 1 & 2: ./script.sh or bash script.sh
  +------------------------------------------+
  | Creates a NEW child process (subshell)   |
  | Script runs in its own environment       |
  | Variable changes do NOT affect parent    |
  | Script exits: child process ends         |
  +------------------------------------------+

  Method 3: source script.sh
  +------------------------------------------+
  | Runs in the CURRENT shell (no subshell)  |
  | Variable changes DO affect current shell |
  | Used for configuration files (.bashrc)   |
  | Used for loading function libraries      |
  +------------------------------------------+
```

**Practical example showing the difference:**

```bash
# test_vars.sh
#!/bin/bash
MY_VAR="hello from script"

# --- Test with subshell ---
bash test_vars.sh
echo $MY_VAR          # prints nothing — variable died with child process

# --- Test with source ---
source test_vars.sh
echo $MY_VAR          # prints "hello from script" — variable lives on
```

### When to use `source`

- Loading shared function libraries into your script
- Applying environment variable files (`.env` files)
- Reloading your `.bashrc` after editing it: `source ~/.bashrc`
- Setting variables in your current shell session from a script

---

## A Complete Example

Here is a script that demonstrates all the concepts:

```bash
#!/usr/bin/env bash
# =============================================================
# Script: env_check.sh
# Purpose: Check that required tools are installed before deploy
# =============================================================

echo "Checking required tools..."

# Check each tool exists
for tool in git docker aws terraform; do
    if command -v "$tool" &>/dev/null; then
        echo "  [OK]  $tool found at $(command -v "$tool")"
    else
        echo "  [MISSING]  $tool not found — please install it"
    fi
done

echo ""
echo "Shell info:"
echo "  This script is running with: $BASH_VERSION"
echo "  Script path: $0"
```

Run it:
```bash
chmod +x env_check.sh
./env_check.sh
```

---

## Summary

> **`#!/bin/bash`** — hardcoded shebang, reliable on known Linux systems.
>
> **`#!/usr/bin/env bash`** — portable shebang, searches PATH for bash.
>
> **`chmod +x script.sh`** — add execute permission so you can run with `./`.
>
> **`./script.sh`** — runs in a subshell. Variables do not leak back to your session.
>
> **`bash script.sh`** — same subshell behavior. Useful when you cannot chmod.
>
> **`source script.sh`** — runs in current shell. Variable changes persist.

| Method | Needs chmod? | Subshell? | Use case |
|--------|-------------|-----------|----------|
| `./script.sh` | Yes | Yes | Normal script execution |
| `bash script.sh` | No | Yes | Debug mode, no permissions |
| `source script.sh` | No | No | Load vars/functions |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← First Script](./first_script.md) &nbsp;|&nbsp; **Next:** [Variables →](../02_variables_and_data/variables.md)

**Related Topics:** [Functions](../04_functions/functions.md) · [Debugging](../06_error_handling/debugging.md)
