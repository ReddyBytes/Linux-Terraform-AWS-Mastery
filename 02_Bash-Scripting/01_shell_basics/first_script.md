# Your First Shell Script

## The Recipe Analogy

Imagine you work in a busy restaurant kitchen. Every morning, the head chef prepares the same 12 dishes: boil water, chop vegetables, season the pan, cook the protein, plate and garnish. Doing this manually every single day wastes time and invites mistakes.

So the chef writes down the recipe — a precise, ordered list of instructions. Any cook can pick up that recipe card and reproduce the exact same dish, every time, without the head chef standing there explaining each step.

**A shell script is that recipe card for your computer.**

Instead of typing the same commands in your terminal over and over, you write them once in a file. The computer reads the file top to bottom and executes each instruction in order. Consistent, repeatable, and fast.

```
  You (Chef)                  Shell Script (Recipe Card)
  +----------+                +-------------------------+
  | Commands |  -- writes --> |  #!/bin/bash            |
  | typed    |                |  command 1              |
  | manually |                |  command 2              |
  |  every   |                |  command 3              |
  |  time    |                +-------------------------+
  +----------+                          |
                                        | runs automatically
                                        v
                              +------------------+
                              |  Linux System    |
                              |  executes each   |
                              |  command in order|
                              +------------------+
```

---

## What Is a Shell Script?

A shell script is a plain text file containing a sequence of commands that your shell (Bash, in this case) can read and execute. The shell is the program that interprets commands — either typed interactively or read from a file.

Shell scripts are used everywhere in the DevOps world:
- Bootstrapping new servers on AWS
- Deploying application code
- Rotating log files at midnight
- Monitoring disk usage and sending alerts
- Running database migrations before a release

---

## The Shebang Line: The First Line of Every Script

Open any professional shell script and you will see this as line 1:

```bash
#!/bin/bash
```

This is called the **shebang** (or hashbang). It tells the operating system: "Use this specific program to run the rest of this file."

```
  #!/bin/bash
  ^^ ^^^^^^^^
  |     |
  |     +-- Path to the interpreter (Bash)
  |
  +-- Shebang: tells the OS this is a script, not plain text
```

Without the shebang, the OS does not know how to run your file. You would have to explicitly type `bash myscript.sh` every time. With the shebang, you can run it directly as `./myscript.sh`.

---

## Writing Your First Script

Open a terminal and create a new file:

```bash
nano hello.sh
```

Type this content:

```bash
#!/bin/bash

# This is a comment — the shell ignores lines starting with #
# Comments are notes for humans reading your code

echo "Hello, World!"
echo "Today is: $(date)"
echo "You are logged in as: $USER"
echo "Your home directory is: $HOME"
```

Save the file (in nano: `Ctrl+O`, `Enter`, `Ctrl+X`).

---

## Making the Script Executable

A new file is just text. Before you can run it as a program, you must grant it execute permission:

```bash
chmod +x hello.sh
```

Check the permissions before and after:

```
Before chmod:   -rw-r--r--  hello.sh   (read/write, not executable)
After  chmod:   -rwxr-xr-x  hello.sh   (read/write/execute)
                   ^
                   x = execute permission added
```

The `chmod +x` command adds the execute bit for the owner, group, and others. We will cover permissions in detail later, but for now just remember: **every script needs `chmod +x` before you can run it with `./`**.

---

## Running Your Script

You have three ways to run a script:

**Method 1: Direct execution (requires `chmod +x`)**
```bash
./hello.sh
```

**Method 2: Explicit interpreter (no `chmod +x` needed)**
```bash
bash hello.sh
```

**Method 3: Source the script (runs in current shell session)**
```bash
source hello.sh
# or the shorthand:
. hello.sh
```

The difference between methods 2 and 3 matters and is explained in the next file.

---

## Expected Output

When you run `./hello.sh`, you should see something like:

```
Hello, World!
Today is: Sat Mar 15 09:30:00 UTC 2026
You are logged in as: ubuntu
Your home directory is: /home/ubuntu
```

Your date, username, and home directory will differ — but the script dynamically fills them in using **variables** (`$USER`, `$HOME`) and **command substitution** (`$(date)`).

---

## A Real-World Starter Script

Here is a slightly more useful first script that a junior DevOps engineer might write on their first day:

```bash
#!/bin/bash

# server_info.sh — Print key information about this server
# Author: Your Name
# Date: 2026-03-15

echo "============================================"
echo "          SERVER INFORMATION REPORT"
echo "============================================"
echo ""
echo "Hostname    : $(hostname)"
echo "OS          : $(uname -o)"
echo "Kernel      : $(uname -r)"
echo "Uptime      : $(uptime -p)"
echo "Logged in as: $USER"
echo "Current dir : $PWD"
echo ""
echo "Disk Usage (/):"
df -h / | tail -1
echo ""
echo "Memory:"
free -h | grep Mem
echo "============================================"
```

This is a real script you would use when SSHing into an unfamiliar server to quickly orient yourself.

---

## Script Structure Best Practices

Every professional script should have:

```bash
#!/bin/bash
# =============================================================
# Script Name : deploy.sh
# Description : Deploys application to production server
# Author      : DevOps Team
# Created     : 2026-03-15
# Usage       : ./deploy.sh [environment] [version]
# =============================================================

# -- Your code starts here --
```

This header comment block is like the label on a recipe card. Anyone picking up the script immediately knows what it does, who wrote it, and how to use it.

---

## Summary

> **What is a shell script?** A plain text file of commands that Bash reads and executes in order — your automation recipe card.
>
> **The shebang (`#!/bin/bash`)** goes on line 1 and tells the OS which interpreter to use.
>
> **`chmod +x script.sh`** grants execute permission so you can run it with `./script.sh`.
>
> **Comments (`#`)** are for humans. Use them generously.
>
> **`echo`** prints text. **`$(command)`** inserts command output. **`$VARIABLE`** inserts a variable's value.

| Task | Command |
|------|---------|
| Create a script | `nano myscript.sh` |
| Make executable | `chmod +x myscript.sh` |
| Run directly | `./myscript.sh` |
| Run with bash | `bash myscript.sh` |
| View permissions | `ls -l myscript.sh` |

---

**[🏠 Back to README](../../README.md)**

**Prev:** [← Linux Interview Questions](../../01_Linux/99_interview_master/linux_questions.md) &nbsp;|&nbsp; **Next:** [Shebang and Execution →](./shebang_and_execution.md)

**Related Topics:** [Variables](../02_variables_and_data/variables.md) · [User Input](../05_input_output/user_input.md)
