# Module 1: Essential Linux Commands Cheat Sheet

A quick-reference table for commands introduced in **Module 1: Fundamentals, Tooling & Processes**.

---

## 1. Virtual Machine Lifecycle (WSL 2 & Multipass)

| Command | Platform | Description | Example / Use Case |
| :--- | :--- | :--- | :--- |
| `wsl --install -d Ubuntu --name [vm]` | Windows PS | Creates and installs a named Ubuntu instance | `wsl --install -d Ubuntu --name devops-vm` |
| `wsl -l -v` | Windows PS | Lists all installed WSL instances and running state | `wsl -l -v` |
| `wsl -d [vm]` | Windows PS | Starts instance and attaches interactive shell | `wsl -d devops-vm` |
| `exit` / `Ctrl+D` | Linux Shell | Detaches from Linux shell back to host terminal | `exit` |
| `wsl -t [vm]` | Windows PS | Stops / terminates a specific running instance | `wsl -t devops-vm` |
| `wsl --shutdown` | Windows PS | Forcibly terminates all running WSL instances | `wsl --shutdown` |
| `multipass launch --name [vm]` | macOS | Creates and launches an Ubuntu cloud instance | `multipass launch --name devops-vm --cpus 2 --memory 2G --disk 10G` |
| `multipass list` | macOS | Lists all Multipass VMs, states, and IPv4 addresses | `multipass list` |
| `multipass info [vm]` | macOS | Displays detailed VM resource specs and stats | `multipass info devops-vm` |
| `multipass start [vm]` | macOS | Boots a stopped VM in the background | `multipass start devops-vm` |
| `multipass shell [vm]` | macOS | Attaches an interactive shell into the VM | `multipass shell devops-vm` |
| `multipass stop [vm]` | macOS | Gracefully halts a running VM | `multipass stop devops-vm` |
| `multipass delete [vm] && multipass purge` | macOS | Permanently deletes VM and reclaims disk space | `multipass delete devops-vm && multipass purge` |

---

## 2. System Information & Environment

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `whoami` | Displays current logged-in username | `whoami` |
| `uname -a` | Prints all system and kernel information | `uname -a` |
| `hostname` | Displays the system's network hostname | `hostname` |
| `hostname -I` | Shows assigned internal IP addresses | `hostname -I` |
| `ps -p 1 -o comm=` | Checks the process name of PID 1 | `ps -p 1 -o comm=` |

---

## 3. Navigation & Directory Hierarchy

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `pwd` | **P**rint **W**orking **D**irectory | `pwd` |
| `cd [dir]` | **C**hange **D**irectory | `cd /etc`, `cd ..`, `cd ~` |
| `cd -` | Returns to the previous working directory | `cd -` |
| `ls` | Lists visible files and directories | `ls` |
| `ls -l` | Detailed long listing (permissions, owner, size) | `ls -l /var/log` |
| `ls -la` | Lists all files including hidden dotfiles (`.`) | `ls -la ~` |
| `ls -lh` | Shows human-readable file sizes (K, M, G) | `ls -lh /var/log` |
| `ls -lt` | Sorts files by modification time (newest first) | `ls -lt /var/log` |
| `ls -lart` | Reverse time sort; newest modified files at bottom | `ls -lart /var/log` |
| `ls -ld [dir]` | Shows directory metadata rather than its contents | `ls -ld /var/log` |
| `ls -R [dir]` | Recursively lists all subdirectories and files | `ls -R configs/` |
| `ls /mnt/c` | Views host Windows C: drive (WSL 2) | `ls /mnt/c/Users` |

---

## 4. File Operations & Text Editing

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `mkdir -p [path]` | Creates directory and parents idempotently | `mkdir -p configs/backup` |
| `touch [file]` | Creates an empty file or updates timestamp | `touch notes.txt` |
| `cp [src] [dst]` | Copies a single file | `cp app.conf app.conf.bak` |
| `cp -r [src] [dst]` | Recursively copies an entire directory | `cp -r configs/ backup/` |
| `cp -a [src] [dst]` | Archive copy: preserves permissions, owner, timestamps | `cp -a configs/ backup/` |
| `mv [src] [dst]` | Moves or renames a file or directory | `mv old.txt new.txt` |
| `rm [file]` | Removes a file permanently (no recycle bin!) | `rm obsolete.txt` |
| `rm -r [dir]` | Recursively removes a directory | `rm -r temp_folder` |
| `rm -rf [dir]` | Forcefully and recursively removes a directory | `rm -rf build_output/` |
| `echo "[text]"` | Prints text string to the terminal screen | `echo "Hello Linux"` |
| `echo "[text]" > [file]` | Writes/overwrites text into a file | `echo "PORT=8080" > app.conf` |
| `echo "[text]" >> [file]` | Appends text to end of file without overwriting | `echo "DEBUG=false" >> app.conf` |
| `nano [file]` | Opens terminal text editor (`Ctrl+O` save, `Ctrl+X` exit) | `nano app.conf` |

---

## 5. Text Inspection & Search

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `cat [file]` | Prints full contents of a file to screen | `cat /etc/os-release` |
| `less [file]` | Interactive pager to scroll through large text files | `less /var/log/dpkg.log` |
| `head -n [N] [file]` | Displays the first $N$ lines of a file | `head -n 5 /etc/passwd` |
| `tail -n [N] [file]` | Displays the last $N$ lines of a file | `tail -n 10 /var/log/dpkg.log` |
| `tail -f [file]` | Streams and follows new file entries in real time | `tail -f /var/log/dpkg.log` |
| `grep [pattern] [file]` | Searches for lines matching a pattern | `grep "PORT" app.conf` |
| `grep -i [pattern] [file]` | Performs case-insensitive pattern search | `grep -i "error" app.log` |
| `grep -rn [pattern] [dir]` | Recursively searches directory with line numbers | `grep -rn "DB_PASS" configs/` |
| `grep -v [pattern] [file]` | Inverts match: returns lines NOT matching pattern | `grep -v "^#" app.conf` |
| `command \| grep [pat]` | Pipes command output through grep filter | `ps aux \| grep bash` |

---

## 6. Package Management (`apt`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo apt update` | Refreshes the local catalog index of available packages | `sudo apt update` |
| `apt search [keyword]` | Searches package repository for software | `apt search htop` |
| `apt show [pkg]` | Displays package version, dependencies, and info | `apt show htop` |
| `sudo apt install -y [pkg]` | Installs a software package non-interactively | `sudo apt install -y htop` |
| `apt list --installed` | Lists installed packages (filterable with grep) | `apt list --installed \| grep htop` |
| `sudo apt remove [pkg]` | Uninstalls package but preserves configuration files | `sudo apt remove -y sl` |
| `sudo apt purge [pkg]` | Uninstalls package AND deletes all configuration files | `sudo apt purge -y sl` |
| `sudo apt autoremove -y` | Removes orphaned dependencies no longer required | `sudo apt autoremove -y` |
| `sudo apt clean` | Clears local package archive download cache | `sudo apt clean` |
| `sudo apt upgrade -y` | Upgrades all installed packages to latest versions | `sudo apt upgrade -y` |

---

## 7. Process Monitoring & Control

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `ps aux` | Comprehensive snapshot of all active processes | `ps aux` |
| `top` | Interactive real-time task manager (`Shift+M` sort RAM, `Shift+P` sort CPU) | `top` |
| `htop` | Modern, visual process viewer (`F5` tree view, `q` quit) | `htop` |
| `[command] &` | Starts a command in the background | `sleep 100 &` |
| `jobs` | Lists active background jobs in current shell session | `jobs` |
| `bg %[job_id]` | Resumes a suspended job in the background | `bg %1` |
| `fg %[job_id]` | Brings a background job to the foreground | `fg %1` |
| `kill [PID]` | Sends SIGTERM (15) to request polite shutdown | `kill 1234` |
| `kill -9 [PID]` | Sends SIGKILL (9) to force-stop immediately | `kill -9 1234` |
| `pkill [name]` | Terminates processes matching a name | `pkill sleep` |
