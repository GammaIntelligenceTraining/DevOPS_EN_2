# Module 4: Core Systems, Security & Network Consolidation — Homework Assignment

Practical laboratory assignments covering process forensics, discretionary access control, network socket auditing, and an unscripted multi-tier production triage challenge.

---

## Assignment Overview

This homework assignment consolidates the core pillars of Linux systems administration through practical, step-by-step laboratory tasks:
- **Task 1: Process Triage & Signal Escalation Protocol** (The Stale Database Sync Worker)
- **Task 2: Multi-User Identity & Directory Traversal Hardening** (The Analytics Ingestion Pipeline)

---

## Task 1: Process Triage & Signal Escalation Protocol

### Scenario
An automated database synchronization daemon (`db_sync_worker`) was left running in the background after an interrupted staging sync job. The worker process continuously executes in a loop, appending timestamp heartbeat records to `/tmp/db_sync.log`. Your objective is to locate the rogue worker, determine its process lineage, and execute a controlled signal escalation to terminate it cleanly without leaving orphaned subshells.

### Requirements & Guided Steps

1. **Simulation Setup:**
   Execute this copy-paste block to generate the simulated worker script, grant execution permissions, and launch it in the background:
   ```bash
   cat << 'EOF' > /tmp/db_sync_worker
   #!/bin/bash
   while true; do
       echo "[$(date +%T)] DB_SYNC_HEARTBEAT - OK" >> /tmp/db_sync.log
       sleep 1
   done
   EOF
   chmod +x /tmp/db_sync_worker
   /tmp/db_sync_worker &
   ```

2. **Process Discovery & Lineage:**
   - Use `pgrep` with process name listing (`-l`) to locate the process: `pgrep -l "db_sync_worker"` (or use `-fl` to match against full command arguments).
   - Use `ps` with custom formatting (`-eo`) to inspect its PID, parent process ID (`PPID`), running user, CPU utilization, and executable command name.
   - Note the numeric PID of the worker process.

3. **Activity Verification:**
   - View the last 5 entries of `/tmp/db_sync.log` to confirm that the worker is actively writing heartbeat records to disk:
   ```bash
   tail -n 5 /tmp/db_sync.log
   ```

4. **Signal Escalation Protocol:**
   - **Step 4A (Polite Termination Request):** Dispatch `SIGTERM` (signal 15) to the isolated PID. Allow a 3-second grace period for the worker to clean up its file handles.
   - **Step 4B (Process Table Verification):** Inspect whether the process exited using `ps -p <PID>`.
   - **Step 4C (Escalation to Unconditional Abort):** If the process is still running or unresponsive, escalate to `SIGKILL` (signal 9) to force immediate kernel reclamation.
   - **Step 4D (Final Confirmation):** Verify that the PID is completely removed from the process table (`ps -p <PID>` returns an error or empty result) and confirm that `/tmp/db_sync.log` has stopped receiving new lines.

### Technical Reference & Syntax Reminder

| Command / Flag | Technical Function | Diagnostic Purpose |
| :--- | :--- | :--- |
| `pgrep -fl <name>` | Matches pattern against process names and full command arguments. | Rapid discovery of target process ID without spawning grep pipelines. |
| `ps -eo pid,ppid,user,%cpu,comm` | Displays custom process table columns. | Pinpoints parent PID (PPID) and CPU utilization for triage. |
| `kill -15 <PID>` | Dispatches `SIGTERM` (Signal 15). | Polite termination request; gives process opportunity to flush buffers and close files. |
| `kill -9 <PID>` | Dispatches `SIGKILL` (Signal 9). | Unconditional kernel abort; immediately frees process memory without notifying process. |
| `ps -p <PID>` | Scopes process lookup strictly to target PID. | Non-zero exit code confirms process is no longer active in the kernel process table. |

### Deliverables
- Terminal session log showing:
  1. The command used to locate the running `db_sync_worker` and its parent PID.
  2. The dispatch of `kill -15` followed by process status inspection.
  3. The dispatch of `kill -9` (if needed) and final confirmation that the process is purged.
  4. Final output of `tail -n 3 /tmp/db_sync.log` showing timestamps no longer incrementing.

---

## Task 2: Multi-User Identity & Directory Traversal Hardening

### Scenario
The data engineering platform requires a secure ingestion drop-box at `/opt/analytics/incoming/` to receive daily data imports. The directory structure must satisfy these strict security constraints:
- Members of the `analysts` group must have read and write permissions on directory contents.
- The web server daemon account (`www-data`) must have read-only access to inspect reports.
- All other unprivileged users on the host must be completely blocked from listing or entering the directory.
- Developer `dev_jordan` must be authorized to reload Nginx configuration files without entering a password.

### Requirements & Guided Steps

1. **User & Group Administration:**
   - Create a dedicated system group named `analysts`.
   - Create a developer account named `dev_jordan` configured with a home directory and default bash shell (`/bin/bash`).
   - Add `dev_jordan` to the `analysts` supplementary group using `usermod -aG` (remember that omitting `-a` would purge existing group memberships).
   - Add the web server service account (`www-data`) to the `analysts` group so it shares group privileges.

2. **Directory Tree & Traversal Permission Hardening:**
   - Create the directory hierarchy `/opt/analytics/incoming/`.
   - Generate a mock data artifact `/opt/analytics/incoming/quarterly_report.csv` containing sample comma-separated values:
   ```text
   metric,value,timestamp
   active_users,14520,2026-09-13T12:00:00Z
   api_requests,892100,2026-09-13T12:00:00Z
   ```
   - Reconcile ownership so that `dev_jordan` is the file and directory owner, and `analysts` is the group owner (`dev_jordan:analysts`).
   - Configure directory permissions to mode `750` (`drwxr-x---`) on both `/opt/analytics/` and `/opt/analytics/incoming/`.
   - Configure file permissions on `quarterly_report.csv` to mode `640` (`-rw-r-----`).

3. **The Traversal Law Verification:**
   - Verify that `dev_jordan` can read the report file using `sudo -u dev_jordan cat ...`.
   - Verify that web server account `www-data` can read the report file using `sudo -u www-data cat ...`.
   - Verify that an unprivileged user outside the group (such as `nobody`) is strictly blocked with `Permission denied`.
   - Run `namei -l /opt/analytics/incoming/quarterly_report.csv` to inspect the full traversal permission bitmask down the directory path.

4. **Scoped Sudoers Delegation:**
   - Create a scoped configuration drop-in file at `/etc/sudoers.d/analysts_maintenance`.
   - Authorize user `dev_jordan` to run `/usr/bin/systemctl reload nginx` and `/usr/bin/nginx -t` with sudo without entering a password (`NOPASSWD:`).
   - Enforce strict permissions on the drop-in file by setting mode `440`.
   - Validate configuration syntax using `sudo visudo -c`.
   - Test permitted commands as `dev_jordan` using `sudo -u dev_jordan sudo -l`.

> [!NOTE]
> **Directory Traversal Law Reminder:**
> Remember that to read a file located at `/a/b/c.csv`, the user must have the **execute bit (`x`)** on every parent directory in the path (`/`, `/a`, `/a/b`). If any parent directory lacks `x` for that user's category (owner, group, or other), access will fail with `Permission denied` regardless of file permissions!

### Deliverables
- Terminal session log showing:
  1. Output of `id dev_jordan` and `id www-data` confirming supplementary group memberships.
  2. Long directory listing `ls -ld /opt/analytics/incoming` and `ls -l /opt/analytics/incoming/quarterly_report.csv`.
  3. Verification tests demonstrating successful file reads as `dev_jordan` and `www-data`, and access rejection as `nobody`.
  4. Output of `sudo visudo -c` confirming zero sudoers syntax errors.
  5. Output of `sudo -u dev_jordan sudo -l` displaying authorized Nginx commands.

---

## Lab Clean-up Commands

Execute these commands in your terminal to clean up all temporary files, accounts, and configurations created during this homework assignment:
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

