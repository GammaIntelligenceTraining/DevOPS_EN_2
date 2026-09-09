# Module 2: Solutions & Instructor Answer Key

This document contains the complete solutions, command sequences, expected outputs, and quick verification checkpoints for **Module 2: Security & Access Control**.

---

## Homework Solutions

### Task 1: Role-Based Account Onboarding

#### Step-by-Step Commands
```bash
# 1. Create analyst account with dedicated home directory and bash shell
sudo useradd -m -s /bin/bash analyst_bob

# 2. Set temporary password
sudo passwd analyst_bob
# (Enter password: e.g., Password123!)

# 3. Create supplementary analytics group
sudo groupadd analytics

# 4. Append user to analytics group (must use -a to prevent group eviction)
sudo usermod -aG analytics analyst_bob

# 5. Verify identity and membership
id analyst_bob
grep analyst_bob /etc/passwd
```

#### Expected Deliverable Outputs
- **`id analyst_bob`:**
  ```text
  uid=1001(analyst_bob) gid=1001(analyst_bob) groups=1001(analyst_bob),1002(analytics)
  ```
- **`grep analyst_bob /etc/passwd`:**
  ```text
  analyst_bob:x:1001:1001::/home/analyst_bob:/bin/bash
  ```

---

### Task 2: Isolated Team Workspace Collaboration

#### Step-by-Step Commands
```bash
# 1. Create shared analytics directory
sudo mkdir -p /opt/analytics_data

# 2. Assign group ownership to analytics
sudo chgrp analytics /opt/analytics_data

# 3. Set restrictive permissions: Owner & Group full (7), Others zero access (0) -> 770
sudo chmod 770 /opt/analytics_data

# 4. Switch to analyst_bob and create report file
sudo su - analyst_bob
echo "Revenue,Churn,Growth" > /opt/analytics_data/q1_metrics.csv
exit

# 5. Inspect directory and file permissions
ls -ld /opt/analytics_data
ls -l /opt/analytics_data/q1_metrics.csv

# 6. Verify that a user outside the analytics group cannot enter or list
# (Testing as regular student user without sudo):
ls -la /opt/analytics_data
# Expected: ls: cannot open directory '/opt/analytics_data': Permission denied
```

#### Expected Deliverable Outputs
- **`ls -ld /opt/analytics_data`:**
  ```text
  drwxrwx--- 2 root analytics 4096 Sep  7 10:00 /opt/analytics_data
  ```
- **`ls -l /opt/analytics_data/q1_metrics.csv`:**
  ```text
  -rw-rw-r-- 1 analyst_bob analyst_bob 21 Sep  7 10:05 /opt/analytics_data/q1_metrics.csv
  ```
- **Unprivileged rejection test:**
  ```text
  ls: cannot open directory '/opt/analytics_data': Permission denied
  ```

---

### Task 3: Granular File Permissions & Secret Tokens

#### Step-by-Step Commands
```bash
# 1. Create data processing script in home directory
echo "echo 'Processing quarterly financial dataset...'" > ~/process_data.sh

# 2. Grant execute permission to Owner and Group using symbolic mode
chmod ug+x,o-x ~/process_data.sh

# 3. Create mock database credentials file
cat << 'EOF' > ~/db_credentials.env
DB_USER=analytics_prod
DB_TOKEN=9f8e7d6c5b4a3210
EOF

# 4. Restrict to owner-only read/write using octal 600
chmod 600 ~/db_credentials.env

# 5. Change ownership of both files to analyst_bob:analytics
sudo chown analyst_bob:analytics ~/process_data.sh ~/db_credentials.env

# 6. Verify permissions and ownership
ls -l ~/process_data.sh ~/db_credentials.env
```

#### Expected Deliverable Outputs
- **`ls -l ~/process_data.sh ~/db_credentials.env`:**
  ```text
  -rw------- 1 analyst_bob analytics 46 Sep  7 10:10 /home/student/db_credentials.env
  -rwxr-xr-- 1 analyst_bob analytics 49 Sep  7 10:12 /home/student/process_data.sh
  ```
  *(Note: Owner has `rwx`, Group has `r-x` or `rwx`, Others have `r--` or `---` without execute)*

---

### Task 4: Least-Privilege Sudo Delegation for Service Management

#### Step-by-Step Commands
```bash
# 1. Create drop-in sudoers rule for the analytics group using visudo
echo "%analytics ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart cron, /usr/bin/systemctl status cron" | sudo EDITOR='tee' visudo -f /etc/sudoers.d/analytics_team

# 2. Validate sudoers syntax
sudo visudo -cf /etc/sudoers.d/analytics_team
# Expected: /etc/sudoers.d/analytics_team: parsed OK

# 3. Switch to analyst_bob and audit allowed privileges
sudo su - analyst_bob
sudo -l

# 4. Test permitted command (succeeds without password)
sudo systemctl status cron

# 5. Test forbidden command (rejected by sudo security policy)
sudo cat /etc/shadow

exit
```

#### Expected Deliverable Outputs
- **`sudo visudo -cf /etc/sudoers.d/analytics_team`:**
  ```text
  /etc/sudoers.d/analytics_team: parsed OK
  ```
- **`sudo -l` (as `analyst_bob`):**
  ```text
  Matching Defaults entries for analyst_bob on devops-vm:
      env_reset, mail_badpass, ...

  User analyst_bob may run the following commands on devops-vm:
      (ALL) NOPASSWD: /usr/bin/systemctl restart cron, /usr/bin/systemctl status cron
  ```
- **`sudo cat /etc/shadow` (rejection):**
  ```text
  Sorry, user analyst_bob is not allowed to execute '/usr/bin/cat /etc/shadow' as root on devops-vm.
  ```

---

### Task 5: Cryptographic SSH Key Generation & Remote Audit

#### Step-by-Step Commands
```bash
# 1. Create .ssh directory for analyst_bob if not present
sudo mkdir -p /home/analyst_bob/.ssh

# 2. Generate Ed25519 keypair non-interactively
sudo ssh-keygen -t ed25519 -C "analyst_bob@finanalytics.internal" -f /home/analyst_bob/.ssh/id_ed25519 -N ""

# 3. Enforce strict permissions and ownership
sudo chown -R analyst_bob:analytics /home/analyst_bob/.ssh
sudo chmod 700 /home/analyst_bob/.ssh
sudo chmod 600 /home/analyst_bob/.ssh/id_ed25519
sudo chmod 644 /home/analyst_bob/.ssh/id_ed25519.pub

# 4. Verify directory listing
sudo ls -la /home/analyst_bob/.ssh

# 5. Check PermitRootLogin in sshd_config
grep -i "PermitRootLogin" /etc/ssh/sshd_config

# 6. Execute pre-flight syntax check
sudo sshd -t
echo $?

# 7. Review active sessions
who
w
```

#### Expected Deliverable Outputs
- **`ls -la /home/analyst_bob/.ssh`:**
  ```text
  drwx------ 2 analyst_bob analytics 4096 Sep  7 10:20 .
  -rw------- 1 analyst_bob analytics  419 Sep  7 10:20 id_ed25519
  -rw-r--r-- 1 analyst_bob analytics  108 Sep  7 10:20 id_ed25519.pub
  ```
- **`sudo sshd -t; echo $?`:**
  ```text
  0
  ```

---

## 30-Second Grading Checklist for Instructors

When reviewing student homework submissions, check these five critical indicators:

1. **Task 1 User & Group:** `id analyst_bob` shows UID 1001+ with membership in `analyst_bob` and `analytics`.
2. **Task 2 Directory Isolation:** `ls -ld /opt/analytics_data` shows `drwxrwx---` (**770**) owned by `root:analytics`.
3. **Task 3 Secret Mode:** `ls -l db_credentials.env` shows `-rw-------` (**600**) owned by `analyst_bob:analytics`.
4. **Task 4 Group Sudo Syntax:** `/etc/sudoers.d/analytics_team` uses `%analytics` group syntax restricting commands to `cron`, and `sudo cat /etc/shadow` is rejected.
5. **Task 5 SSH Key Permissions:** `/home/analyst_bob/.ssh/id_ed25519` has mode **600** and directory has mode **700**.
