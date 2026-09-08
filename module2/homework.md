# Module 2 Homework: Security & Access Control

Five practical tasks covering user and group management, directory permissions (770), file modes (symbolic and octal), group sudo delegation, and SSH keypair generation.

---

## Task 1: Role-Based Account Onboarding

Onboard a new data analyst and configure their primary and supplementary group memberships.

### Requirements:
1. Create a new user account named `analyst_bob` configured with a dedicated home directory (`/home/analyst_bob`) and interactive shell `/bin/bash`.
2. Set a temporary password for `analyst_bob`.
3. Create a new supplementary group named `analytics`.
4. Add `analyst_bob` to the `analytics` group without removing the user from any other group memberships.
5. Verify the account details in `/etc/passwd` and confirm group memberships using `id`.

<details>
<summary>Hints (Click to expand)</summary>

- Account creation with home directory and shell: `sudo useradd -m -s /bin/bash <username>`
- Password setting: `sudo passwd <username>`
- Group creation: `sudo groupadd <groupname>`
- Group membership append: `sudo usermod -aG <groupname> <username>` (ensure `-a` is included to prevent group eviction)
- Identity verification: `id <username>` and `grep <username> /etc/passwd`

</details>

**Deliverables:**
- Console output of `id analyst_bob`.
- The matching entry for `analyst_bob` from `/etc/passwd`.

---

## Task 2: Isolated Team Workspace Collaboration

Establish a restricted directory for sensitive financial reporting where only team members have access.

### Requirements:
1. Create a directory named `/opt/analytics_data` using administrative privileges.
2. Change the group ownership of `/opt/analytics_data` to the `analytics` group.
3. Configure the directory permissions so that:
   - The Owner (`root`) and members of the `analytics` Group have full access (Read, Write, and Execute).
   - Others (all other users on the host) have **zero permissions** (no read, no write, and cannot enter the directory).
4. Switch to `analyst_bob` and create a report file named `/opt/analytics_data/q1_metrics.csv`.
5. Switch back to your primary user and verify that unprivileged access to `/opt/analytics_data` is blocked.

<details>
<summary>Hints (Click to expand)</summary>

- Group ownership: `sudo chgrp <group> <directory>`
- Restrictive mode: Mode `770` (`rwxrwx---`) permits Owner and Group while locking out Others
- Testing as user: `sudo su - <username>`
- Verification: Attempt `ls /opt/analytics_data` from an account outside the `analytics` group

</details>

**Deliverables:**
- Long listing output of `ls -ld /opt/analytics_data`.
- Long listing output of `ls -l /opt/analytics_data/q1_metrics.csv`.
- Terminal output demonstrating that a non-member user receives `Permission denied` when attempting to list `/opt/analytics_data`.

---

## Task 3: Granular File Permissions & Secret Tokens

Apply symbolic and octal permission controls to data pipelines and API credentials.

### Requirements:
1. In your home directory, create a shell script named `process_data.sh` containing the line:
   ```bash
   echo "Processing quarterly financial dataset..."
   ```
2. Using **symbolic notation**, grant execute permission to both the Owner (`u`) and Group (`g`) simultaneously, while ensuring Others (`o`) have no execute permissions.
3. In your home directory, create a credential file named `db_credentials.env` containing mock credentials:
   ```text
   DB_USER=analytics_prod
   DB_TOKEN=9f8e7d6c5b4a3210
   ```
4. Using **octal notation**, lock down `db_credentials.env` so that **only the owner** can read and write to it (`rw-------`), while Group and Others have zero permissions.
5. Transfer ownership of both `process_data.sh` and `db_credentials.env` to user `analyst_bob` and group `analytics`.

<details>
<summary>Hints (Click to expand)</summary>

- Multi-target symbolic chmod: `chmod ug+x,o-x <filename>`
- Restricted octal mode: Mode `600` (`rw-------`)
- Ownership modification: `sudo chown <user>:<group> <filename>`

</details>

**Deliverables:**
- Long listing output of `ls -l process_data.sh db_credentials.env` showing owners, groups, and permission bits.

---

## Task 4: Least-Privilege Sudo Delegation for Service Management

Configure granular administrative permissions so the analytics team can restart and inspect the system cron scheduler without granting full superuser power.

### Requirements:
1. Confirm that `analyst_bob` currently cannot execute administrative commands with `sudo`.
2. Create a drop-in sudoers file named `/etc/sudoers.d/analytics_team` using `visudo`.
3. Configure the rule targeting the group `%analytics` so that any member can execute `/usr/bin/systemctl restart cron` and `/usr/bin/systemctl status cron` as root without entering a password (`NOPASSWD:`), while all other privileged commands remain forbidden.
4. Validate the syntax of the drop-in file using the `visudo` check mode (`-cf`).
5. Switch to `analyst_bob` and audit active privileges with `sudo -l`.
6. Test executing `sudo systemctl status cron` (must succeed without password) and `sudo cat /etc/shadow` (must be rejected).

<details>
<summary>Hints (Click to expand)</summary>

- Group sudo syntax: `%<groupname> ALL=(ALL) NOPASSWD: /path/to/cmd1, /path/to/cmd2`
- Safe automated creation: `echo "..." | sudo EDITOR='tee' visudo -f /etc/sudoers.d/<filename>`
- Pre-flight syntax validation: `sudo visudo -cf /etc/sudoers.d/<filename>`
- Privilege inspection: `sudo -l`

</details>

**Deliverables:**
- Content of `/etc/sudoers.d/analytics_team`.
- Terminal output of `sudo visudo -cf /etc/sudoers.d/analytics_team`.
- Terminal output of `sudo -l` executed as `analyst_bob`.
- Terminal output showing the rejection when attempting `sudo cat /etc/shadow`.

---

## Task 5: Cryptographic SSH Key Generation & Remote Audit

Generate an Ed25519 authentication key pair for the new analyst and perform a configuration security audit.

### Requirements:
1. Generate an Ed25519 SSH keypair for `analyst_bob` non-interactively:
   - Key type: `ed25519`
   - Key comment: `analyst_bob@finanalytics.internal`
   - Output path: `/home/analyst_bob/.ssh/id_ed25519`
   - Passphrase: empty (`-N ""`) for automated testing
2. Ensure the `/home/analyst_bob/.ssh` directory has permissions `700`, the private key has `600`, and the public key has `644`, all owned by `analyst_bob:analytics`.
3. Inspect `/etc/ssh/sshd_config` and verify whether `PermitRootLogin` is configured securely.
4. Execute the pre-flight syntax check `sudo sshd -t` to confirm the SSH daemon configuration is error-free.
5. Display active logged-in sessions using `who` and `w`.

<details>
<summary>Hints (Click to expand)</summary>

- Key generation: `sudo ssh-keygen -t ed25519 -C "<comment>" -f <path> -N ""`
- Strict DAC permissions: `sudo chmod 700 <dir>`, `sudo chmod 600 <private_key>`, `sudo chmod 644 <public_key>`
- Ownership: `sudo chown -R <user>:<group> <path>`
- Active config filter: `grep -i "PermitRootLogin" /etc/ssh/sshd_config`
- SSH syntax verification: `sudo sshd -t`

</details>

**Deliverables:**
- Long listing output of `ls -la /home/analyst_bob/.ssh`.
- Output of `grep -i "PermitRootLogin" /etc/ssh/sshd_config`.
- Exit code output of `sudo sshd -t; echo $?`.
