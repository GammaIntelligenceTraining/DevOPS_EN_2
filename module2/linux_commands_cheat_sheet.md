# Module 2: Essential Linux Commands Cheat Sheet

A quick-reference table for commands introduced in **Module 2: Security & Access Control**.

---

## 1. Identity & Account Inspection

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `whoami` | Displays current logged-in username | `whoami` |
| `id` | Shows current UID, primary GID, and supplementary groups | `id` |
| `id [user]` | Inspects UID and group memberships of a specific user | `id dev_student` |
| `groups` | Lists group memberships of current user | `groups` |
| `groups [user]` | Lists group memberships of a specific user | `groups dev_student` |

---

## 2. User & Group Management

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo useradd -m -s [shell] [user]` | Creates new user with home directory and shell | `sudo useradd -m -s /bin/bash dev_student` |
| `sudo passwd [user]` | Sets or changes a user account password | `sudo passwd dev_student` |
| `sudo passwd -l [user]` | Locks account password (disables password login) | `sudo passwd -l dev_student` |
| `sudo passwd -u [user]` | Unlocks account password | `sudo passwd -u dev_student` |
| `sudo passwd -S [user]` | Shows password status (P=usable, L=locked, NP=none) | `sudo passwd -S dev_student` |
| `sudo usermod -aG [group] [user]` | Appends user to a supplementary group (never omit `-a`) | `sudo usermod -aG engineering dev_student` |
| `sudo userdel -r [user]` | Deletes user account and removes home directory | `sudo userdel -r dev_student` |
| `sudo groupadd [group]` | Creates a new group | `sudo groupadd engineering` |
| `sudo gpasswd -d [user] [group]` | Removes a user from a specific group | `sudo gpasswd -d dev_student engineering` |
| `sudo groupdel [group]` | Deletes a group from the system | `sudo groupdel engineering` |

---

## 3. Switching User Environments

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `su [user]` | Switches to user preserving current environment | `su dev_student` |
| `su - [user]` | Switches to user with fresh, clean login environment | `su - dev_student` |
| `sudo su - [user]` | Switches to any user using administrative privilege | `sudo su - dev_student` |
| `exit` | Closes current shell session and returns to previous | `exit` |

---

## 4. File & Directory Permissions (`chmod`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `chmod u+x [file]` | Grants execute permission to the file owner | `chmod u+x deploy.sh` |
| `chmod g+w [file]` | Grants write permission to group members | `chmod g+w shared.txt` |
| `chmod o-rwx [file]` | Revokes all permissions from others | `chmod o-rwx secrets.txt` |
| `chmod a+r [file]` | Grants read permission to everyone (user, group, others) | `chmod a+r public.conf` |
| `chmod 755 [file/dir]` | Sets rwxr-xr-x (standard for executables and folders) | `chmod 755 deploy.sh` |
| `chmod 644 [file]` | Sets rw-r--r-- (standard for readable configs and files) | `chmod 644 app.conf` |
| `chmod 600 [file]` | Sets rw------- (owner read/write only, private key standard) | `chmod 600 id_ed25519` |
| `chmod 700 [dir]` | Sets rwx------ (owner full access only, `.ssh` standard) | `chmod 700 ~/.ssh` |
| `chmod 775 [dir]` | Sets rwxrwxr-x (team shared folder standard) | `chmod 775 /srv/engineering` |
| `chmod -R [mode] [dir]` | Recursively applies permissions across directory tree | `chmod -R 750 /srv/app` |
| `find [dir] -perm 777` | Finds insecure world-writable files/directories | `find /srv -perm 777` |

---

## 5. Ownership Management (`chown`, `chgrp`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo chown [user] [file]` | Changes file owner | `sudo chown dev_student app.conf` |
| `sudo chown [user]:[group] [file]` | Changes owner and group simultaneously | `sudo chown dev_student:engineering app.conf` |
| `sudo chgrp [group] [file]` | Changes group ownership of a file | `sudo chgrp engineering app.conf` |
| `sudo chown -R [user]:[group] [dir]`| Recursively changes ownership for an entire directory tree | `sudo chown -R dev_student:engineering /srv/project` |

---

## 6. Privilege Escalation & Sudo Delegation

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo [command]` | Executes a single command with root privileges | `sudo cat /etc/shadow` |
| `sudo -l` | Lists allowed (and forbidden) sudo commands for current user | `sudo -l` |
| `sudo visudo` | Safely edits `/etc/sudoers` with locking and syntax checks | `sudo visudo` |
| `sudo visudo -cf [file]` | Validates syntax of a sudoers drop-in configuration file | `sudo visudo -cf /etc/sudoers.d/dev_student` |

---

## 7. SSH Management & Security Auditing

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `ssh-keygen -t ed25519` | Generates modern, secure Ed25519 SSH key pair | `ssh-keygen -t ed25519` |
| `ssh-copy-id [user]@[host]` | Installs public key on remote host (`~/.ssh/authorized_keys`) | `ssh-copy-id dev_student@192.168.1.50` |
| `ssh -i [key] [user]@[host]` | Connects to remote host using specific private key | `ssh -i ~/.ssh/id_ed25519 dev_student@192.168.1.50` |
| `sudo sshd -t` | Tests SSH daemon configuration syntax without restarting | `sudo sshd -t` |
| `sudo systemctl status ssh` | Checks operational runtime status of SSH daemon | `sudo systemctl status ssh` |
| `sudo systemctl restart ssh` | Restarts SSH daemon to apply configuration updates | `sudo systemctl restart ssh` |
| `who` | Displays currently logged-in users and their terminal devices | `who` |
| `w` | Displays active users, idle times, and current running processes | `w` |
| `last -n [count]` | Shows history of recent logins, logouts, and system reboots | `last -n 10` |
| `lastlog` | Displays the most recent login timestamp for all accounts | `lastlog` |
