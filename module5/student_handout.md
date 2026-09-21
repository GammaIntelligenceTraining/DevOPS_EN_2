# Module 5: Diagnostics, Protection & Logs — Student Handout

A comprehensive engineering reference for Linux perimeter defense with UFW, systemd journal forensics, and wire-level packet inspection with tcpdump.

---

## 1. Linux Firewall Architecture & The Netfilter Stack

Linux firewalls operate directly inside the kernel. Understanding the distinction between the kernel packet-filtering engine and userspace management utilities is essential for modern systems engineering.

![UFW Firewall Packet Flow & DevOps Mechanics](../../assets/module5/ufw_firewall_packet_flow.png)

### 1.1 The Kernel Stack vs. Userspace Tools

- **Netfilter (Kernel Space):** The underlying packet-filtering and Network Address Translation (NAT) engine built into the Linux kernel. Every network packet entering, traversing, or exiting the host triggers Netfilter hooks (`PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`).
- **iptables / nftables (Userspace Engines):** Low-level command-line utilities used to define tables, chains, and packet-matching rules inside Netfilter. While highly expressive, raw syntax is complex to manage manually.
- **UFW (Uncomplicated Firewall):** A streamlined administrative interface designed for Debian and Ubuntu systems. UFW translates high-level administrative intent (such as `allow ssh`) into production-grade iptables and nftables rulesets.

> [!NOTE]
> In cloud infrastructure (such as AWS EC2 or Google Cloud Compute Engine), cloud security groups provide an external virtual perimeter. However, defense-in-depth requires maintaining host-level firewalls (UFW/nftables) inside virtual machines to prevent lateral movement if another node in the private subnet is compromised.

### 1.2 Stateful Inspection

UFW relies on Netfilter's connection tracking subsystem (`conntrack`):
- **Stateful Tracking:** The firewall monitors active bidirectional communication flows, tracking TCP sequence numbers, ephemeral ports, and ICMP request-reply pairings.
- **Return Traffic Automation:** When a client or server initiates an outbound connection (such as `apt update` or `curl https://example.com`), the kernel records the outgoing flow. Inbound reply packets matching the established state are automatically admitted, eliminating the need to open high-numbered ephemeral return ports manually.

---

## 2. Perimeter Configuration with UFW

A production firewall enforces a strict **Default Deny** baseline: all unsolicited incoming traffic is dropped unless an explicit exception is granted.

### 2.1 Default Security Policy

Configure the baseline security posture by rejecting unsolicited inbound packets while permitting outbound connections initiated by local applications:

```bash
# Set baseline incoming policy to drop all unsolicited traffic
# 'default deny incoming': sets Netfilter INPUT chain default policy to DROP all unsolicited packets
sudo ufw default deny incoming

# Set baseline outgoing policy to permit all system-initiated outbound connections
# 'default allow outgoing': sets Netfilter OUTPUT chain default policy to ACCEPT locally generated traffic
sudo ufw default allow outgoing
```

> [!CAUTION]
> On remote servers (such as cloud instances or headless virtual machines), never enable UFW before explicitly permitting SSH (`sudo ufw allow ssh` or `sudo ufw allow 22/tcp`). Enabling the firewall without an active SSH allow rule will permanently sever your administrative session.

### 2.2 Service and Port Rule Syntax

Rules can target predefined service names defined in `/etc/services`, or explicit port and protocol combinations:

Authorize incoming SSH, standard HTTP web traffic, and secure HTTPS traffic using explicit port and protocol directives:

```bash
# Allow SSH by service name (resolves to port 22/tcp via /etc/services database)
sudo ufw allow ssh

# Allow HTTP and HTTPS with explicit numerical port and transport protocol
# '<port>/<protocol>': binds rule exclusively to TCP transport without opening UDP
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow a specific contiguous port range for UDP services
# '<start>:<end>/udp': creates a single stateful Netfilter rule spanning the entire port span
sudo ufw allow 60000:61000/udp
```

### 2.3 Granular Access Control & Subnets

Production environments follow the **Principle of Least Privilege**, restricting administrative and database ports to known IP addresses or internal CIDR blocks:

Restrict administrative access and database ingress to specific trusted management hosts and internal application subnets:

```bash
# Permit SSH exclusively from a single trusted management workstation
# 'from 192.0.2.45': restricts source IP; 'to any': matches local host interfaces
# 'port 22': target port; 'proto tcp': enforces TCP protocol handshake
sudo ufw allow from 192.0.2.45 to any port 22 proto tcp

# Permit PostgreSQL database access exclusively from an internal application subnet
# 'from 10.0.4.0/24': authorizes any host in the 10.0.4.1–10.0.4.254 CIDR block
# 'port 5432': standard PostgreSQL database listener socket
sudo ufw allow from 10.0.4.0/24 to any port 5432 proto tcp

# Restrict ingress traffic on a specific physical or virtual network interface
# 'in on eth1': matches packets arriving exclusively on interface eth1, ignoring eth0/lo
sudo ufw allow in on eth1 to any port 8080 proto tcp
```

### 2.4 Rate Limiting (`ufw limit`)

To protect public-facing authentication endpoints from brute-force dictionary attacks, UFW provides state-aware connection throttling:

Apply connection rate limiting to the SSH listening port to throttle rapid connection attempts:

```bash
# Apply stateful rate limiting to SSH authentication port
# 'limit': dynamically drops source IPs initiating >=6 connection attempts within a 30-second window
# 'ssh/tcp': targets SSH service on TCP port 22
sudo ufw limit ssh/tcp
```

**Throttling Logic:** If an IP address initiates 6 or more connection attempts within a 30-second window, the firewall automatically blocks further packets from that source IP for the remainder of the interval.

### 2.5 Deny vs. Reject

| Directive | Kernel Action | Client Observation | Recommended Environment |
| :--- | :--- | :--- | :--- |
| **`deny` (DROP)** | Silently discards packet without acknowledging the sender. | Connection hangs until application timeout (`Connection timed out`). | Public WAN interfaces; obscures service presence from reconnaissance scanners. |
| **`reject`** | Discards packet and returns a TCP RST or ICMP Port Unreachable packet. | Immediate failure message (`Connection refused`). | Internal LAN subnets; enables rapid debugging and transparent failover. |

Configure explicit deny and reject rules on retired administrative and transfer protocol ports:

```bash
# Silently drop incoming packets on port 23 (Telnet)
# 'deny': DROP action discards packet with zero reply, causing remote scanner to hang and time out
sudo ufw deny 23/tcp

# Actively reject incoming packets on port 21 (FTP)
# 'reject': REJECT action immediately returns TCP RST or ICMP port unreachable to fail fast
sudo ufw reject 21/tcp
```

### 2.6 Priority Rule Ordering (`ufw insert`)

Netfilter processes firewall rules sequentially using **first-match semantics**. As soon as a packet matches a rule, that action is taken, and no further rules are evaluated:

Insert a high-priority deny rule at position 1 to immediately blacklist an offending host before any permissive rules are evaluated:

```bash
# Insert high-priority block rule at index 1 ahead of permissive rules
# 'insert 1': forces rule to the top of the chain so first-match evaluation stops attackers immediately
# 'deny from 198.51.100.25': targets offending source IP; 'to any': blocks access across all local ports
sudo ufw insert 1 deny from 198.51.100.25 to any
```

---

## 3. Firewall Logging & Kernel Ring Buffer

When UFW drops a packet, Netfilter writes a structured audit message into the kernel ring buffer (`/dev/kmsg`).

### 3.1 Enabling Firewall Telemetry

Adjust the firewall logging verbosity to ensure dropped packet headers are captured for diagnostic review:

```bash
# Configure logging verbosity to medium to capture dropped packets
# 'medium': logs all blocked packets matching policy, all invalid packets, and all new connections
sudo ufw logging medium
```

### 3.2 What Gets Logged as `[UFW BLOCK]` vs. What Does Not

UFW applies logging rules depending on how a connection is terminated and whether an explicit rule matches:

- **Default Ingress Policy Blocks (`[UFW BLOCK]` Logged):** Unsolicited packets targeting ports with no explicit rule (such as port 9999) traverse the entire user ruleset without matching. They fall through to the default policy chain (`ufw-after-logging-input`), which automatically logs `[UFW BLOCK]` before dropping the packet.
- **Explicit Deny / Reject Rules (Silent by Default):** When a rule is added without the `log` option (e.g., `sudo ufw deny 8080` or `sudo ufw reject 8081`), UFW drops or rejects the packet immediately inside the user chain (`ufw-user-input`). By default (at logging level `low`), user rules do not log. To record audit entries for explicit rules, append `log` during creation (e.g., `sudo ufw deny log 8080/tcp`).
- **Allowed Traffic (Not Blocked):** Packets matching an active `allow` rule are accepted and never trigger a `[UFW BLOCK]` entry.

### 3.3 Decoding `[UFW BLOCK]` Log Anatomy

Kernel block events contain structured metadata identifying the connection:

```text
[UFW BLOCK] IN=eth0 OUT= MAC=00:15:5d:01:02:03 SRC=192.0.2.100 DST=172.28.16.50 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41235 DF PROTO=TCP SPT=54210 DPT=9090 WINDOW=64240 RES=0x00 SYN URGP=0
```

- **`IN`:** The network interface where the packet arrived (`eth0`).
- **`SRC`:** Source IP address of the transmitting client (`192.0.2.100`).
- **`DST`:** Destination IP address targeted on this host (`172.28.16.50`).
- **`PROTO`:** Transport layer protocol (`TCP` or `UDP`).
- **`SPT`:** Source Port (ephemeral client port assigned by sending OS, e.g. `54210`).
- **`DPT`:** Destination Port (target application port, e.g. `9090`).
- **`SYN`:** TCP flag indicating a new connection handshake attempt.

---

## 4. Systemd Journal Forensics (`journalctl`)

Systemd collects kernel and service logs into a unified, indexed binary journal managed by `systemd-journald`.

![Systemd Journal Forensics and Triage Pipeline](../../assets/module5/journalctl_forensics_pipeline.png)

### 4.1 Telemetry Sources: What Can Be Reviewed Through `journalctl`

`systemd-journald` acts as the single pane of glass for host telemetry by intercepting and indexing messages from multiple subsystems:

- **Kernel Subsystem Telemetry (`/dev/kmsg` via `-k`):**
  - Netfilter packet drop events (`[UFW BLOCK]`)
  - Out-of-Memory (OOM) killer process terminations
  - Storage I/O errors, block device timeouts, and filesystem read-only remounts
  - Hardware device driver faults and physical/virtual network interface state changes
- **System Services & Daemon Standard Streams (`-u <unit>`):**
  - Unit lifecycle transitions (`Starting`, `Started`, `Reloading`, `Deactivated`, `Failed`)
  - Standard output (`stdout`) and standard error (`stderr`) streams emitted by services
  - Configuration parsing errors and unhandled process crashes
- **Security, Authentication & Audit Events:**
  - SSH daemon authentication attempts (`Accepted password`, `Failed password for invalid user`)
  - Administrative command executions invoked via `sudo`
  - User session allocation, seat management, and terminal logins handled by `systemd-logind`
- **Crash & Reboot Post-Mortems (`-b`, `-b -1`):**
  - Cross-boot log retention enabling root-cause diagnosis of unexpected server reboots or kernel panics
  - Boot performance profiling and startup phase timing

### 4.2 Forensic Querying Patterns & Practical Examples

#### 1. Kernel Telemetry & Firewall Investigation

Query the kernel ring buffer to investigate firewall drops and hardware-level issues without loading userland application noise:

```bash
# Query kernel ring buffer for the last 20 firewall block events
# '-k': filters messages originated by the Linux kernel (/dev/kmsg)
# '-n 20': displays only the 20 most recent entries
# '--no-pager': streams output directly to stdout without launching interactive less
sudo journalctl -k -n 20 --no-pager | grep "UFW BLOCK"

# Live stream kernel events in real time as packets strike the firewall
# '-k': kernel telemetry only; '-f' (--follow): continuously tails new entries as they arrive
sudo journalctl -kf

# Filter kernel messages for hardware or driver errors from the current boot
# '-k': kernel only; '-p err': priority level 3 (Error) and above; '-b': current boot only
sudo journalctl -k -p err -b --no-pager
```

#### 2. Severity Priority Filtering for Rapid Triage

Syslog defines 8 standard priority levels (0=`emerg`, 1=`alert`, 2=`crit`, 3=`err`, 4=`warning`, 5=`notice`, 6=`info`, 7=`debug`). Filter system-wide logs to eliminate informational clutter during high-pressure incidents:

```bash
# Display only error and critical messages generated across all services since boot
# '-p err': filters syslog priority level 3 (err) through 0 (emerg); '-b': limits to current boot
sudo journalctl -p err -b --no-pager

# Isolate all warning and error messages generated across the system in the last hour
# '-p warning..err': priority range from 4 (warning) to 3 (err); '--since': relative time cutoff
sudo journalctl -p warning..err --since "1 hour ago" --no-pager
```

#### 3. Time-Scoped Outage Investigation

Scope log output to the exact duration of a reported incident using ISO 8601 timestamps or relative human-readable offsets:

```bash
# Query logs strictly within a historical incident window
# '--since' / '--until': ISO 8601 timestamps defining an exact start and end cutoff for the incident
sudo journalctl --since "2026-09-16 10:00:00" --until "2026-09-16 10:30:00" --no-pager

# Query records generated in the last 30 minutes
# '--since "30m ago"': relative offset calculated backwards from execution time
sudo journalctl --since "30m ago" --no-pager
```

#### 4. Service Diagnostics & Multi-Unit Correlation

Isolate daemon state transitions or correlate multiple interrelated services side-by-side:

```bash
# Filter logs for a specific service unit within a relative time window
# '-u nginx.service': isolates records belonging to the specified systemd unit
# '--since "15 minutes ago"': limits scope to the last 15 minutes; '--no-pager': avoids pager blocking
sudo journalctl -u nginx.service --since "15 minutes ago" --no-pager

# Correlate logs from two interrelated services simultaneously
# '-u ssh.service -u nginx.service': merges records from multiple units in chronological order
sudo journalctl -u ssh.service -u nginx.service --since "1 hour ago" --no-pager

# Triage service startup failure with detailed catalog context
# '-x' (--catalog): enriches log lines with explanatory documentation from system catalog
# '-e' (--pager-end): jumps immediately to the newest entries at the end of the pager
# '-u nginx.service': scopes catalog inspection specifically to the failing unit
sudo journalctl -xeu nginx.service
```

> [!NOTE]
> Service journal queries (`journalctl -u <unit>`) capture systemd process lifecycle events (`Starting`, `Reloading`, `Deactivated`) and standard process streams (`stdout`/`stderr`). If a service has experienced no lifecycle events or restarts within the requested `--since` window, `-- No entries --` is returned. Web request access logs are written directly to `/var/log/nginx/access.log`.

#### 5. Reboot Post-Mortems & Journal Storage Maintenance

Investigate past crash events across reboots and manage journal disk capacity:

```bash
# List all recorded system boots with their timestamps and boot IDs
# '--list-boots': displays boot index (0=current, -1=previous), 32-character boot ID, and time bounds
journalctl --list-boots

# Inspect the final 50 log records before the previous system reboot
# '-b -1': queries the boot cycle immediately preceding the current boot; '-n 50': last 50 lines
sudo journalctl -b -1 -n 50 --no-pager

# Audit total storage capacity consumed by persistent journal archives
# '--disk-usage': calculates disk footprint across active runtime and archived journal files
sudo journalctl --disk-usage

# Prune historical journal archives to keep storage under 100 megabytes
# '--vacuum-size=100M': deletes oldest archived journal files until total footprint is below 100MB
sudo journalctl --vacuum-size=100M
```

---

## 5. Wire Packet Inspection with `tcpdump`

### 5.1 Purpose & Role in Production Engineering

Application logs (`/var/log/nginx/access.log`) and systemd journal records (`journalctl`) only report events after network packets have successfully traversed the operating system's protocol stack into userspace. If a connection is silently dropped by an upstream network hop, an invalid checksum causes packet rejection, or a service fails to establish a TCP handshake, application logs remain entirely silent.

`tcpdump` is an industry-standard wire-level packet sniffer that attaches directly to network interface drivers via `libpcap`. It provides the definitive source of truth regarding network traffic on a host:
- **Isolating Root Cause Boundaries:** Proves whether client packets are physically reaching the server interface or disappearing upstream in cloud security groups, virtual switches, or hardware routers.
- **Handshake Diagnostics:** Verifies the three-way TCP handshake (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK) in real time to distinguish between silent firewall drops, active port rejections (`RST`), and service listening failures.
- **Wire-Level Protocol Forensics:** Inspects raw protocol headers, DNS transaction latencies, and unencrypted application payloads directly from network frames without needing application instrumentation or debug logging flags.

![Wire Packet Inspection and Protocol Anatomy](../../assets/module5/tcpdump_packet_anatomy.png)

### 5.2 Berkeley Packet Filters (BPF)

`tcpdump` compiles user expressions into low-level BPF bytecode executed inside the kernel. This allows the kernel to discard irrelevant packets without copying them to userspace memory.

Capture packets on all interfaces with DNS resolution disabled, target specific ports, and save binary pcap captures:

```bash
# Capture packets on all interfaces without resolving DNS hostnames
# '-i any': captures traffic across all physical, virtual, and loopback interfaces
# '-n': suppresses reverse DNS lookups to avoid packet drop caused by lookup latency
sudo tcpdump -i any -n

# Sniff HTTP traffic and display ASCII payload text
# '-i any': all interfaces; '-A': decodes and displays packet payload bytes in human-readable ASCII
# 'port 80': BPF port filter matching source or destination port 80
sudo tcpdump -i any -A port 80

# Capture TCP packets while strictly excluding administrative SSH traffic
# '-i any': all interfaces; '"tcp and not port 22"': BPF boolean filter preventing terminal feedback loops
sudo tcpdump -i any "tcp and not port 22"

# Capture 20 packets directed at a specific host and save to a pcap capture file
# '-c 20': limits capture to exactly 20 matching packets and exits automatically
# 'host 10.0.0.5': BPF filter matching traffic to/from this IP; '-w': persists raw binary pcap to disk
sudo tcpdump -i any -c 20 host 10.0.0.5 -w /tmp/incident_trace.pcap

# Read and decode an existing binary capture file
# '-n': displays numerical IP addresses; '-r': reads and parses captured frames from a disk file
tcpdump -n -r /tmp/incident_trace.pcap
```

---

## 6. The Outside-In Diagnostic Sequence

When an outage strikes, execute this standardized 5-stage diagnostic hierarchy to pinpoint failures without guessing:

$$\text{Client Test} \longrightarrow \text{Perimeter (UFW)} \longrightarrow \text{Process/Sockets (ss/systemd)} \longrightarrow \text{Syntax/Logs (journalctl)} \longrightarrow \text{Filesystem}$$

1. **Stage 1: Outside Client Test (`curl -I http://<ip>`):**
   - Differentiate between connection timeout (firewall drop), connection refused (no listening socket), or HTTP 500/502 (service crashed).
2. **Stage 2: Perimeter Verification (`sudo ufw status numbered`):**
   - Check if the target port is authorized and verify rule processing order.
3. **Stage 3: Socket & Process Health (`sudo ss -tunlp` & `systemctl status`):**
   - Confirm socket is bound to wildcard `0.0.0.0` (or `*:80`) rather than loopback `127.0.0.1`.
4. **Stage 4: Forensics & Syntax (`sudo journalctl -u <service>` & `nginx -t`):**
   - Inspect daemon logs and validate configuration file syntax.
5. **Stage 5: Remediation & Post-Fix Validation:**
   - Apply the targeted fix and verify end-to-end client connectivity.

---

## 7. Practice & Next Steps
- Keep [linux_commands_cheat_sheet.md](./linux_commands_cheat_sheet.md) open for quick syntax lookups.
- Complete both Core Practice and the DevOps Stretch Challenge in [homework.md](./homework.md).
