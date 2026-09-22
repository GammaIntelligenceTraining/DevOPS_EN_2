# Module 6: Bash Automation & Infrastructure Scripting — Homework Assignment

A practical, scenario-driven laboratory assignment covering automated developer onboarding, multi-tier service health auditing, automation architecture reflection, and an automated backup integrity pipeline.

---

## Assignment Structure

This homework assignment offers two tiers of engagement:
- **Part 1: Core Practice (Recommended for all students):** Direct application of concepts covered in class—automated user onboarding with input validation, multi-tier web service health checks, and a formal engineering reflection on idempotency and defensive scripting.
- **Part 2: DevOps Stretch Challenge (Advanced application):** An unscripted, production-grade automation scenario requiring independent problem-solving—engineering an automated application backup and cryptographic verification pipeline with retention management.

---

# Part 1: Core Practice

---

## Task 1: Automated Developer Onboarding (`onboard.sh`)

### Scenario
Manual developer onboarding is slow, prone to human error, and creates inconsistent user environments. Your team has been tasked with building a standalone, idempotent automation script that takes a developer's username as an input argument and prepares their complete Linux workspace according to security standards.

### Technical Requirements
1. **Interpreter & Safety Directives:**
   - Use the standard Bash shebang.
   - Enforce strict error handling to ensure the script aborts immediately on command failure, uninitialized variable reference, or pipeline error.
2. **Privilege Verification:**
   - Verify that the script is executed with root privileges using the effective user ID parameter.
   - If executed by an unprivileged user, print an error message to standard error and exit with code `1`.
3. **Argument Validation:**
   - The script must require exactly one command-line argument representing the developer's target username.
   - If no argument or multiple arguments are supplied, display usage syntax and exit with code `2`.
4. **Idempotent User Account Creation:**
   - Query the system user database to determine if the target username already exists before attempting creation.
   - If the user already exists, print an informational message and skip creation without returning an error.
   - If the user does not exist, provision the account with a standard home directory and default shell `/bin/bash`.
5. **Workspace Scaffolding & Permissions:**
   - Ensure a dedicated project directory exists at `/home/<username>/projects`.
   - Reconcile ownership across the user's home and project directories so that the user and their primary group own all assets.
   - Configure directory permissions on `/home/<username>/projects` to mode `750` so only the developer and their group have read, write, and traversal rights.
6. **Package Dependencies:**
   - Idempotently verify that the `tree` and `curl` packages are installed on the host. If missing, install them non-interactively.

### Verification Steps
To verify your script, execute the test workflow below:
1. Attempt execution without `sudo` and verify exit status `1`.
2. Attempt execution without arguments and verify exit status `2`.
3. Execute `sudo ./onboard.sh dev_alex` and verify user creation, package verification, and directory scaffolding.
4. Execute `sudo ./onboard.sh dev_alex` a second time and verify that the script succeeds quietly without errors (idempotency).
5. Inspect the generated directory permissions and ownership using `ls -ld /home/dev_alex/projects`.

### Deliverables
- Complete source code of `onboard.sh`.
- Terminal output log demonstrating the five verification steps.

---

## Task 2: Multi-Tier Web Service Health Auditor (`service_audit.sh`)

### Scenario
The monitoring infrastructure requires a lightweight, standalone health check utility that evaluates the operational state of web servers. The script must assess three independent architectural tiers—process execution, network perimeter ingress, and application response—and exit with standardized return codes for automated monitoring systems.

### Technical Requirements
1. **Script Structure:**
   - Construct an executable shell script named `service_audit.sh`.
   - Implement modular functions or structured blocks corresponding to three operational tiers.
2. **Tier 1 — Process Execution (Daemon Health):**
   - Query systemd to verify that the `nginx` service is in an active state.
   - If the daemon is inactive or failed, output a diagnostic error and terminate immediately with exit code `1`.
3. **Tier 2 — Perimeter Ingress (Firewall Health):**
   - Query UFW status to verify that traffic on port `80/tcp` is actively allowed.
   - If port `80` is not permitted in the firewall rules, output a diagnostic error and terminate immediately with exit code `2`.
4. **Tier 3 — Application Endpoint (HTTP Health):**
   - Perform a local loopback HTTP probe against `http://127.0.0.1:80` with a connection timeout of 3 seconds.
   - Extract the HTTP response status code.
   - If the HTTP response code is anything other than `200`, output a diagnostic error including the observed status code and terminate with exit code `3`.
5. **Success State:**
   - If all three tiers pass successfully, output an informational confirmation indicating all tiers are healthy and terminate with exit code `0`.

### Verification Steps
To verify your health auditor across all failure tiers, test each condition sequentially:
1. Execute `sudo ./service_audit.sh` and verify clean success (`exit 0`) when Nginx and UFW are properly running and port 80 is open.
2. Temporarily stop Nginx (`sudo systemctl stop nginx`), execute `sudo ./service_audit.sh`, and verify it exits with code `1`. Restart Nginx.
3. Temporarily block port 80 (`sudo ufw deny 80/tcp`), execute `sudo ./service_audit.sh`, and verify it exits with code `2`. Re-allow port 80 (`sudo ufw allow 80/tcp`).
4. Temporarily replace `/var/www/html/index.html` with restrictive permissions (`sudo chmod 000 /var/www/html/index.html`) to trigger a `403 Forbidden` response, execute `sudo ./service_audit.sh`, and verify it exits with code `3`. Restore permissions (`sudo chmod 644 /var/www/html/index.html`).

### Deliverables
- Complete source code of `service_audit.sh`.
- Terminal output log demonstrating exit codes across healthy and simulated failure states.

---

## Task 3: Production Automation Architecture & Idempotency Reflection

### Context
In this module, you built automated provisioning and auditing scripts and explored the Law of Idempotency. Enterprise DevOps relies on these principles to ensure stability across fleet deployments.

### Requirements
Compose a structured engineering reflection (300–500 words) answering the following architectural questions:
1. **Imperative vs. Idempotent Automation:** Contrast fragile imperative scripting with convergent idempotent patterns. Explain why commands such as `mkdir -p` and conditional checks before user creation prevent pipeline breakage.
2. **Defensive Scripting Safety:** Explain the specific failure modes prevented by `set -e`, `set -u`, and `set -o pipefail`. Describe a concrete scenario where omitting `set -u` could lead to catastrophic data loss.
3. **The UNIX Exit Code Contract:** Explain why machine-readable return codes (such as those implemented in `service_audit.sh`) are essential for integration with enterprise monitoring platforms and CI/CD pipelines.
4. **Security Hardening in Automation:** Why must automated onboarding scripts explicitly enforce DAC permissions (`750` on projects) rather than relying on default system umask?

### Deliverables
- A Markdown document containing your structured technical reflection.

---

# Part 2: DevOps Stretch Challenge

> [!NOTE]
> This challenge is intended for students pursuing advanced DevOps, Site Reliability Engineering (SRE), or cloud infrastructure roles. It requires writing a robust, defensive automation pipeline without step-by-step guidance.

---

## The Automated Multi-Directory Backup & Integrity Pipeline (`backup_pipeline.sh`)

### Scenario
Production web servers store dynamic application assets in `/var/www/html` and critical server configurations in `/etc/nginx`. A disaster recovery standard mandates an automated, self-contained backup script that packages these assets, computes cryptographic integrity checksums, validates the archive, and enforces retention policies.

### Technical Specifications

Construct an executable script named `backup_pipeline.sh` meeting the following production standards:

#### 1. Invocation & Argument Parsing
- The script must accept two arguments:
  - Argument 1: Target directory to back up (e.g., `/var/www/html`).
  - Argument 2: Destination backup repository directory (e.g., `/var/backups/web`).
- If arguments are missing, output clear usage syntax and exit with code `1`.
- If the source directory does not exist, report a fatal error and exit with code `2`.

#### 2. Idempotent Target Initialization
- The backup repository directory must be created idempotently (`mkdir -p`) if it does not exist.
- Ensure the destination directory has secure administrative permissions (`700`).

#### 3. Archive Generation & Cryptographic Checksum
- Generate a gzip-compressed tar archive named in the format:
  `backup_<dirname>_<YYYYMMDD_HHMMSS>.tar.gz`
- Immediately compute the SHA-256 checksum of the generated archive file and save it as an adjacent file with the extension `.sha256` (e.g., `backup_html_20260918_143000.tar.gz.sha256`).

#### 4. Automated Verification Probe
- Before declaring the backup operation successful, execute an automated integrity validation using `sha256sum --check` against the generated `.sha256` file.
- If the integrity check fails, delete the corrupted archive, log a critical alert, and exit with code `3`.

#### 5. Automated Retention Enforcement
- The script must implement an automated cleanup policy that inspects the destination directory and retains only the 5 most recent backup archives (and their associated `.sha256` files), deleting older archives.
- If fewer than 5 backups exist, no files are deleted.

#### 6. Structured ISO-8601 Logging
- All actions, informational notices, file sizes, and verification results must be printed to standard output prefixed with standard timestamps (`[YYYY-MM-DD HH:MM:SS] [LEVEL]`).

### Recommended Verification Workflow
To verify your backup pipeline across operational stages and retention enforcement:
1. Create a mock application directory (such as `/tmp/test_app`) containing sample files.
2. Execute `sudo ./backup_pipeline.sh /tmp/test_app /var/backups/web` and verify archive creation and SHA-256 verification.
3. Verify cryptographic integrity manually using `cd /var/backups/web && sha256sum --check *.sha256`.
4. Execute the script multiple times (at least 6 times) with a brief pause between runs to confirm that the retention policy prunes the oldest archives, keeping exactly the 5 latest archives.

### Deliverables
- Complete source code of `backup_pipeline.sh`.
- Terminal execution log showing:
  1. Successful backup generation and SHA-256 verification.
  2. Multiple runs demonstrating the retention policy pruning older files while keeping the latest 5.

---

## Environment Cleanup Script

When you have completed testing and capturing output logs, execute the following commands to remove student accounts, test directories, and temporary files:

The following commands clean up test artifacts, restore default firewall settings, and remove created user accounts:

```bash
# 1. Remove test user created during onboarding
sudo userdel -r dev_alex 2>/dev/null || true

# 2. Remove test backup repositories and mock application directories
sudo rm -rf /var/backups/web /tmp/test_app /tmp/test_backup 2>/dev/null || true

# 3. Ensure Nginx is running and Port 80 is allowed
sudo systemctl restart nginx
sudo ufw allow 80/tcp

# 4. Remove scratch scripts created during testing
rm -f onboard.sh service_audit.sh backup_pipeline.sh
```
