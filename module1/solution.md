# Module 1: Solutions & Instructor Answer Key

This document contains the complete solutions, command sequences, expected outputs, and quick verification checkpoints for **Module 1: Fundamentals, Tooling & Processes**.

---

## Homework Solutions

### Task 1: System Baseline Audit

#### Commands & Sample Outputs
1. **Print current username:**
   ```bash
   whoami
   ```
   *Sample Output:* `student` or `ubuntu`

2. **Display kernel version and architecture:**
   ```bash
   uname -a
   ```
   *Sample Output:* `Linux devops-vm 6.8.0-40-generic #40-Ubuntu SMP PREEMPT_DYNAMIC ... x86_64 GNU/Linux`

3. **Discover internal IP address:**
   ```bash
   hostname -I
   ```
   *Sample Output:* `172.28.14.92` (or `192.168.64.4`)

4. **Verify PID 1:**
   ```bash
   ps -p 1 -o comm=
   ```
   *Expected Output:* `systemd` (or `init`)

---

### Task 2: Directory Architecture & File Operations

#### Step-by-Step Command Walkthrough
1. **Idempotent Directory Creation:**
   ```bash
   mkdir -p ~/ops_practice/production/logs
   ```

2. **Create hidden file:**
   ```bash
   touch ~/ops_practice/production/.deploy_token
   ```

3. **Copy hidden file into logs:**
   ```bash
   cp ~/ops_practice/production/.deploy_token ~/ops_practice/production/logs/
   ```

4. **Path Navigation:**
   ```bash
   cd ~/ops_practice/production/logs
   # Absolute jump to /etc:
   cd /etc
   # Relative jump back:
   cd ../home/<username>/ops_practice/production/logs
   # (Or shortcut):
   cd ~/ops_practice/production/logs
   ```

---

### Task 3: Text Editing with Nano & Content Filtering

#### Steps & Sample Outputs
1. **Edit with Nano:**
   ```bash
   nano ~/ops_practice/production/app.conf
   ```
   *(Student writes the 3 lines, presses `Ctrl+O` then `Enter`, then `Ctrl+X`)*.

2. **Display with `cat`:**
   ```bash
   cat ~/ops_practice/production/app.conf
   ```
   *Expected Output:*
   ```text
   ENVIRONMENT=production
   LOG_LEVEL=debug
   PORT=8080
   ```

3. **Filter with `grep`:**
   ```bash
   grep "PORT" ~/ops_practice/production/app.conf
   ```
   *Expected Output:*
   ```text
   PORT=8080
   ```

---

### Task 4: Process Management & Job Control

#### Step-by-Step Command Walkthrough
1. **Start 400-second sleep in background:**
   ```bash
   sleep 400 &
   ```
   *Returns:* `[1] <PID_A>`

2. **Start 600-second sleep in foreground, suspend, and push to background:**
   ```bash
   sleep 600
   ```
   Press `Ctrl+Z` (Terminal outputs: `[2]+ Stopped sleep 600`).
   ```bash
   bg %2
   ```

3. **Check active jobs (`jobs` output):**
   ```bash
   jobs
   ```
   *Expected Deliverable Output:*
   ```text
   [1]-  Running                 sleep 400 &
   [2]+  Running                 sleep 600 &
   ```

4. **Find PID of 400-second process:**
   ```bash
   ps aux | grep "sleep 400" | grep -v grep
   ```

5. **Terminate by PID:**
   ```bash
   kill <PID_A>
   ```

6. **Terminate remaining by name:**
   ```bash
   pkill sleep
   ```

---

### Task 5: Package Management & Tooling Challenge (`apt`)

#### Commands & Sample Outputs
1. **Update package index:**
   ```bash
   sudo apt update
   ```

2. **Search for `tree`:**
   ```bash
   apt search tree
   ```

3. **Install `tree`:**
   ```bash
   sudo apt install -y tree
   ```

4. **Generate directory tree output:**
   ```bash
   tree -a ~/ops_practice
   ```

*Expected Deliverable Output:*
```text
/home/student/ops_practice
└── production
    ├── .deploy_token
    ├── app.conf
    └── logs
        └── .deploy_token

2 directories, 3 files
```

---