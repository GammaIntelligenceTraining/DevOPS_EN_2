# Module 4: Core Systems, Security & Network Consolidation — Student Handout

A comprehensive operational field guide consolidating Linux filesystem navigation, process forensics, access control, network socket auditing, and service management.

---

## 1. The DevOps Core Architecture

Modern systems administration requires diagnosing three interdependent operational layers: active processes executing in memory, user identities and permissions governing access, and network sockets receiving traffic. When an incident occurs, an engineer navigates across all three layers systematically.

```text
+-----------------------------------------------------------------------------------+
| PILLAR 1: FILESYSTEM NAVIGATION, SEARCH, ARCHIVING & STREAMS                      |
| Path Resolution · Flag Comparisons (ls/grep) · Packaging (tar) · Redirection/tee  |
+----------------------------------------+------------------------------------------+
                                         | Process executes under user context
                                         v
+-----------------------------------------------------------------------------------+
| PILLAR 2: PROCESS FORENSICS, SIGNALS & JOB CONTROL                                |
| Process Snapshot (ps) · Targeting (pgrep) · Signal Escalation (15/9) · Job Control|
+----------------------------------------+------------------------------------------+
                                         | Process accesses local files
                                         v
+-----------------------------------------------------------------------------------+
| PILLAR 3: SECURITY, DAC PERMISSIONS & SCOPED SUDO                                 |
| The Traversal Law (r vs x) · Account Management · Sudoers Delegation · visudo -c  |
+----------------------------------------+------------------------------------------+
                                         | Process binds listening network socket
                                         v
+-----------------------------------------------------------------------------------+
| PILLAR 4: SOCKETS, SYSTEMD ORCHESTRATION & WEB SERVICES                           |
| Kernel Sockets (ss) · Journal Forensics (journalctl) · Systemd Units · nc & curl  |
+-----------------------------------------------------------------------------------+
```

---

## 2. Pillar 1: Filesystem Navigation, Search, Archiving & Streams

### 2.1 File Inodes & Path Traversal Context
Every filesystem object in Linux is represented by an inode storing core metadata: user ownership (UID), group ownership (GID), timestamps, file size, and permission bits. Directory inodes contain directory entries (dirents) mapping filenames to child inode numbers. When accessing any nested file, the kernel Virtual File System (VFS) validates path traversal permissions from root (`/`) down to the target.

### 2.2 `ls` Flag Mechanics
Different flag combinations alter whether `ls` reports file content, inode metadata, directory nodes, or size-sorted rankings.

| Command Invocation | What Is Displayed | What Is Hidden / Omitted | When to Use in Production |
| :--- | :--- | :--- | :--- |
| `ls /var/www/html` | Filenames only across columns. | Hidden dotfiles, permissions, owners, sizes, timestamps. | Quick visual verification of file presence. |
| `ls -la /var/www/html` | Detailed listing of all entries including `.` and `..`. | Nothing. Reveals hidden files and directory permissions. | Security audits and ownership investigations. |
| `ls -ld /var/www/html` | Detailed listing of the directory inode itself. | Contents inside the directory. | Auditing directory traversal bits (`x`) and directory ownership. |
| `ls -lhS /var/log` | Detailed listing sorted by file size descending (`-S`). | Chronological ordering. | Identifying files consuming disk capacity during disk space alerts. |

To inspect directory node permissions without listing its child contents, execute `ls` with the `-ld` flag:
```bash
# Flag '-l': Displays long-format permissions, ownership, and timestamps
# Flag '-d': Scopes inspection to the directory node itself rather than entering it
ls -ld /var/www/html
```

### 2.3 Text Filtering with `grep`
Production log triage requires precise text filtering to separate diagnostic anomalies from informational noise:
- `grep -i "pattern"`: Ignores case distinctions (e.g., matches `error`, `Error`, and `ERROR`).
- `grep -v "pattern"`: Inverts matching logic, excluding lines matching the pattern (useful for filtering routine health check noise).
- `grep -rnE "pattern1|pattern2"`: Searches recursively (`-r`), prefixes output with line numbers (`-n`), and evaluates extended regular expressions (`-E`) enabling the pipe (`|`) operator for OR logic.

To isolate critical error messages across nested log files with line numbers, execute a recursive extended regex query:
```bash
# Flag '-r': Searches subdirectories recursively
# Flag '-n': Prints 1-indexed line numbers
# Flag '-E': Enables extended regular expressions to evaluate OR logic ('|')
grep -rnE "WARN|CRITICAL" /var/log/
```

### 2.4 Real-Time File Finding & Archiving (`find` & `tar`)
The `find` utility searches the live filesystem hierarchy based on inode attributes rather than pre-built databases:
- `find <path> -type f -name "*.log"`: Restricts results strictly to regular files ending in `.log`.
- `find <path> -type f -perm 777`: Locates world-writable files for security audits.

To package and verify diagnostic logs for offline review without modifying originals, use `tar`:
```bash
# Create a gzip-compressed archive from a target directory
# Flags: '-c' = create; '-z' = gzip compression; '-v' = verbose progress; '-f' = archive filename; '-C' = change directory
tar -czvf /tmp/logs_archive.tar.gz -C /var/log .

# Inspect archive contents without extracting files to disk
# Flags: '-t' = table of contents; '-z' = gzip filter; '-v' = verbose; '-f' = archive filename
tar -tzvf /tmp/logs_archive.tar.gz
```

### 2.5 Standard Streams & The `sudo tee` Solution
Every process initializes with three standard file descriptors: `0` (`stdin`), `1` (`stdout`), and `2` (`stderr`).

![Standard Streams & Redirection](../assets/module1/linux_standard_streams.png)

#### Redirection Mechanics
- `>` overwrites standard output (FD 1).
- `>>` appends standard output (FD 1).
- `2>` redirects standard error (FD 2).
- `&>` redirects both standard output and standard error into a single destination.
- `2>&1` duplicates FD 2 onto FD 1, routing errors into the standard output stream.

#### The Sudo Redirection Trap
A common operational failure occurs when attempting to write to privileged files using sudo with redirection:
```bash
# This command fails with: bash: /etc/app.conf: Permission denied
sudo echo "config_value=true" > /etc/app.conf
```
**Why it fails:** The interactive shell evaluates and opens the redirection target (`> /etc/app.conf`) as the unprivileged invoking user *before* executing the `sudo` command.

**The Solution (`sudo tee`):** The `tee` utility reads from standard input and writes to disk. By piping output to `sudo tee`, the file write is performed by `tee` running with root privileges:
```bash
# Overwrite a protected configuration file using elevated privileges
echo "config_value=true" | sudo tee /etc/app.conf

# Append to a protected configuration file without truncating existing lines
echo "debug=false" | sudo tee -a /etc/app.conf
```

#### Here-Documents: Quoted vs Unquoted Delimiters
When streaming multiline configurations into files via `cat << EOF`:
- `<< EOF` (unquoted): Evaluates shell parameter substitutions (`$VAR`, `$(cmd)`).
- `<< 'EOF'` (quoted delimiter): Disables shell evaluation completely, preserving all strings literally. This is essential when authoring Nginx configs, sudoers rules, or shell scripts containing `$` symbols.

---

## 3. Pillar 2: Process Forensics, Signals & Job Control

### 3.1 Kernel Process States
A process transitions through multiple execution states inside the Linux kernel scheduler:
- **`R` (Running / Runnable):** Actively executing on a CPU core or queued in the CPU scheduler.
- **`S` (Interruptible Sleep):** Waiting for an event, timer, or I/O operation. Can be awakened by signals.
- **`D` (Uninterruptible Sleep):** Blocked waiting for synchronous device I/O (such as disk writes). Cannot be killed by any signal, including `SIGKILL`.
- **`Z` (Zombie / Defunct):** Terminated via `exit()`, but whose exit status has not yet been collected by its parent process via `waitpid()`. Consumes a process table slot but zero memory.

![Linux Process Lifecycle & Signals](../assets/module1/linux_process_lifecycle.png)

### 3.2 Process Inspection Styles: `ps aux` vs `ps -ef` vs `ps -eo`
Different listing modes provide distinct diagnostic views:
- **`ps aux` (BSD Style):** Displays memory and CPU percentage utilization (`%CPU`, `%MEM`, `VSZ`, `RSS`) and process execution states (`STAT`).
- **`ps -ef` (POSIX Style):** Displays parent-child lineage, explicitly including Parent Process ID (`PPID`) and launch time (`STIME`).
- **`ps -eo` (DevOps Custom Format):** Generates structured tabular output with explicit column definitions, ideal for parsing and triage:

```bash
# Display top CPU-consuming processes with custom columns
# Flags: '-e' = all processes; '-o' = custom columns; '--sort=-%cpu' = sort descending by CPU
ps -eo pid,ppid,user,%cpu,%mem,comm --sort=-%cpu | head -n 10
```

### 3.3 Process Targeting with `pgrep`
Avoid constructing brittle grep pipelines such as `ps aux | grep worker | grep -v grep | awk '{print $2}'`. Use `pgrep` to query the kernel process table directly:
```bash
# Match against full command-line arguments and display process name alongside PID
pgrep -fl "db_sync_worker"

# Extract pure numerical PID for scripting and signal delivery
pgrep -f "db_sync_worker"
```

### 3.4 Signal Escalation Protocol
Processes communicate operational directives via POSIX signals. In production troubleshooting, engineers enforce a sequential escalation protocol:

```text
Graceful Termination (SIGTERM: 15)  ---> Grace Period (2-5s) ---> Unconditional Abort (SIGKILL: 9)
```

| Signal | Number | Catchable? | Technical Function & Operational Rationale |
| :--- | :--- | :--- | :--- |
| `SIGHUP` | `1` | Yes | Terminal hangup. Daemons repurpose this signal to re-read configuration files without dropping connections. |
| `SIGINT` | `2` | Yes | Keyboard interrupt (`Ctrl+C`). Prompts process to terminate interactively. |
| `SIGTERM` | `15` | Yes | Standard polite termination. Notifies application to flush file buffers, close database handles, and exit cleanly. |
| `SIGKILL` | `9` | **NO** | Unconditional kernel abort. Handled directly by the kernel; cannot be caught, blocked, or ignored. Forcibly destroys process address space. |

To terminate a process cleanly, dispatch `SIGTERM` first, grant a brief grace period, verify whether the PID still exists, and escalate to `SIGKILL` only if the process remains stuck:
```bash
# Step 1: Dispatch polite termination request
kill -15 2840

# Step 2: Grant grace period for buffer flushing
sleep 2

# Step 3: Check if PID is still in process table
ps -p 2840

# Step 4: Escalate to unconditional abort if still present
kill -9 2840
```

### 3.5 Job Control & Session Persistence
When running long-lived administrative tasks in an interactive shell:
- Append `&` to run a command in the background subshell.
- `jobs -l`: Lists active background jobs with corresponding PIDs.
- `fg %1`: Brings Job 1 to the terminal foreground.
- `Ctrl+Z`: Suspends the active foreground process (transitioning it to `STOP` state).
- `bg %1`: Resumes suspended Job 1 in the background.
- `nohup <cmd> > file.log 2>&1 &`: Disconnects the process from the controlling terminal's `SIGHUP` signal so execution persists after SSH logout.

---

## 4. Pillar 3: Security, DAC Permissions & Scoped Sudo

### 4.1 DAC Permission Bits & Octal Math
Every file and directory possesses 9 standard permission bits divided into three user tiers:
- **Owner (`u`):** The user account that owns the file.
- **Group (`g`):** Members of the group assigned to the file.
- **Others (`o`):** All other authenticated users on the system.

Each tier evaluates three discrete access bits: Read (`r` = 4), Write (`w` = 2), and Execute (`x` = 1).

| Octal | Symbolic | Applicable Target | Operational Context & Least Privilege |
| :--- | :--- | :--- | :--- |
| `755` | `rwxr-xr-x` | Directories & Executables | Owner has full control; group and others can read and traverse. Standard web document roots. |
| `750` | `rwxr-x---` | Team Directories | Owner has full control; group can enter and read; outside users blocked completely. |
| `644` | `rw-r--r--` | Public / Web Assets | Owner reads/writes; group and others read-only. Standard configuration files. |
| `640` | `rw-r-----` | Confidential Data | Owner reads/writes; authorized group reads; others have zero access. |
| `440` | `r--r-----` | Sudoers Drop-in Files | Read-only for root and shadow group; zero write permissions for anyone. Sudo rejects files with loose permissions. |

### 4.2 The Golden Law of Directory Traversal
The meaning of permission bits changes fundamentally between files and directories:

| Permission Bit | Function on a Regular File | Function on a Directory Node |
| :--- | :--- | :--- |
| **Read (`r`)** | Read file content (`cat`, `less`). | List entries inside the directory (`ls`). |
| **Write (`w`)** | Modify file content. | Create, rename, or delete files inside the directory. |
| **Execute (`x`)** | Execute file as a program/script. | **Traverse and search.** Enter directory (`cd`), access child inodes, and open files inside (`cat /dir/file`). |

> [!IMPORTANT]
> **The Traversal Rule:** To read a file at `/a/b/c.txt`, the user must possess the **execute bit (`x`)** on every parent directory in the path (`/`, `/a`, `/a/b`). If any parent directory lacks `x` for that user's category, the kernel rejects the read with `Permission denied`, even if the target file has `chmod 777`!

To audit permissions on every directory component along an entire path down to the file, execute `namei`:
```bash
# Traces permission bits and ownership across every parent directory in the chain
namei -l /opt/shared_app/config.env
```

### 4.3 Multi-User Administration
User accounts and groups isolate service daemons and team members:
- `sudo groupadd <group>`: Creates a system group.
- `sudo useradd -m -s /bin/bash <user>`: Creates an interactive user account with home directory (`-m`) and default bash shell (`-s`).
- `sudo usermod -aG <group> <user>`: Appends supplementary group membership (`-a` is critical to prevent wiping existing groups).
- `id <user>`: Inspects UID, primary GID, and supplementary groups.
- `sudo userdel -r <user>`: Deletes user and purges their home directory from disk.

### 4.4 Scoped Sudoers Delegation & Validation
Rather than granting unbounded root privileges, production environments delegate specific administrative commands via scoped drop-in files in `/etc/sudoers.d/`:

```text
# Syntax format:
<USER/GROUP> <HOST> = (<RUNAS>) NOPASSWD: <ABSOLUTE_COMMAND_PATH>
```

To configure scoped sudo permissions for a user to manage Nginx without entering a password, write the rule, enforce mode `440`, and validate syntax:
```bash
# Step 1: Create scoped drop-in configuration
cat << 'EOF' | sudo tee /etc/sudoers.d/dev_alex_nginx >/dev/null
dev_alex ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
EOF

# Step 2: Enforce strict mode 440 (sudo ignores files with loose permissions)
sudo chmod 440 /etc/sudoers.d/dev_alex_nginx

# Step 3: Validate syntax across all sudoers files before logging out
sudo visudo -c

# Step 4: Verify authorized commands as the target user
sudo -u dev_alex sudo -l
```

---

## 5. Pillar 4: Sockets, Systemd Orchestration & Web Services

### 5.1 Kernel Socket Forensics with `ss -tunlp`
Network sockets bridge external network packets to local processes. The `ss` utility interrogates kernel socket control blocks:
- `-t`: Filter for TCP sockets.
- `-u`: Filter for UDP sockets.
- `-n`: Numeric IP and port addresses (disables reverse DNS lookups to avoid hanging).
- `-l`: Listening sockets only (filters out transient established client connections).
- `-p`: Displays process name and PID owning the socket (requires `sudo`).

```bash
# Interrogate listening sockets and filter for Port 80
sudo ss -tunlp | grep :80
```

#### Socket Binding Addresses
- **`0.0.0.0:80`:** Listens across all local IPv4 network interfaces (loopback, private LAN, public WAN).
- **`127.0.0.1:80`:** Listens strictly on the loopback interface; unreachable by external clients.
- **`[::]:80`:** Listens across all IPv6 interfaces (and IPv4 if dual-stack is supported).

### 5.2 Multi-Layer Network Probing: Layer 4 (`nc`) vs Layer 7 (`curl`)
Diagnosing connectivity issues requires testing both transport connection establishment and application protocol handling:
- **Layer 4 Transport Probe (`nc -zv`):** Checks if the remote port completes the TCP 3-way handshake. Flags: `-z` (zero-I/O scan without sending data), `-v` (verbose output).
- **Layer 7 Application Probe (`curl -I`):** Sends an HTTP HEAD request to inspect application response status codes and server headers.

To isolate whether an outage is caused by a listening port failure or an application crash, test both layers:
```bash
# Layer 4 TCP connection handshake test
nc -zv 127.0.0.1 80

# Layer 7 HTTP application response header test
curl -I http://127.0.0.1:80
```

### 5.3 Automated Health Checks & Telemetry with `curl`
To extract pure HTTP status codes for monitoring scripts without capturing HTML bodies, execute `curl` with formatting flags:
```bash
# Returns HTTP status code integer only (e.g. 200 or 502)
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:80
```

For performance benchmarking and SRE latency breakdown:
```bash
# Outputs connection timing telemetry breakdown
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nDNS Lookup: %{time_namelookup}s\nTCP Connect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" http://127.0.0.1:80
```

### 5.4 Systemd Service Lifecycle & Unit Masking
Systemd supervises background daemons through unit files:
- `sudo systemctl status <unit>`: Displays active execution state, PID, memory, and recent journal entries.
- `sudo systemctl restart <unit>`: Stops and restarts daemon (causes brief connection interruption).
- `sudo systemctl reload <unit>`: Dispatches `SIGHUP` to reload configuration without dropping active client connections.
- `systemctl is-active <unit>`: Returns exit code `0` if service is active and prints `active`.

#### Understanding Unit Masking (`systemctl mask`)
When a unit is masked (`sudo systemctl mask <unit>`), systemd links the unit file directly to `/dev/null`:
```text
/etc/systemd/system/nginx.service -> /dev/null
```
This is the strongest possible suppression in systemd: the service **cannot be started manually, cannot be started on boot, and cannot be activated by any dependent service or socket**. Attempting to start it returns:
`Failed to start nginx.service: Unit nginx.service is masked.`

To restore a masked service:
```bash
# Remove the symlink to /dev/null
sudo systemctl unmask nginx

# Reload systemd manager configuration from disk
sudo systemctl daemon-reload

# Restart the service and verify active state
sudo systemctl restart nginx
systemctl is-active nginx
```

### 5.5 Systemd Journal Forensics (`journalctl -xeu`)
When a service fails to start, query the systemd binary journal for explanatory context:
```bash
# Flags: '-x' = catalog explanations; '-e' = jump to end; '-u' = scope to unit; '--no-pager' = print to terminal
sudo journalctl -xeu nginx --no-pager
```

---

## 6. Real-World Production Incident Playbooks

### Playbook 1: CPU-Stuck Rogue Daemon & Signal Escalation
**Incident:** A background computation process enters an infinite loop, starving system resources.

To locate the rogue process, inspect parent-child lineage, and enforce the signal escalation protocol:
```bash
# Step 1: Identify the rogue process PID sorted by CPU
ps -eo pid,ppid,user,%cpu,comm --sort=-%cpu | head -n 5

# Step 2: Query process using pgrep
pgrep -fl "worker"

# Step 3: Dispatch polite termination request
kill -15 <PID>
sleep 2

# Step 4: Verify whether process is still active
ps -p <PID>

# Step 5: Escalate to unconditional abort if unresponsive
kill -9 <PID>
```

### Playbook 2: The Directory Traversal Lockout (Missing `+x` Trap)
**Incident:** A web service returns `403 Forbidden` or `Permission denied` when accessing a file whose permissions are `644`.

To identify missing traversal permissions along the directory path and restore access:
```bash
# Step 1: Audit permissions across every parent directory
namei -l /var/www/html/index.html

# Step 2: Notice if any parent directory has mode 644 (drw-r--r--) lacking execute bits
ls -ld /var/www/html

# Step 3: Grant the mandatory directory traversal execution bit
sudo chmod 755 /var/www/html

# Step 4: Verify unprivileged user can now read the asset
sudo -u www-data cat /var/www/html/index.html > /dev/null
```

### Playbook 3: "Address Already in Use" (Errno 98) Port Collision
**Incident:** Nginx fails to start because a rogue process holds TCP Port 80.

To triage the collision, identify the rogue process holding the port, and restore the service:
```bash
# Step 1: Query journal to verify port collision error
sudo journalctl -xeu nginx --no-pager | grep "Address already in use"

# Step 2: Identify rogue PID holding Port 80
sudo ss -tunlp | grep :80

# Step 3: Terminate the rogue listener process
sudo kill -9 <ROGUE_PID>

# Step 4: Start Nginx cleanly and verify active state
sudo systemctl start nginx
systemctl is-active nginx
```

### Playbook 4: Systemd Mask & Failed State Recovery
**Incident:** Nginx fails to start with `Unit nginx.service is masked`.

To unmask the unit file, clear failed state counters, and re-launch the daemon:
```bash
# Step 1: Verify unit is masked
systemctl status nginx

# Step 2: Remove mask symlink
sudo systemctl unmask nginx

# Step 3: Reload systemd configuration cache
sudo systemctl daemon-reload

# Step 4: Clear internal failure counters
sudo systemctl reset-failed nginx

# Step 5: Start service and verify active state
sudo systemctl start nginx
systemctl is-active nginx
```

---

## 7. The Outside-In Troubleshooting Methodology

When diagnosing an unreachable service in production, follow the systematic **Outside-In** diagnostic framework from the external client perspective down to local filesystem permissions:

```text
1. EXTERNAL CLIENT TEST (Layer 7):
   curl -I http://<SERVER_IP>:80
   Does client receive HTTP 200 OK, an HTTP error (502/403), or connection timeout?
           |
           v
2. TRANSPORT PROBING (Layer 4):
   nc -zv <SERVER_IP> 80
   Does the TCP 3-way handshake succeed (open) or fail (refused/timeout)?
           |
           v
3. KERNEL SOCKET LISTENERS:
   sudo ss -tunlp | grep :80
   Is a process bound and listening on Port 80? What PID owns it?
           |
           v
4. SERVICE LIFECYCLE & STATE:
   sudo systemctl status nginx
   Is the service active, inactive, failed, or masked?
           |
           v
5. CONFIGURATION VALIDATION:
   sudo nginx -t
   Are there syntax errors in configuration files?
           |
           v
6. SYSTEM & SERVICE LOGS:
   sudo journalctl -xeu nginx --no-pager
   What specific kernel error or fatal exception caused the failure?
           |
           v
7. FILESYSTEM & DIRECTORY TRAVERSAL:
   namei -l /var/www/html/index.html
   Does the web daemon possess 'x' traversal bits on all parent directories?
```
