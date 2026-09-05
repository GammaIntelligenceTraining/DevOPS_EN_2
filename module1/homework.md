# Module 1 Homework: Fundamentals, Tooling & Processes

## Task 1: System Baseline Audit

Document your assigned Linux environment.

### Requirements:
1. Print your current username.
2. Display the Linux kernel version and CPU architecture.
3. Discover your machine's assigned internal IP address.
4. Verify whether PID 1 is running on your machine.

<details>
<summary>Hints (Click to expand)</summary>

- Username: `whoami`
- Kernel & architecture: `uname -a`
- Internal IP: `hostname -I`
- Process 1 name: `ps -p 1 -o comm=`

</details>

**Deliverable:**
List the four executed commands and their exact terminal outputs.

---

## Task 2: Directory Hierarchy and File Operations

Practice path navigation and directory operations using relative and absolute paths.

### Requirements:
1. In your home directory (`~`), create the directory path `ops_practice/production/logs` using a single command.
2. Inside `ops_practice/production/`, create an empty hidden file named `.deploy_token`.
3. Copy the hidden file `.deploy_token` into `ops_practice/production/logs/`.
4. From `ops_practice/production/logs/`, navigate to `/etc` using an absolute path.
5. Return to `ops_practice/production/logs/` using a relative path.

<details>
<summary>Hints (Click to expand)</summary>

- The `-p` flag in `mkdir` creates intermediate directories without erroring if they already exist.
- Hidden files begin with a leading period (`.`).
- Absolute paths begin with `/`. Relative paths use `..` to reference the parent directory.

</details>

**Deliverable:**
List the commands executed for steps 1 through 5.

---

## Task 3: Configuration Editing with Nano

Create and inspect configuration files in a terminal environment.

### Requirements:
1. Use `nano` to create and edit `~/ops_practice/production/app.conf`.
2. Insert the following lines:
   ```text
   ENVIRONMENT=production
   LOG_LEVEL=debug
   PORT=8080
   ```
3. Save the file and exit `nano`.
4. Print the entire file to the terminal using `cat`.
5. Use `grep` to filter and output only the line containing `PORT`.

<details>
<summary>Hints (Click to expand)</summary>

- Open: `nano <path>`
- Save: `Ctrl+O`, then press `Enter`
- Exit: `Ctrl+X`
- Pattern search: `grep "pattern" filename`

</details>

**Deliverable:**
Output of `cat ~/ops_practice/production/app.conf` and the `grep` command used.

---

## Task 4: Process Management and Job Control

Manage process execution states and background tasks.

### Requirements:
1. Start a command that sleeps for 400 seconds in the background.
2. Start a command that sleeps for 600 seconds in the foreground, suspend it, and move it to the background.
3. List active shell jobs to verify both commands are running.
4. Locate the Process ID (PID) of the 400-second sleep process using `ps` and `grep`.
5. Terminate the 400-second process using `kill` and its PID.
6. Terminate all remaining sleep processes using `pkill`.

<details>
<summary>Hints (Click to expand)</summary>

- Background execution: append `&` to the command
- Suspend foreground task: `Ctrl+Z`
- Resume in background: `bg`
- List jobs: `jobs`
- Process search: `ps aux | grep "sleep 400" | grep -v grep`
- Terminate by PID: `kill <PID>`
- Terminate by name: `pkill <name>`

</details>

**Deliverable:**
Output of `jobs` showing both background jobs before termination, and the commands used in steps 5 and 6.

---

## Task 5: Package Management with APT

Search for, install, and execute a new utility package.

### Requirements:
1. Refresh the local package repository index.
2. Search package repositories for the `tree` utility.
3. Install the `tree` package.
4. Run `tree -a ~/ops_practice` to generate a directory diagram showing all files, including hidden dotfiles.

<details>
<summary>Hints (Click to expand)</summary>

- Refresh index: `sudo apt update`
- Search: `apt search tree`
- Install: `sudo apt install -y tree`
- Visualize: `tree -a ~/ops_practice`

</details>

**Deliverable:**
Terminal output of `tree -a ~/ops_practice`.
