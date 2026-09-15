# Module 4: Core Systems, Security & Network Consolidation — Command Cheat Sheet

A quick-reference guide and flag comparison matrix consolidating commands, switches, and operational patterns covered during the Module 4 workshop.

---

## 1. Filesystem Navigation, Search, Archiving & Stream Redirection

Inspecting directory structures, filtering text patterns, packaging archives, and controlling shell streams.

| Command | Primary Flags & Syntax | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `ls` | `-la`<br>`-lh`<br>`-ld <path>`<br>`-lhS` | Lists files and directories. `-a` reveals hidden dotfiles; `-l` provides long metadata; `-ld` inspects directory inode permissions without entering; `-lhS` sorts by size descending. | `ls -ld /var/www/html`<br>`ls -lhS /var/log` |
| `cd` | `-`<br>`~` | Navigates the directory hierarchy. `cd -` toggles between current and previous working directories during multi-path administrative workflows. | `cd -` |
| `cat` | `-n`<br>`<< 'EOF'` | Displays file contents to stdout. `-n` numbers output lines. Used with Here-Documents (`<< 'EOF'`) to template configurations without variable expansion. | `cat -n /etc/nginx/nginx.conf`<br>`cat << 'EOF' \| sudo tee file.conf` |
| `head` | `-n <lines>` | Outputs the first N lines of a file or pipeline stream. Commonly used to truncate diagnostic outputs like `ps -eo` or `top` dumps. | `head -n 10 /var/log/syslog`<br>`ps -eo pid,comm \| head -n 5` |
| `tail` | `-n <lines>`<br>`-f` | Outputs the last N lines of a file. `-f` follows active file appends in real time, essential for live debugging of access and error logs. | `tail -f /var/log/nginx/error.log`<br>`tail -n 20 /tmp/db_sync.log` |
| `grep` | `-i`<br>`-v`<br>`-r`<br>`-n`<br>`-E` | Filters text matching regular expressions. `-i` ignores case; `-v` inverts matches to strip noise; `-r` searches recursively; `-n` displays line numbers; `-E` enables extended regex (`\|` OR logic). | `grep -rnE "WARN\|CRITICAL" /var/log/`<br>`grep -v "healthcheck" access.log` |
| `find` | `-type f\|d`<br>`-name "<glob>"`<br>`-perm <mode>` | Traverses filesystem hierarchy in real time. Locates configuration files, identifies specific extensions, or audits insecure permission modes (`-perm 777`). | `find /var/log -type f -name "*.log"`<br>`find /tmp -type f -perm 777` |
| `tar` | `-czvf <arch>`<br>`-tzvf <arch>`<br>`-C <dir>` | Manages compressed tar archives. `-c` creates archive; `-t` lists contents without extracting; `-z` compresses with gzip; `-v` verbose progress; `-f` filename; `-C` changes working directory. | `tar -czvf /tmp/logs.tar.gz -C /var/log .`<br>`tar -tzvf /tmp/logs.tar.gz` |
| `tee` | `sudo tee <path>`<br>`sudo tee -a <path>` | Reads standard input and writes to both standard output and files. Overcomes the sudo redirection trap by executing file writes with elevated privileges. `-a` appends. | `echo "worker_threads=4" \| sudo tee /etc/app.conf`<br>`echo "debug=true" \| sudo tee -a /etc/app.conf` |

### Side-by-Side Flag Comparison: `ls` Flag Mechanics

| Command Invocation | What Is Displayed | What Is Hidden / Omitted | When to Use in Production |
| :--- | :--- | :--- | :--- |
| `ls /var/www/html` | Filenames only in multiple columns. | Hidden dotfiles, permissions, owners, sizes, timestamps. | Quick visual verification of file presence. |
| `ls -la /var/www/html` | Detailed listing of all entries including `.` and `..`. | Nothing. Reveals hidden files and directory permissions. | Full security audits and ownership investigations. |
| `ls -ld /var/www/html` | Detailed listing of the directory inode itself. | Contents inside the directory. | Auditing directory traversal bits (`x`) and directory ownership. |
| `ls -lhS /var/log` | Detailed listing sorted by file size descending (`-S`). | Chronological ordering. | Identifying files consuming disk capacity during disk space alerts. |

### Standard Streams & Redirection Matrix

| Operator | Stream Affected | Technical Behavior | Enterprise Example |
| :--- | :--- | :--- | :--- |
| `>` | `stdout` (1) | Overwrites destination file with standard output. | `ps aux > /tmp/ps_snapshot.txt` |
| `>>` | `stdout` (1) | Appends standard output to destination file without truncating. | `date >> /tmp/deployment.log` |
| `2>` | `stderr` (2) | Redirects standard error stream only; `stdout` remains on terminal. | `nginx -t 2> /tmp/syntax_errors.log` |
| `&>` | `stdout` + `stderr` | Redirects both standard output and standard error into a single destination file. | `app_build.sh &> /tmp/build.log` |
| `2>&1` | `stderr` to `stdout` | Duplicates file descriptor 2 onto file descriptor 1. | `command > /tmp/out.log 2>&1` |
| `\|` | `stdout` to `stdin` | Streams output of first command directly into input of second command. | `ps -eo pid,%cpu,comm \| grep nginx` |

---

## 2. Process Forensics, Signals & Job Control

Monitoring process trees, filtering process lists, executing signal escalation protocols, and managing shell jobs.

| Command | Primary Flags & Syntax | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `ps` | `aux`<br>`-ef`<br>`-eo <cols>`<br>`-p <pid>` | Captures a snapshot of active processes. `aux` provides BSD memory/CPU metrics; `-ef` traces POSIX parent-child lineage (PPID); `-eo` customizes tabular output; `-p` scopes to specific PID. | `ps aux \| grep worker`<br>`ps -eo pid,ppid,user,%cpu,comm --sort=-%cpu`<br>`ps -p 4520` |
| `htop` | Interactive UI | Interactive real-time process viewer. Enables visual inspection of per-core CPU loads, memory/swap consumption, and process tree hierarchies. | `htop` |
| `pgrep` | `-fl <pattern>`<br>`-f <pattern>` | Searches kernel process table by pattern. `-f` matches full command-line arguments; `-l` displays process names alongside PIDs. Eliminates brittle `ps \| grep` pipelines. | `pgrep -fl "db_sync_worker"`<br>`pgrep -f "sleep 300"` |
| `kill` | `-15 <pid>`<br>`-9 <pid>`<br>`-1 <pid>` | Dispatches POSIX signals to processes by PID. `-15` (SIGTERM) requests graceful termination; `-9` (SIGKILL) forces unconditional kernel termination; `-1` (SIGHUP) triggers configuration reloads. | `kill -15 2840`<br>`kill -9 2840`<br>`kill -1 $(pgrep nginx)` |
| `sleep` | `<sec>` | Suspends execution for specified duration. Used to grant grace periods during signal escalation or launch background timers. | `sleep 2`<br>`sleep 300 &` |
| `jobs` | `-l` | Lists active background jobs managed by the current shell session. `-l` displays associated process IDs (PIDs) alongside job numbers. | `jobs -l` |
| `fg` | `%<job_id>` | Brings a background or suspended job into the terminal foreground. | `fg %1` |
| `bg` | `%<job_id>` | Resumes a suspended job (`Ctrl+Z`) in the background. | `bg %1` |
| `nohup` | `nohup <cmd> &` | Executes a command immune to `SIGHUP` terminal hangup signals. Ensures long-running processes continue running when SSH disconnects. | `nohup ./sync_data.sh > /tmp/sync.log 2>&1 &` |

### Side-by-Side Flag Comparison: `ps` Invocation Styles

| Command Invocation | Output Format & Columns | Primary Value in Enterprise Operations |
| :--- | :--- | :--- |
| `ps aux` | `USER, PID, %CPU, %MEM, VSZ, RSS, TTY, STAT, START, TIME, COMMAND` | Memory forensics (`VSZ`/`RSS`), CPU percentage, and execution states (`STAT`). |
| `ps -ef` | `UID, PID, PPID, C, STIME, TTY, TIME, CMD` | Parent Process ID (`PPID`) tracing to locate supervisor daemons. |
| `ps -eo pid,ppid,user,%cpu,comm --sort=-%cpu` | Minimalist custom output sorted by CPU utilization descending. | Automated monitoring scripts and resource bottleneck triage. |

---

## 3. User Administration, Ownership, Permissions & Scoped Sudo

Managing accounts, configuring Discretionary Access Control (DAC), evaluating directory traversal permissions, and delegating scoped administrative privileges.

| Command | Primary Flags & Syntax | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `groupadd` | `<group>` | Creates a new user group for collaborative access control. | `sudo groupadd devops_team` |
| `useradd` | `-m`<br>`-s <shell>`<br>`-G <groups>` | Creates user accounts. `-m` creates the home directory (`/home/<user>`); `-s` sets default login shell; `-G` specifies supplementary groups. | `sudo useradd -m -s /bin/bash -G devops_team dev_alex` |
| `usermod` | `-aG <groups>`<br>`-s <shell>` | Modifies existing user accounts. `-aG` appends supplementary groups without purging existing memberships (omitting `-a` clears unlisted groups!). | `sudo usermod -aG devops_team dev_alex`<br>`sudo usermod -s /bin/bash dev_alex` |
| `userdel` | `-r` | Deletes a user account. `-r` recursively removes the user's home directory and mail spool from disk. | `sudo userdel -r dev_alex` |
| `chmod` | `<octal>`<br>`+x`<br>`-R` | Modifies DAC permission bitmasks. Standard octal modes include `755` (dirs/binaries), `750` (private team dirs), `644` (public files), `640` (private files), `440` (sudoers drop-ins). `-R` applies recursively. | `sudo chmod 750 /opt/shared_app`<br>`sudo chmod 640 /opt/shared_app/config.env`<br>`sudo chmod 440 /etc/sudoers.d/dev_alex_nginx` |
| `chown` | `<user>:<group>`<br>`:<group>`<br>`-R` | Modifies user owner and group owner. `:<group>` changes group ownership without altering user owner. `-R` operates recursively across directory trees. | `sudo chown -R dev_alex:devops_team /opt/shared_app`<br>`sudo chown :devops_team /opt/shared_app` |
| `namei` | `-l <path>` | Evaluates and prints ownership and permission bits for every parent directory node from root `/` down to the target path. | `namei -l /var/www/html/index.html`<br>`namei -l /opt/shared_app/config.env` |
| `id` | `<user>` | Displays numerical UID, primary GID, and all supplementary GID memberships for target user. Used in verification and idempotency assertions. | `id dev_alex`<br>`id www-data` |
| `sudo` | `-u <user>`<br>`-l` | Executes commands with elevated privileges. `-u` executes within the context of a specified user identity; `-l` lists allowed administrative commands for that account. | `sudo -u dev_alex sudo -l`<br>`sudo -u dev_alex cat /opt/shared_app/config.env` |
| `visudo` | `-c` | Validates syntax of `/etc/sudoers` and drop-in files in `/etc/sudoers.d/`. `-c` runs in check-only mode without opening an interactive editor. | `sudo visudo -c` |

---

## 4. Sockets, Systemd Service Management & Diagnostic Probing

Interrogating kernel network sockets, managing daemon lifecycles, parsing journal logs, and executing Layer 4 and Layer 7 diagnostic probes.

| Command | Primary Flags & Syntax | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `ss` | `-t` (TCP)<br>`-u` (UDP)<br>`-n` (numeric)<br>`-l` (listen)<br>`-p` (process) | Interrogates kernel network socket structures. `-tunlp` displays active listening sockets, numerical IP/port bindings, and owning process names and PIDs. | `sudo ss -tunlp`<br>`sudo ss -tunlp \| grep :80` |
| `systemctl` | `status`<br>`start / stop`<br>`restart / reload`<br>`mask / unmask`<br>`is-active`<br>`daemon-reload`<br>`reset-failed` | Controls systemd units and daemon lifecycle. `reload` sends SIGHUP without downtime; `mask` symlinks unit to `/dev/null` to prevent activation; `unmask` restores unit; `is-active` returns zero exit status if running; `daemon-reload` refreshes disk configs; `reset-failed` clears unit error counters. | `sudo systemctl status nginx`<br>`sudo systemctl reload nginx`<br>`sudo systemctl mask nginx`<br>`sudo systemctl unmask nginx`<br>`systemctl is-active nginx` |
| `journalctl` | `-u <unit>`<br>`-xe`<br>`--no-pager` | Queries systemd binary event journal. `-u` scopes output to a specific service unit; `-x` enriches logs with explanatory catalog text; `-e` jumps to newest logs; `--no-pager` outputs directly to stdout. | `sudo journalctl -xeu nginx --no-pager` |
| `nginx` | `-t` | Validates Nginx configuration syntax across all included configuration files without restarting or reloading worker processes. | `sudo nginx -t` |
| `nc` | `-zv <host> <port>`<br>`-lk -p <port>` | Arbitrary TCP/UDP networking tool. `-zv` performs a zero-I/O Layer 4 TCP 3-way handshake to verify socket reachability. `-lk -p` creates a persistent listening socket for simulation. | `nc -zv 127.0.0.1 80`<br>`sudo nc -lk -p 80 &` |
| `curl` | `-I` (HEAD)<br>`-s` (silent)<br>`-o /dev/null`<br>`-w "<format>"` | Transfers data over HTTP/S. `-I` performs a HEAD request to inspect HTTP response headers; `-s -o /dev/null -w "%{http_code}\n"` extracts numerical status codes for automation. | `curl -I http://127.0.0.1:80`<br>`curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:80` |

### Side-by-Side Flag Comparison: Socket & Probing Utilities

| Diagnostic Need | Optimal Command & Flags | Rationale |
| :--- | :--- | :--- |
| **Verify Port 80 is listening** | `sudo ss -tunlp \| grep :80` | Displays socket state, local bind address, and owning process PID. |
| **Test port reachability (Layer 4)** | `nc -zv 127.0.0.1 80` | Performs TCP 3-way handshake without sending application payloads. |
| **Test web server response (Layer 7)** | `curl -I http://127.0.0.1:80` | Issues HTTP HEAD request to inspect HTTP status code and server headers. |
| **Extract HTTP code in automation** | `curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:80` | Returns pure numerical status (e.g., `200` or `502`) for automated assertions. |

### `curl -w` (--write-out) Status & Diagnostic Metrics Reference

The `-w` / `--write-out` option formats custom output strings using runtime transfer variables:

| Variable Token | Extracted Metric & Description | Diagnostic & Operational Purpose |
| :--- | :--- | :--- |
| `%{http_code}` | Numerical HTTP(S) response status code (e.g. `200`, `301`, `403`, `502`). | Automated health checks, uptime alerting, and CI/CD smoke test assertions. |
| `%{response_code}` | Protocol-agnostic numerical response code (alias to `http_code`). | Universal response code extraction across HTTP, FTP, and other transfer protocols. |
| `%{http_version}` | Negotiated HTTP version (e.g. `1.1`, `2`, `3`). | Verifies HTTP/2 or HTTP/3 ALPN protocol upgrades behind load balancers. |
| `%{content_type}` | `Content-Type` header value of the received response (e.g. `text/html`). | Validates that services return expected MIME types rather than HTML error pages. |
| `%{remote_ip}` | IP address of the remote server or proxy that answered the connection. | Confirms traffic reaches the intended server node or CDN edge rather than stale DNS targets. |
| `%{remote_port}` | Destination port number used for the connection (e.g. `80`, `443`, `8080`). | Confirms correct port forwarding, reverse proxy routing, or container port mapping. |
| `%{time_namelookup}` | Elapsed time (in seconds) from request invocation until DNS resolution completed. | Diagnoses slow DNS resolvers or misconfigured `/etc/resolv.conf`. |
| `%{time_connect}` | Elapsed time (in seconds) from start until TCP 3-way handshake completed. | Measures raw network transport latency between client and server. |
| `%{time_starttransfer}` | Elapsed time (in seconds) until the server delivered the first byte (TTFB). | Measures backend application processing time and server responsiveness. |
| `%{time_total}` | Total transaction duration (in seconds) from start until transfer fully finished. | End-to-end SLA benchmarking and latency monitoring. |
