# Module 3: Essential Linux Commands Cheat Sheet

A quick-reference table for commands introduced in **Module 3: Networking Essentials & Services**.

---

## 1. Network Interfaces & IP Configuration

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `ip link show` | Displays all network interfaces and their physical link states | `ip link show` |
| `ip a` | Displays all assigned IPv4 and IPv6 addresses | `ip a` |
| `ip -br a` | Brief one-line overview of interfaces and assigned IPs | `ip -br a` |
| `ip r` | Displays kernel IP routing table and default gateway | `ip r` |
| `hostname -I` | Prints all internal assigned IP addresses | `hostname -I` |
| `curl -s ifconfig.me` | Queries external service to discover public IP | `curl -s ifconfig.me` |

---

## 2. DNS Investigation & Resolution

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `dig [domain]` | Standard DNS lookup (inspect `ANSWER SECTION`) | `dig google.com` |
| `dig +short [domain]` | Returns only resolved IP addresses | `dig +short google.com` |
| `dig [domain] [type]` | Queries specific DNS record (A, AAAA, CNAME, MX, TXT) | `dig google.com MX` |
| `dig @[resolver] [domain]` | Queries a specific DNS server directly | `dig @1.1.1.1 google.com` |
| `dig [domain] +trace` | Traces full lookup path from Root to Authoritative nameservers | `dig google.com +trace` |
| `dig -x [ip]` | Performs reverse DNS lookup (PTR record) | `dig -x 8.8.8.8` |
| `host [domain]` | Quick summary of domain IP and mail records | `host github.com` |
| `nslookup [domain]` | Conversational DNS lookup utility | `nslookup google.com` |

---

## 3. Systemd Service Lifecycle (`systemctl`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `systemctl status [service]` | Displays service runtime state, PID, cgroups, and recent logs | `systemctl status nginx` |
| `sudo systemctl start [service]` | Starts service immediately in RAM | `sudo systemctl start nginx` |
| `sudo systemctl stop [service]` | Stops running service | `sudo systemctl stop nginx` |
| `sudo systemctl restart [service]`| Performs full stop and restart of service | `sudo systemctl restart nginx` |
| `sudo systemctl reload [service]` | Reloads service configuration without dropping connections | `sudo systemctl reload nginx` |
| `sudo systemctl enable [service]` | Enables service to start automatically on system boot | `sudo systemctl enable nginx` |
| `sudo systemctl disable [service]`| Disables automatic startup on boot | `sudo systemctl disable nginx` |
| `sudo systemctl daemon-reload` | Reloads systemd manager configuration after unit changes | `sudo systemctl daemon-reload` |
| `systemctl is-active [service]` | Returns active or inactive runtime status | `systemctl is-active nginx` |
| `systemctl is-enabled [service]`| Returns enabled or disabled boot configuration | `systemctl is-enabled nginx` |
| `systemctl list-units --type=service --state=running` | Lists all currently active background services | `systemctl list-units --type=service` |

---

## 4. Sockets & Port Monitoring (`ss`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo ss -tunlp` | Displays all listening TCP/UDP sockets with process names/PIDs | `sudo ss -tunlp` |
| `sudo ss -tunlp \| grep :[port]`| Filters listening sockets for a specific port number | `sudo ss -tunlp \| grep :80` |
| `ss -ta` | Displays all TCP sockets (listening and established) | `ss -ta` |
| `ss -s` | Summarizes socket statistics and active counts | `ss -s` |

---

## 5. Web Diagnostics & Connectivity Tools

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `curl [url]` | Fetches web resource and prints response body | `curl http://localhost` |
| `curl -I [url]` | Fetches HTTP headers only (status code, server software) | `curl -I http://localhost` |
| `curl -v [url]` | Verbose mode: displays DNS lookup, TCP handshake, TLS data | `curl -v https://google.com` |
| `curl -s [url]` | Silent mode: suppresses progress meter (ideal for scripts) | `curl -s ifconfig.me` |
| `curl -o [file] [url]` | Downloads remote content to specified filename | `curl -o index.html http://localhost` |
| `curl -O [url]` | Downloads remote file preserving remote filename | `curl -O http://example.com/logo.png` |
| `curl -L [url]` | Automatically follows HTTP 301/302 redirects | `curl -L http://github.com` |
| `curl -k [url]` | Insecure mode: allows connecting to untrusted/self-signed SSL | `curl -k https://localhost:8443` |
| `nc -zv [host] [port]` | Netcat port scan: tests if remote TCP port is reachable | `nc -zv localhost 80` |
| `ping -c [N] [host]` | Tests ICMP reachability to destination with $N$ packets | `ping -c 4 8.8.8.8` |
| `mtr -t [host]` | Real-time path traceroute and hop loss diagnostics | `mtr -t 8.8.8.8` |

---

## 6. Service Logging & Forensics (`journalctl`)

| Command | Description | Example / Use Case |
| :--- | :--- | :--- |
| `sudo journalctl -u [service]` | Displays all aggregated logs for a specific service unit | `sudo journalctl -u nginx` |
| `sudo journalctl -xeu [service]`| Detailed failure triage with catalog hints at end of log | `sudo journalctl -xeu nginx` |
| `sudo journalctl -u [service] -f` | Follows service log events in real-time (live stream) | `sudo journalctl -u nginx -f` |
| `sudo journalctl -u [service] -n [N]`| Prints the most recent $N$ log entries | `sudo journalctl -u nginx -n 20` |
| `sudo journalctl -u [service] --since "[time]"` | Filters service logs by relative time window | `sudo journalctl -u nginx --since "10 min ago"` |
