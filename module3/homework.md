# Module 3 Homework: Networking Essentials & Services

## Task 1: DNS Investigation & Trace

Investigate the Domain Name System using command-line diagnostic tools.

### Requirements:
1. Perform a standard DNS lookup for `wikipedia.org` and identify the IPv4 address returned in the answer section.
2. Query only the clean IP address using the short output flag.
3. Discover the mail exchange (MX) servers configured for `wikipedia.org`.
4. Perform a full recursive lookup trace (`+trace`) for `wikipedia.org` to visualize the delegation path from the root nameservers (`.`) down to the authoritative nameservers.
5. Perform a reverse DNS lookup (`-x`) on the IP address `1.1.1.1` to determine its associated domain pointer.

<details>
<summary>Hints (Click to expand)</summary>

- Standard query: `dig <domain>`
- Short output: `dig +short <domain>`
- Record type: `dig <domain> MX`
- Trace path: `dig <domain> +trace`
- Reverse lookup: `dig -x <ip_address>`

</details>

**Deliverable:**
The output of `dig +short wikipedia.org`, the output of `dig wikipedia.org MX`, and the PTR record from `dig -x 1.1.1.1`.

---

## Task 2: Network Interface & Routing Audit

Document your machine's network configuration and default routing gateway.

### Requirements:
1. List all active network interfaces and their physical link states using `ip link`.
2. Display your assigned internal IPv4 address and subnet mask using `ip a`.
3. Identify the loopback interface and note its assigned IP address.
4. Inspect the kernel routing table using `ip r` and locate your Default Gateway IP.
5. Query an external service to discover your outbound public IP address.

<details>
<summary>Hints (Click to expand)</summary>

- Interfaces: `ip link show`
- IP addresses: `ip -br a` or `ip a`
- Routing: `ip r` (look for `default via ...`)
- Public IP: `curl -s ifconfig.me`

</details>

**Deliverable:**
The line from `ip a` showing your primary non-loopback IPv4 address, the `default via` line from `ip r`, and your public IP output.

---

## Task 3: Web Service Deployment with Systemd

Deploy and manage the Nginx web server using the Systemd service manager.

### Requirements:
1. Install the `nginx` web server package using `apt`.
2. Check the initial status of `nginx` to see if it is running or stopped.
3. Start the `nginx` service using `systemctl`.
4. Configure `nginx` so it starts automatically whenever the operating system boots.
5. Inspect the detailed status output to verify that the runtime state is `active (running)` and the boot state is `enabled`.

<details>
<summary>Hints (Click to expand)</summary>

- Installation: `sudo apt update && sudo apt install -y nginx`
- Service status: `systemctl status nginx`
- Start service: `sudo systemctl start nginx`
- Enable service: `sudo systemctl enable nginx`

</details>

**Deliverable:**
The first 6 lines of `systemctl status nginx` showing `Loaded` and `Active` status.

---

## Task 4: Socket Inspection & Port Binding Audit

Verify that the Nginx service has successfully bound to the network port.

### Requirements:
1. Inspect all active listening TCP sockets on your machine using `ss`.
2. Filter the output to display only the socket listening on Port 80.
3. Identify whether Nginx is listening locally (`127.0.0.1`) or on all interfaces (`0.0.0.0` or `*`).
4. Identify the Process ID (PID) associated with the Port 80 socket.
5. Test raw TCP connectivity to Port 80 using `nc` (Netcat).

<details>
<summary>Hints (Click to expand)</summary>

- Socket audit: `sudo ss -tunlp`
- Filter port: `sudo ss -tunlp | grep :80`
- Netcat port scan: `nc -zv localhost 80`

</details>

**Deliverable:**
The filtered output of `sudo ss -tunlp | grep :80` and the output of `nc -zv localhost 80`.

---

## Task 5: Web Forensics & Access Log Analysis

Simulate web client requests and trace their execution in the systemd journal.

### Requirements:
1. Issue an HTTP request to `http://localhost` using `curl` and verify that the default Nginx welcome HTML is returned.
2. Issue an HTTP request fetching **only headers** using `curl -I http://localhost` and record the HTTP status code and Server header.
3. Issue a verbose HTTP request using `curl -v http://localhost` and observe the TCP connection handshake (`Connected to localhost...`).
4. Query the systemd journal for Nginx to review the recorded access log entries corresponding to your curl requests.

<details>
<summary>Hints (Click to expand)</summary>

- Fetch HTML: `curl http://localhost`
- Headers only: `curl -I http://localhost`
- Verbose client: `curl -v http://localhost`
- Journal logs: `sudo journalctl -u nginx -n 10 --no-pager`

</details>

**Deliverable:**
The output of `curl -I http://localhost` and the last 3 log entries from `sudo journalctl -u nginx -n 3 --no-pager`.
