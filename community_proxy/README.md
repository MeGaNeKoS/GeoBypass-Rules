## Section 1: Overview

# Proxy Server Setup Documentation

This guide covers the setup of a proxy server environment on Ubuntu using:

- **3proxy** (for HTTP and SOCKS proxy services)
- **HAProxy** (to expose multiple proxy types through a single port)
- **iptables** (to restrict incoming connections and secure the server)
- **tc** (for per-IP bandwidth limiting)

All installation steps, configurations, and management tasks are included for reproducibility and easy maintenance.

---


## Section 2: Prerequisites


## Prerequisites

- Ubuntu Server (tested on 20.04/22.04)
- Root or sudo user access
- Basic Linux command-line knowledge
- Prebuilt `.deb` package for 3proxy (from official GitHub releases)
- Network access to download required packages and tools

---


## 3. Installation

### 3.1. Install 3proxy

1. **Download the Prebuilt .deb File**

   Get the latest release from the [official 3proxy GitHub releases page](https://github.com/z3APA3A/3proxy/releases).

   ```sh
   wget https://github.com/z3APA3A/3proxy/releases/download/0.9.4/3proxy-0.9.4.x86_64.deb
   ```


2. **Install the Package**

   ```sh
   sudo dpkg -i 3proxy-0.9.4.x86_64.deb
   sudo apt-get install -f    # Fix dependencies if needed
   ```

3. **Verify Installation**

   * Binary: `/usr/local/3proxy/bin/3proxy`
   * Default config: `/usr/local/3proxy/conf/3proxy.cfg`
   * Logs (if enabled): `/usr/local/3proxy/logs/`

---

### 3.2. Install HAProxy

```sh
sudo apt update
sudo apt install haproxy
```

---

### 3.3. Install iptables & tc (Optional, for firewalling and bandwidth limiting)

These are usually present on Ubuntu, but to ensure:

```sh
sudo apt install iptables iproute2
```

---

### 3.4. System Overview

At this point, you should have:

* 3proxy installed (HTTP and SOCKS support)
* HAProxy installed (for single port exposure)
* iptables and tc available (for firewalling and bandwidth limiting)

---

## 4. Configuration

### 4.1. 3proxy Configuration

The 3proxy main configuration file is located at:

```
/usr/local/3proxy/conf/3proxy.cfg
```

Below is an example configuration that provides both HTTP and SOCKS proxy services on specified ports, with strong authentication and no logging (for privacy):

```cfg
nscache 65536
nserver 8.8.8.8
nserver 8.8.4.4

# Define users (example)
users proxyuser:CL:proxysecret

# Enable strong authentication
auth strong

# Allow access for the user
allow proxyuser

# Launch HTTP Proxy on port 8080
proxy -p8080

# Launch SOCKS Proxy on port 1080
socks -p1080

# Deny all other access
deny *
```

**Notes:**

* `users` line defines allowed users.
* `auth strong` enables password authentication.
* Logging is disabled (no `log` directive).
* Update `proxyuser` and `proxysecret` with your own values.

---

### 4.2. HAProxy Configuration

The HAProxy configuration file is at:

```
/etc/haproxy/haproxy.cfg
````

Below is an example configuration that exposes both HTTP and SOCKS5 proxies on a single public port (**5445**). Incoming connections are automatically routed to the correct proxy backend based on protocol detection.

```cfg
frontend multiproxy
    bind *:5445
    mode tcp
    tcp-request inspect-delay 2s
    tcp-request content accept if WAIT_END

    acl is_socks5 payload(0,1) -m bin 05

    use_backend socks5_backend if is_socks5
    default_backend http_backend

backend socks5_backend
    mode tcp
    server socks5 127.0.0.1:1080

backend http_backend
    mode tcp
    server http 127.0.0.1:8080
````

**How it works:**

* Listens on port `5445` for all incoming proxy traffic.
* If the first byte of the connection payload is `0x05` (SOCKS5 handshake), forwards to the local SOCKS5 proxy (`127.0.0.1:1080`).
* All other traffic is forwarded to the local HTTP proxy (`127.0.0.1:8080`).

**After editing, reload HAProxy:**

```sh
sudo systemctl reload haproxy
```

---

### 4.4. Firewall Setup (iptables, Optional but recommended)

To secure the server, set up firewall rules to **only allow** SSH and your proxy port, and **block everything else**.

Below is a simple script to reset and configure your iptables firewall accordingly:

```bash
#!/bin/bash

# === Set your allowed ports here ===
SSH_PORT=22
PROXY_PORT=5445

# === Flush old rules ===
iptables -F
iptables -X
iptables -t mangle -F
iptables -t mangle -X

# === Set default policies to DROP everything ===
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# === Allow all loopback traffic ===
iptables -A INPUT -i lo -j ACCEPT

# === Allow established and related connections ===
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# === Allow SSH and Proxy ports ===
iptables -A INPUT -p tcp --dport $SSH_PORT -j ACCEPT
iptables -A INPUT -p tcp --dport $PROXY_PORT -j ACCEPT

# === Drop everything else explicitly (for clarity) ===
iptables -A INPUT -j DROP

echo "Firewall rules applied: Only SSH (port $SSH_PORT) and Proxy (port $PROXY_PORT) allowed."
````

#### Save the rules (Ubuntu/Debian):

Install `iptables-persistent` to auto-load rules after reboot:

```sh
sudo apt-get install iptables-persistent
sudo netfilter-persistent save
```

or, manually:

```sh
sudo sh -c 'iptables-save > /etc/iptables/rules.v4'
```

#### To list or verify rules:

```sh
sudo iptables -L -n -v
```

---

**Customize as needed:**

* Change `PROXY_PORT` if your proxy moves to a different port.
* Add additional `iptables -A INPUT -p tcp --dport ... -j ACCEPT` lines for other ports you wish to allow.

---

**Note:**
These rules do **not** interfere with your `tc`/bandwidth-limiting rules, since those are in the `mangle` table for marking, not filtering.

---

### 4.5. Per-IP Bandwidth Limiting with tc & iptables (Optional)

This section explains how to limit the bandwidth usage per client IP using Linux `tc` (traffic control) and `iptables` marks.

#### How it works:

- Only traffic on your proxy port (e.g., `5445`) is affected.
- `iptables` marks all proxy connections.
- `tc` (traffic control) applies per-IP upload and download limits.
- A helper script keeps these rules up-to-date as client IPs change.

---

#### Main setup script: `/usr/local/sbin/setup_proxy_bw_limit.sh`

Below is a **customizable script** for your use:

```bash
#!/bin/bash

IFACE="eth0"             # Your network interface (e.g., eth0, ens3, etc.)
PROXY_PORT=5445          # The port your proxy listens on
LIMIT="10mbit"           # Per-IP bandwidth limit
TOTAL_BW="1000mbit"          # Total maximum bandwidth for all users combined

HELPER_SCRIPT="/usr/local/sbin/per_ip_limit.sh"

echo "=== Setting up per-IP bandwidth limit for proxy port $PROXY_PORT on $IFACE ==="

# 1. Clean up old tc and iptables rules
tc qdisc del dev $IFACE root 2>/dev/null
tc qdisc del dev ifb0 root 2>/dev/null
tc qdisc del dev $IFACE ingress 2>/dev/null

iptables -t mangle -D OUTPUT -p tcp --sport $PROXY_PORT -j MARK --set-mark 10 2>/dev/null
iptables -t mangle -D PREROUTING -p tcp --dport $PROXY_PORT -j MARK --set-mark 10 2>/dev/null

# 2. Load and set up IFB device for ingress (download) shaping
modprobe ifb
ip link add ifb0 type ifb 2>/dev/null
ip link set ifb0 up

tc qdisc add dev $IFACE ingress
tc filter add dev $IFACE parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

# 3. Set up root qdiscs and parent classes
tc qdisc add dev $IFACE root handle 1: htb default 30 r2q 30
tc class add dev $IFACE parent 1: classid 1:1 htb rate $TOTAL_BW ceil $TOTAL_BW

tc qdisc add dev ifb0 root handle 1: htb default 30 r2q 30
tc class add dev ifb0 parent 1:classid 1:1 htb rate $TOTAL_BW ceil $TOTAL_BW

# 4. Add iptables marks for proxy port
iptables -t mangle -A OUTPUT -p tcp --sport $PROXY_PORT -j MARK --set-mark 10
iptables -t mangle -A PREROUTING -p tcp --dport $PROXY_PORT -j MARK --set-mark 10

# 5. Create the per-IP limit helper script
cat << 'EOF' > $HELPER_SCRIPT
#!/bin/bash
IFACE="eth0"
LIMIT="10mbit"
PROXY_PORT=5445
IFB_IFACE="ifb0"

# Get all unique client IPs connected to the proxy port
IPS=$(netstat -ntu | awk -v port=":$PROXY_PORT\$" '$4 ~ port {print $5}' | cut -d: -f1 | sort -u)

for ip in $IPS; do
  # Use the last two octets for a unique classid
  minor=$(echo $ip | awk -F. '{printf "%02x%02x", $3, $4}')
  classid="1:${minor}"

  # Download limit (egress on eth0)
  tc class add dev $IFACE parent 1:1 classid $classid htb rate $LIMIT ceil $LIMIT 2>/dev/null
  tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 match ip dst $ip match mark 10 0xff flowid $classid 2>/dev/null

  # Upload limit (egress on ifb0)
  tc class add dev $IFB_IFACE parent 1:1 classid $classid htb rate $LIMIT ceil $LIMIT 2>/dev/null
  tc filter add dev $IFB_IFACE protocol ip parent 1:0 prio 1 u32 match ip src $ip match mark 10 0xff flowid $classid 2>/dev/null
done
EOF

chmod +x $HELPER_SCRIPT

# 6. Run the helper script now to apply limits for current connections
sudo $HELPER_SCRIPT

# 7. Schedule the helper script to run every minute (to keep rules updated for new clients)
(crontab -l 2>/dev/null; echo "* * * * * $HELPER_SCRIPT") | crontab -

echo "=== Bandwidth limit setup complete! Each IP is limited to $LIMIT on proxy port $PROXY_PORT ==="
echo "Run '$HELPER_SCRIPT' manually to update limits for current connections at any time."
````

---

**Tips:**

* Adjust `IFACE`, `PROXY_PORT`, and `LIMIT` at the top of the script for your needs.
* The helper script can be run manually to update rules for new client IPs.
* The cron job ensures limits are refreshed automatically every minute.

**To check tc status:**

```sh
tc -s qdisc show dev eth0
tc -s qdisc show dev ifb0
```

**To reset all limits:**

```sh
tc qdisc del dev eth0 root
tc qdisc del dev ifb0 root
tc qdisc del dev eth0 ingress
```

---
