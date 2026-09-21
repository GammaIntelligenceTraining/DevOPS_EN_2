# Module 4: Core Systems, Security & Network Consolidation — Homework Solutions & Reference Walkthroughs

Complete, verified reference implementations, command walkthroughs, and diagnostic triage logs for the Module 4 homework assignments and Stretch Challenge.

---

# Part 1: Core Practice Solutions

---

## Task 1: Process Triage & Signal Escalation Protocol

### Scenario Context
An automated database synchronization daemon (`db_sync_worker`) was left running in the background after an interrupted sync job. The worker process runs in an infinite subshell loop, appending heartbeat records to `/tmp/db_sync.log`.

### Reference Implementation & Walkthrough

Execute the following commands to simulate the orphaned synchronization worker, locate its PID and parent PID using custom formatted output, execute a polite termination request, and escalate to unconditional termination if needed:
```bash
# Step 1: Deploy and launch the background simulation worker script
# Creates an executable script, sets executable permissions, and detaches execution
cat << 'EOF' > /tmp/db_sync_worker
#!/bin/bash
while true; do
    echo "[$(date +%T)] DB_SYNC_HEARTBEAT - OK" >> /tmp/db_sync.log
    sleep 1
done
EOF
chmod +x /tmp/db_sync_worker
/tmp/db_sync_worker &
# Example Output: [1] 5422

# Step 2: Locate the worker process by name using pgrep
# Flag '-l': Displays the process executable name alongside its numeric PID
# (Flag '-f' can also be combined as '-fl' to match against full command arguments)
pgrep -l "db_sync_worker"
# Example Output: 5422 db_sync_worker

# Step 3: Inspect process details, user, and parent process ID (PPID) using ps
# Flag '-e': Selects all processes across all user accounts
# Flag '-o': Custom column layout specifying exact metrics needed for triage
# Flag '--sort=-%cpu': Sorts descending by CPU percentage (- prefix denotes descending)
# 'grep': Filters output specifically for the target worker binary
ps -eo pid,ppid,user,%cpu,comm --sort=-%cpu | grep -E "PID|db_sync_worker"
# Example Output:
#   PID  PPID USER     %CPU COMMAND
#  5422  5421 ubuntu    0.2 db_sync_worker

# Step 4: Verify active heartbeat log generation
# Flag '-n 5': Displays the 5 most recent heartbeat records
tail -n 5 /tmp/db_sync.log
# Example Output:
# [12:20:01] DB_SYNC_HEARTBEAT - OK
# [12:20:02] DB_SYNC_HEARTBEAT - OK
# [12:20:03] DB_SYNC_HEARTBEAT - OK
# [12:20:04] DB_SYNC_HEARTBEAT - OK
# [12:20:05] DB_SYNC_HEARTBEAT - OK

# Step 5: Dispatch SIGTERM (Signal 15) to request graceful shutdown
# Why Signal 15 first? Adheres to enterprise signal escalation protocol, giving daemons opportunity to flush buffers
kill -15 5422

# Step 6: Grant a 3-second grace period and verify process status
sleep 3
ps -p 5422

# Step 7: Escalate to SIGKILL (Signal 9) if the process persisted
# Signal 9: Direct kernel intervention; forcibly halts process and unconditionally reclaims memory allocations
kill -9 5422 2>/dev/null || true

# Step 8: Final verification confirming PID is completely purged from kernel process table
ps -p 5422 || echo "Process cleanly eradicated."

# Confirm log writes have permanently ceased
tail -n 3 /tmp/db_sync.log
```

### Verification Criteria
- Student correctly identified PID and PPID.
- Student followed the escalation protocol: dispatched signal `15` first, evaluated process state, and only escalated to signal `9` if necessary.
- Confirmed that `/tmp/db_sync.log` stopped receiving new timestamp ticks.

---

## Task 2: Multi-User Identity & Directory Traversal Hardening

### Scenario Context
The data engineering platform requires a secured drop-box directory at `/opt/analytics/incoming/` to receive daily data imports. Group `analysts` must collaborate with read/write access, the web server service account (`www-data`) must read reports, and unprivileged users must be blocked. Developer `dev_jordan` needs scoped sudo privileges to reload Nginx and test syntax.

### Reference Implementation & Walkthrough

Execute the following commands to create the group and user, establish the hardened directory tree, configure collaborative permissions, and deploy the scoped sudoers drop-in:
```bash
# Step 1: Create system security group and developer user account
# 'groupadd': Creates security group 'analysts' in /etc/group
# 'useradd -m': Generates home directory (/home/dev_jordan)
# 'useradd -s /bin/bash': Assigns standard interactive bash shell
sudo groupadd analysts
sudo useradd -m -s /bin/bash dev_jordan

# Step 2: Append group memberships safely
# Flags '-aG': '-a' (append) is CRITICAL; omitting '-a' would purge existing group memberships!
sudo usermod -aG analysts dev_jordan
sudo usermod -aG analysts www-data

# Verify group memberships using 'id'
# 'id': Queries NSS database to verify UID, primary GID, and supplementary groups
id dev_jordan
# Output: uid=1002(dev_jordan) gid=1002(dev_jordan) groups=1002(dev_jordan),1003(analysts)
id www-data
# Output includes: groups=33(www-data),1003(analysts)

# Step 3: Establish directory structure and mock data artifact
# Flag '-p': Idempotent directory creation
sudo mkdir -p /opt/analytics/incoming

# 'tee': Splits standard input to write to a root-owned file while preserving elevated sudo permissions
cat << 'EOF' | sudo tee /opt/analytics/incoming/quarterly_report.csv >/dev/null
metric,value,timestamp
active_users,14520,2026-09-13T12:00:00Z
api_requests,892100,2026-09-13T12:00:00Z
EOF

# Step 4: Reconcile ownership and collaborative traversal permissions
# Flag '-R': Recursively updates ownership across the entire directory branch
sudo chown -R dev_jordan:analysts /opt/analytics

# Mode '750' = drwxr-x--- (Owner full control, group members can traverse/list, others blocked)
sudo chmod 750 /opt/analytics
sudo chmod 750 /opt/analytics/incoming

# Mode '640' = -rw-r----- (Owner read/write, group members read, others blocked)
sudo chmod 640 /opt/analytics/incoming/quarterly_report.csv

# Verify directory and file permissions
# Flag '-ld': Inspects directory node attributes directly without listing interior files
ls -ld /opt/analytics /opt/analytics/incoming
# Output: drwxr-x--- 3 dev_jordan analysts 4096 ... /opt/analytics

# Flag '-l': Long format verifying permissions, owner, and group on file
ls -l /opt/analytics/incoming/quarterly_report.csv
# Output: -rw-r----- 1 dev_jordan analysts 85 ... quarterly_report.csv

# Step 5: Test the Traversal Law across user identities
# Test 5A: dev_jordan (Owner) can read file
sudo -u dev_jordan cat /opt/analytics/incoming/quarterly_report.csv

# Test 5B: www-data (Group Member) can read file
sudo -u www-data cat /opt/analytics/incoming/quarterly_report.csv

# Test 5C: nobody (Other) is strictly blocked
sudo -u nobody cat /opt/analytics/incoming/quarterly_report.csv 2>&1 || echo "Access blocked as expected"

# Audit full path traversal permissions with namei
namei -l /opt/analytics/incoming/quarterly_report.csv

# Step 6: Deploy scoped sudoers drop-in file
# 'tee': Reads Here-Document from stdin and writes with root privileges to /etc/sudoers.d/
# 'NOPASSWD:': Authorizes targeted commands without prompting for user password
cat << 'EOF' | sudo tee /etc/sudoers.d/analysts_maintenance >/dev/null
dev_jordan ALL=(root) NOPASSWD: /usr/bin/systemctl reload nginx, /usr/sbin/nginx -t
EOF

# Enforce strict 440 read-only mode (r--r-----)
# Why 440? Sudo ignores configuration drop-ins if they possess write bits for group or world
sudo chmod 440 /etc/sudoers.d/analysts_maintenance

# Validate syntax across all sudoers files
# Flag '-c': Check syntax mode; validates file format without opening an editor
sudo visudo -c
# Output: /etc/sudoers.d/analysts_maintenance: parsed OK

# Step 7: Test privilege delegation as dev_jordan
# Flag '-u dev_jordan': Assumes identity context of target user
# Flag '-l': Lists permitted administrative commands for that account
sudo -u dev_jordan sudo -l
# Output includes: (root) NOPASSWD: /usr/bin/systemctl reload nginx, /usr/sbin/nginx -t

# Verify that dev_jordan can execute nginx syntax validation without password prompt
sudo -u dev_jordan sudo /usr/sbin/nginx -t
# Output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Verification Criteria
- `id dev_jordan` and `id www-data` confirm membership in group `analysts`.
- Directory `/opt/analytics/incoming` has mode `750` owned by `dev_jordan:analysts`.
- File `quarterly_report.csv` has mode `640` readable by `dev_jordan` and `www-data`, blocked for `nobody`.
- Sudoers file `/etc/sudoers.d/analysts_maintenance` has mode `440` and passes `sudo visudo -c`.

---

## Post-Homework Environment Reset

Execute these cleanup commands on student practice machines to restore a clean baseline:
```bash
# 1. Clean up Task 1 test files
sudo rm -f /tmp/db_sync_worker /tmp/db_sync.log

# 2. Clean up Task 2 directories, users, and sudoers drop-in
sudo rm -rf /opt/analytics /etc/sudoers.d/analysts_maintenance
sudo loginctl terminate-user dev_jordan 2>/dev/null || true
sudo pkill -u dev_jordan -9 2>/dev/null || true
sudo userdel -r dev_jordan 2>/dev/null || true
sudo gpasswd -d www-data analysts 2>/dev/null || true
sudo groupdel analysts 2>/dev/null || true
```
