# Module 6: Bash Automation & Capstone — Command Cheat Sheet

A comprehensive quick-reference guide for Bash scripting syntax, defensive directives, conditional operators, control structures, and enterprise idempotent provisioning patterns.

---

## 1. Script Lifecycle & Defensive Execution Directives

In production DevOps environments, automation scripts run unattended inside CI/CD runners, cron schedulers, or Kubernetes container initialization phases. Defensive directives ensure that silent errors never cascade into silent disasters.

| Directive / Command | Category | Description | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- | :--- |
| `#!/bin/bash` | Shebang | Kernel execution instruction defining the interpreter. | Required at line 1 of every script so kernel `execve()` launches Bash rather than falling back to default shell (`/bin/sh` or Dash). | `#!/bin/bash` |
| `chmod +x <file.sh>` | Security | Grants execute bit (`x`) on the script file. | Enables direct execution (`./deploy.sh`) by deployment pipelines and service managers without wrapping in `bash <file>`. | `chmod +x /opt/scripts/deploy.sh` |
| `./<file.sh>` | Execution | Executes script as a child process in a subshell. | Standard execution pattern ensuring script variables and working directory shifts remain isolated from parent environment. | `./provision.sh` |
| `bash -x <file.sh>` | Forensics | Traces execution by printing every expanded command. | Used during incident triage or pipeline debugging to inspect exact runtime variable expansions and branch choices. | `bash -x ./onboard.sh dev_alex` |
| `source <file.sh>` or `. <file.sh>` | Execution | Executes script inside current shell environment. | Used in environment bootstrapping and secrets management to export variables directly into the active terminal session. | `source /etc/environment.d/production.env` |
| `set -e` | Defensive | Halts execution immediately if any command returns non-zero. | Crucial in deployment pipelines to prevent subsequent commands from executing after a database migration or package download fails. | `set -e` |
| `set -u` | Defensive | Treats uninitialized variables as fatal runtime errors. | Prevents catastrophic operations such as `rm -rf /${DEPLOY_DIR}` when `DEPLOY_DIR` is accidentally undefined. | `set -u` |
| `set -o pipefail` | Defensive | Preserves non-zero exit status across piped sequences. | Prevents pipeline masking where a failing primary command (e.g., `mysqldump \| gzip`) produces exit code 0 because `gzip` succeeded. | `set -o pipefail` |
| `set -euo pipefail` | Defensive | Combines all three defensive directives into a single declaration. | Standard enterprise boilerplate at the beginning of every production script to enforce zero tolerance for unchecked failures. | `set -euo pipefail` |

---

## 2. Special Variables & Parameter Expansion

Dynamic shell scripts rely on special built-in tokens to validate arguments, inspect process states, and safely expand configuration defaults.

| Token | Scope | Description | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- | :--- |
| `$0` | Special | Script path or invocation name. | Used in error reporting and usage guidance messages to dynamically reference the script name without hardcoding. | `echo "Usage: sudo $0 <cluster_name>"` |
| `$1, $2 ... $9` | Special | Positional input arguments passed to script or function. | Used to accept dynamic infrastructure parameters such as usernames, hostnames, IP addresses, or deployment environments. | `ENVIRONMENT="${1:-staging}"` |
| `$#` | Special | Integer count of command-line arguments supplied. | Evaluated at script entry to validate that callers supplied the required number of parameters before proceeding. | `if [ "$#" -ne 2 ]; then exit 1; fi` |
| `$@` | Special | Array of all positional arguments as discrete quoted tokens. | Used in iteration loops to process multiple targets (e.g., list of servers, packages, or user accounts) safely preserving whitespace. | `for host in "$@"; do ssh "$host"; done` |
| `$$` | Special | Process ID (PID) of the active script process. | Written to lockfiles (`/var/run/script.pid`) to enforce single-instance concurrency and facilitate watchdog tracking. | `echo "$$" > /var/run/app_worker.pid` |
| `$?` | Special | Exit status byte (0–255) of the last executed command. | Interrogated immediately following critical commands to determine branch logic and trigger automated rollback handlers. | `if [ "$?" -ne 0 ]; then rollback; fi` |
| `$EUID` | Environment | Effective User ID of the process owner (0 for root). | Verified as the first operation in administrative scripts to fail fast if non-privileged users attempt system modifications. | `[ "${EUID:-$(id -u)}" -eq 0 ] \|\| exit 1` |
| `$(command)` | Subshell | Captures stdout of child command into a variable. | Used to query live runtime state such as active default gateway, public IP, timestamp, or kernel version. | `GATEWAY=$(ip route show default \| awk '{print $3}')` |
| `${VAR:-default}` | Expansion | Uses variable if defined; falls back to default if unset. | Standard pattern for infrastructure configuration defaults (e.g., fallback ports, default timeout values, default environments). | `PORT="${PORT:-8080}"` |

---

## 3. Test Evaluation Operators `[ ]`

System administration scripts evaluate conditional tests to verify file states, numerical thresholds, and configuration strings before taking action.

The test evaluation operators are organized below across filesystem inspections, numeric evaluations, and string comparisons:

### Filesystem Inspections
| Operator | Condition Evaluated | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `-f <path>` | True if path exists and is a regular file. | Pre-validates presence of configuration templates, SSL certificates, or secrets before launching services. | `[ -f "/etc/nginx/sites-available/default" ]` |
| `-d <path>` | True if path exists and is a directory. | Validates target mount points, document roots, or log directories before writing data. | `[ -d "/var/www/html" ]` |
| `-x <path>` | True if file exists and has execute bit set. | Verifies that prerequisite dependencies (e.g., `docker`, `curl`, `jq`) are present and executable before invocation. | `[ -x "/usr/bin/curl" ]` |
| `-s <path>` | True if file exists and size > 0 bytes. | Verifies that generated artifacts, downloaded files, or backup archives are not empty 0-byte corruptions. | `[ -s "/backups/db_backup.sql" ]` |
| `-r <path>` | True if file exists and is readable by caller. | Verifies read permissions before attempting to parse sensitive files like private keys or password databases. | `[ -r "/etc/ssl/certs/server.crt" ]` |

### Numeric Comparisons
| Operator | Condition Evaluated | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `-eq` | True if numeric values are equal. | Used to verify expected HTTP response codes (e.g., 200 OK) or zero error counts. | `[ "$HTTP_STATUS" -eq 200 ]` |
| `-ne` | True if numeric values are not equal. | Used for non-root privilege validation or non-zero exit code detection. | `[ "$EUID" -ne 0 ]` |
| `-gt` | True if first value is greater than second. | Used in monitoring scripts to alert when CPU, disk utilization, or failure counts exceed thresholds. | `[ "$DISK_USAGE_PERCENT" -gt 90 ]` |
| `-lt` | True if first value is less than second. | Used in retry loops to ensure attempts remain below maximum threshold. | `[ "$RETRY_COUNT" -lt 5 ]` |
| `-ge` | True if value is greater than or equal to. | Evaluates quorum counts in distributed clusters or worker thread counts. | `[ "$ACTIVE_NODES" -ge 3 ]` |
| `-le` | True if value is less than or equal to. | Enforces concurrency limits or resource bounds. | `[ "$CONNS" -le 1000 ]` |

### String Comparisons
| Operator | Condition Evaluated | Production & Enterprise Use Case (When & Why) | Real-World Example |
| :--- | :--- | :--- | :--- |
| `==` | True if strings are identical. | Evaluates deployment environments (`production` vs `staging`) or role definitions. | `[ "$DEPLOY_ENV" == "production" ]` |
| `!=` | True if strings are not identical. | Detects unexpected state changes or unapproved host configurations. | `[ "$CURRENT_BRANCH" != "main" ]` |
| `-z` | True if string is empty (zero length). | Verifies whether mandatory environment variables or CLI arguments were omitted. | `[ -z "$DATABASE_URL" ]` |
| `-n` | True if string is non-empty (has content). | Confirms presence of required tokens, passwords, or configuration overrides. | `[ -n "$API_TOKEN" ]` |

---

## 4. Control Structures & Modularity Patterns

Structured control flow enables robust state checking, iterative processing, and modular code organization across large infrastructure codebases.

The following reference implementations illustrate standard conditional branching, multi-option pattern matching, retry loops, and reusable logging functions:

The conditional branching structure evaluates multiple logic paths using `if`, `elif`, and `else` statements:
```bash
if [ "$EUID" -ne 0 ]; then
    echo "Error: Root privileges required." >&2
    exit 1
elif [ ! -f "/etc/nginx/nginx.conf" ]; then
    echo "Error: Nginx configuration missing." >&2
    exit 2
else
    echo "Pre-flight validation passed."
fi
```

The pattern matching `case` structure provides clear execution routing across enumerated administrative actions:
```bash
case "$ACTION" in
    start)
        systemctl start nginx
        ;;
    stop)
        systemctl stop nginx
        ;;
    reload|restart)
        nginx -t && systemctl reload nginx
        ;;
    *)
        echo "Usage: $0 {start|stop|reload}" >&2
        exit 1
        ;;
esac
```

The retry while-loop construct handles transient network latency with automated backoff:
```bash
MAX_RETRIES=3
COUNT=0
while [ "$COUNT" -lt "$MAX_RETRIES" ]; do
    if curl -s -f http://127.0.0.1:80 &>/dev/null; then
        echo "Web service verified healthy."
        break
    fi
    COUNT=$((COUNT + 1))
    echo "Waiting for service to bind... attempt $COUNT of $MAX_RETRIES"
    sleep 2
done
```

The modular logging function standardizes timestamped diagnostic output across automation runs:
```bash
log_event() {
    local LEVEL="$1"
    local MESSAGE="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$LEVEL] $MESSAGE"
}
```

---

## 5. Enterprise Idempotent Provisioning Patterns

In production Infrastructure as Code (IaC), scripts must converge the system toward the Desired State regardless of whether the script is run on a clean box or an existing production server.

| Task | Fragile Imperative Command (Crash-Prone) | Resilient Idempotent Pattern (Convergent) | Enterprise Benefit / Rationale |
| :--- | :--- | :--- | :--- |
| **Directory Creation** | `mkdir /data/web` | `mkdir -p /data/web` | `-p` creates parent paths if missing and exits 0 without error if path already exists. |
| **User Administration** | `useradd developer` | `id "developer" &>/dev/null \|\| useradd -m -s /bin/bash "developer"` | Inspects system user database first; provisions only if missing, avoiding Exit Code 9. |
| **Package Installation** | `apt install -y nginx` | `dpkg -s nginx &>/dev/null \|\| (apt-get update -qq && apt-get install -y -qq nginx)` | Checks local package status before updating repos and triggering redundant network transfers. |
| **Firewall Ingress** | `ufw allow 80/tcp` | `ufw status \| grep -q '80/tcp' \|\| ufw allow 80/tcp` | Prevents redundant duplicate rule insertions in `/etc/ufw/user.rules`. |
| **Firewall Activation** | `ufw enable` | `ufw --force enable` | `--force` suppresses interactive TTY confirmation prompts that freeze CI/CD pipelines. |
| **Daemon Reload** | `systemctl restart nginx` | `nginx -t && systemctl reload nginx` | Validates syntax first; performs zero-downtime graceful reload instead of hard restart. |
| **Ownership Enforcement** | `chown user file` | `chown -R user:group /target/dir` | Recursively reconciles desired ownership without crashing if permissions already match. |
| **SSH Key Generation** | `ssh-keygen` (interactive) | `[ -f "$KEY" ] \|\| ssh-keygen -t ed25519 -N "" -C "tag" -f "$KEY" -q` | Pre-checks keyfile existence; generates non-interactively with empty passphrase if absent. |
| **Public Key Authorization** | `cat key.pub >> authorized_keys` | `grep -q -F -f "$PUB" "$AUTH" 2>/dev/null \|\| cat "$PUB" >> "$AUTH"` | Checks for existing key signature to prevent unbounded duplicate entries in `authorized_keys`. |
| **Lockfile Concurrency** | `echo $$ > /tmp/lock` | `[ -f "$LOCK" ] && kill -0 "$(cat "$LOCK")" 2>/dev/null \|\| echo $$ > "$LOCK"` | Verifies if recorded PID is alive, handling stale locks while preventing overlapping cron runs. |

