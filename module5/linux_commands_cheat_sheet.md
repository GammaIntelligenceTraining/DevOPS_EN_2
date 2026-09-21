# Module 5: Diagnostics, Protection & Logs — Command Cheat Sheet

A concise operational reference for UFW firewall administration, systemd journal forensics, and tcpdump packet analysis.

---

## 1. UFW (Uncomplicated Firewall)

| Command | Category | Description | Enterprise & Production Use Case |
| :--- | :--- | :--- | :--- |
| `sudo ufw status` | Status | Displays firewall operational state and active rules | Basic status audit: verify whether kernel packet filtering is currently active or inactive (`sudo ufw status`) |
| `sudo ufw status verbose` | Status | Shows firewall state, default policies, and logging level | Security compliance check: inspect default ingress/egress policies and active logging verbosity (`sudo ufw status verbose`) |
| `sudo ufw status numbered` | Status | Lists active rules prefixed with bracketed numerical indices | Rule management: inspect exact rule order indices prior to modifying or deleting entries (`sudo ufw status numbered`) |
| `sudo ufw enable` | Lifecycle | Activates firewall and enforces rules across reboots | Production activation: enforce firewall rules and register boot systemd unit (`sudo ufw enable`) |
| `sudo ufw disable` | Lifecycle | Deactivates firewall without modifying configured rules | Emergency bypass: temporarily suspend packet filtering to isolate network routing issues (`sudo ufw disable`) |
| `sudo ufw reset` | Lifecycle | Deactivates firewall and resets all rules to factory defaults | Host decommissioning: purge all custom firewall rulesets before reprovisioning (`sudo ufw reset`) |
| `sudo ufw default <policy> <dir>` | Policy | Sets baseline policy (`deny`, `allow`, `reject`) for incoming or outgoing | Zero-trust baseline: establish default-deny posture on all incoming traffic (`sudo ufw default deny incoming`) |
| `sudo ufw allow <port>/<protocol>` | Rules | Opens a specific port for TCP or UDP traffic | Web tier ingress: permit inbound HTTP or HTTPS traffic on standard production ports (`sudo ufw allow 80/tcp`) |
| `sudo ufw allow <service>` | Rules | Opens port mapped by service name in `/etc/services` | Service authorization: open management ports using standard service mappings (`sudo ufw allow ssh`) |
| `sudo ufw allow from <ip>` | Granular | Permits all ingress traffic originating from specific source IP | Bastion authorization: permit unrestricted administration from dedicated jump host (`sudo ufw allow from 10.0.1.25`) |
| `sudo ufw allow from <cidr> to any port <port>` | Granular | Restricts port access to authorized network subnet | Database segmentation: permit PostgreSQL access exclusively from internal application subnet (`sudo ufw allow from 10.0.4.0/24 to any port 5432 proto tcp`) |
| `sudo ufw limit <service>/<proto>` | Security | Rate-limits connections (throttles IPs with >=6 attempts in 30s) | Brute-force mitigation: throttle SSH authentication abuse on exposed endpoints (`sudo ufw limit ssh/tcp`) |
| `sudo ufw deny <port>/<proto>` | Rules | Silently drops traffic directed at a port (stealth drop) | Port cloaking: silently drop reconnaissance scans on database ports without acknowledging attacker (`sudo ufw deny 3306/tcp`) |
| `sudo ufw reject <port>/<proto>` | Rules | Drops traffic and returns TCP RST or ICMP unreachable | LAN failover: immediately reject connections to retired services so internal clients fail over fast (`sudo ufw reject 8080/tcp`) |
| `sudo ufw insert 1 deny from <ip>` | Security | Inserts high-priority block rule at top of ruleset | Incident response: immediately blacklist active attacking IP at position 1 to preempt allow rules (`sudo ufw insert 1 deny from 198.51.100.25 to any`) |
| `sudo ufw delete <number>` | Management | Deletes rule matching its numerical index in `status numbered` | Policy pruning: remove obsolete firewall rules by index number (`sudo ufw delete 2`) |
| `sudo ufw logging <level>` | Logging | Adjusts logging verbosity (`low`, `medium`, `high`, `full`, `off`) | Forensics tuning: increase logging to medium during incident triage to record dropped packet metadata (`sudo ufw logging medium`) |

---

## 2. Systemd Journal Forensics (`journalctl`)

| Command | Scope | Description | Enterprise & Production Use Case |
| :--- | :--- | :--- | :--- |
| `sudo journalctl -k` | Kernel | Queries kernel ring buffer messages | Firewall forensics: review Netfilter kernel dropped packet records (`[UFW BLOCK]`) (`sudo journalctl -k`) |
| `sudo journalctl -kf` | Kernel | Continuously streams new kernel log events in real time | Live perimeter monitoring: stream kernel packet drops as they happen during port scan attacks (`sudo journalctl -kf`) |
| `sudo journalctl -u <unit>` | Service | Displays log entries recorded by specific systemd unit | Service auditing: review process lifecycle and daemon stream output (`sudo journalctl -u nginx.service`) |
| `sudo journalctl -u <unit1> -u <unit2>` | Correlation | Interleaves log streams from multiple units chronologically | Multi-tier debugging: correlate web proxy and backend service events side-by-side (`sudo journalctl -u ssh.service -u nginx.service`) |
| `sudo journalctl -u <unit> -f` | Service | Live tail of log entries from specific systemd unit | Deployment verification: follow application startup and access events during canary releases (`sudo journalctl -u nginx.service -f`) |
| `sudo journalctl -xeu <unit>` | Triage | Explanatory catalog logs with detailed error context | Crash diagnosis: jump to recent daemon failures with kernel catalog error descriptions (`sudo journalctl -xeu nginx.service`) |
| `sudo journalctl -p <priority> -b` | Priority | Filters messages matching severity level (0-7) since boot | Error triage: filter system events to show only errors and critical warnings (`sudo journalctl -p err -b`) |
| `sudo journalctl --since "<window>"` | Time Window | Displays log messages recorded within relative time window | Incident scoping: review system telemetry generated during a specific incident window (`sudo journalctl --since "15m ago"`) |
| `journalctl --list-boots` | Boot Forensics | Lists all recorded system boot cycles and timestamps | Crash analysis: identify system reboot history and unique boot IDs (`journalctl --list-boots`) |
| `sudo journalctl -b -1` | Boot Forensics | Displays logs from the boot preceding current boot | Post-mortem: investigate logs immediately prior to an unexpected power cycle or reboot (`sudo journalctl -b -1 -n 50`) |
| `sudo journalctl --disk-usage` | Maintenance | Displays total storage consumed by journal archives | Storage management: inspect systemd journal disk footprint to prevent partition exhaustion (`sudo journalctl --disk-usage`) |
| `sudo journalctl --vacuum-size=<size>` | Maintenance | Prunes archived journal files to stay below size threshold | Storage remediation: safely free disk space by capping journal archives at a specific limit (`sudo journalctl --vacuum-size=100M`) |

---

## 3. Wire Packet Inspection (`tcpdump`)

| Command | Category | Description | Enterprise & Production Use Case |
| :--- | :--- | :--- | :--- |
| `sudo tcpdump -i <interface>` | Capture | Sniffs packets on specific network interface | Interface triage: capture raw traffic traversing primary virtual or physical NIC (`sudo tcpdump -i eth0`) |
| `sudo tcpdump -i any` | Capture | Sniffs packets traversing all active network interfaces | Host-wide sniffing: inspect traffic across all interfaces including loopback and bridges (`sudo tcpdump -i any`) |
| `sudo tcpdump -i any -n` | Performance | Disables hostname reverse-DNS resolution | High-load capture: suppress reverse DNS lookups to avoid packet loss during traffic spikes (`sudo tcpdump -i any -n`) |
| `sudo tcpdump -i any -nn` | Performance | Disables both hostname DNS and port name translation | Raw numerical capture: display numerical IPs and port numbers for exact socket analysis (`sudo tcpdump -i any -nn`) |
| `sudo tcpdump -i any -A` | Payload | Decodes and displays packet payload bytes in ASCII text | HTTP inspection: inspect unencrypted HTTP requests, headers, and API bodies on the wire (`sudo tcpdump -i any -A port 80`) |
| `sudo tcpdump -i any -X` | Payload | Displays packet header and payload in hex and ASCII side-by-side | Binary debugging: inspect protocol byte offsets and binary payload structures (`sudo tcpdump -i any -X port 80`) |
| `sudo tcpdump -i any -e` | Link Layer | Prints link-level Ethernet headers showing MAC addresses | Gateway forensics: verify source and destination hardware MAC addresses (`sudo tcpdump -i any -e port 80`) |
| `sudo tcpdump -i any -c <count>` | Limiter | Captures exact number of matching packets and exits | Automation & scripts: sample exactly 10 packets for health checks and exit automatically (`sudo tcpdump -i any -c 10 port 80`) |
| `sudo tcpdump -i any -w <file>` | Persistence | Records raw binary packet stream for Wireshark analysis | Escalation artifact: save pcap file to share with network engineering or security teams (`sudo tcpdump -i any -w trace.pcap port 80`) |
| `tcpdump -nn -r <file.pcap>` | Playback | Reads, filters, and decodes an existing binary pcap file | Offline forensics: analyze historical packet capture files using BPF filters without capturing live (`tcpdump -nn -r trace.pcap`) |
| `sudo tcpdump -i any port 53` | BPF Filter | Captures DNS resolution queries and responses on port 53 | DNS troubleshooting: monitor lookup delays, failed resolutions, or NXDOMAIN responses (`sudo tcpdump -i any -nn port 53`) |
| `sudo tcpdump -i any port <port>` | BPF Filter | Captures traffic where source or destination port matches | Service sniffing: isolate traffic directed at a specific database or web port (`sudo tcpdump -i any port 443`) |
| `sudo tcpdump -i any host <ip>` | BPF Filter | Captures traffic directed to or originating from specific IP | Host tracing: monitor all network interactions with a specific backend microservice or host (`sudo tcpdump -i any host 10.0.0.5`) |
| `sudo tcpdump -i any "tcp and not port 22"` | BPF Filter | Captures TCP traffic while excluding administrative SSH | Terminal safety: sniff application TCP traffic without cluttering output with your active SSH session (`sudo tcpdump -i any "tcp and not port 22"`) |
| `sudo tcpdump -i any "tcp[tcpflags] & tcp-syn != 0"` | TCP Flags | Isolates packets with SYN flag set (connection starts) | Port scan detection: monitor initial connection handshake attempts across ports (`sudo tcpdump -i any "tcp[tcpflags] & tcp-syn != 0"`) |
| `sudo tcpdump -i any "tcp[tcpflags] & tcp-rst != 0"` | TCP Flags | Isolates packets with RST flag set (connection resets) | Failure diagnosis: identify rejected connections or aborted socket sessions (`sudo tcpdump -i any "tcp[tcpflags] & tcp-rst != 0"`) |

---

## 4. Supporting Diagnostic Commands

| Command | Tool | Description | Enterprise & Production Use Case |
| :--- | :--- | :--- | :--- |
| `hostname -I` | Network | Lists all IP addresses assigned to local host interfaces | IP discovery: query assigned internal IP addresses across active network interfaces (`hostname -I`) |
| `ip route show default` | Network | Identifies default gateway IP address (host bridge address) | Virtualization gateway audit: discover Hyper-V vSwitch or Multipass host bridge IP (`ip route show default`) |
| `sudo ss -tunlp` | Sockets | Lists active TCP and UDP listening sockets with bound PIDs | Socket state audit: verify whether target service is bound and listening on expected port (`sudo ss -tunlp \| grep :80`) |
| `curl -I <url>` | HTTP Client | Fetches HTTP response headers to test service reachability | Outside client test: probe web endpoint from client to test for timeouts, refusals, or headers (`curl -I http://172.28.16.50`) |
| `nc -zv <host> <port>` | Transport | Performs TCP port connectivity probe without payload data | Perimeter testing: verify if remote firewall port is open without initiating application handshake (`nc -zv 172.28.16.50 80`) |
