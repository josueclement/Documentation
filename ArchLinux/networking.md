# Networking

Network configuration and diagnostics — `ip`, `ss`, DNS resolution, NetworkManager, `systemd-networkd`, `nftables` firewall rules, and common troubleshooting tools.

## ip (iproute2)

The `ip` command is the modern replacement for `ifconfig`, `route`, and `arp`. It manages network interfaces, addresses, routes, and neighbors. Changes are immediate but non-persistent (lost on reboot unless configured via NetworkManager or systemd-networkd).

### Viewing Interfaces and Addresses

```bash
# Show all interfaces with addresses
ip addr
ip a                         # shorthand

# Show specific interface
ip addr show enp0s3

# Show only IPv4 addresses
ip -4 addr

# Show only IPv6 addresses
ip -6 addr

# Brief one-line-per-interface summary
ip -br addr
ip -br link

# Show interface statistics (RX/TX bytes, errors, drops)
ip -s link show enp0s3
```

### Configuring Addresses

These changes are immediate but temporary — they won't survive a reboot.

```bash
# Add an IP address to an interface
sudo ip addr add 192.168.1.100/24 dev enp0s3

# Remove an IP address
sudo ip addr del 192.168.1.100/24 dev enp0s3

# Flush all addresses on an interface
sudo ip addr flush dev enp0s3
```

### Managing Link State

```bash
# Bring interface up / down
sudo ip link set enp0s3 up
sudo ip link set enp0s3 down

# Show link status
ip link show

# Set MTU
sudo ip link set enp0s3 mtu 9000

# Add a VLAN interface
sudo ip link add link enp0s3 name enp0s3.100 type vlan id 100
```

### Routing

The routing table determines where packets go. The default route is the gateway for all traffic not matching a more specific route.

```bash
# Show routing table
ip route
ip r                         # shorthand

# Show default gateway
ip route show default

# Add a static route
sudo ip route add 10.0.0.0/8 via 192.168.1.1

# Add default gateway
sudo ip route add default via 192.168.1.1

# Delete a route
sudo ip route del 10.0.0.0/8

# Show which route a specific destination would use
ip route get 8.8.8.8
```

### Neighbor Table (ARP)

```bash
# Show ARP / neighbor table
ip neigh
```

## ss (Socket Statistics)

`ss` replaces the older `netstat` for displaying socket information. It shows listening ports, established connections, and which processes own them. The most common invocation is `ss -tlnp` (TCP listening, numeric, with process info).

### Listing Sockets

```bash
# Show all listening TCP ports with process info
ss -tlnp

# Show all listening UDP ports with process info
ss -ulnp

# Show all connections (TCP + UDP, listening + established)
ss -tunap

# Show established connections only
ss -t state established
```

### Filtering

```bash
# Show connections to/from a specific port
ss -tnp sport = :443
ss -tnp dport = :443

# Filter by process name
ss -tnp | grep nginx

# Filter by address
ss -tn dst 192.168.1.0/24

# Show Unix sockets
ss -xlnp
```

### Summary and Timers

```bash
# Show socket summary statistics (total counts by type)
ss -s

# Show timer information on connections
ss -to
```

### Flags Reference

| Flag | Meaning |
|------|---------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-a` | All sockets (listening + non-listening) |
| `-n` | Numeric (don't resolve names) |
| `-p` | Show process using socket |
| `-r` | Resolve addresses to hostnames |
| `-s` | Summary statistics |

## DNS

DNS resolution tools for querying, debugging, and managing name resolution.

### dig

`dig` is the most detailed DNS query tool. It shows the full response including answer, authority, and additional sections.

```bash
# Basic lookup
dig example.com
dig example.com +short       # just the answer

# Query specific record types
dig example.com MX
dig example.com AAAA
dig example.com TXT
dig example.com NS

# Use a specific DNS server
dig @8.8.8.8 example.com

# Reverse DNS lookup (IP → hostname)
dig -x 8.8.8.8

# Trace the full resolution path (root → TLD → authoritative)
dig +trace example.com

# Show all records
dig example.com ANY +noall +answer
```

### Simpler Tools

```bash
# host — quick and simple
host example.com
host -t MX example.com

# nslookup — interactive or one-shot
nslookup example.com
nslookup -type=ns example.com
```

### systemd-resolved

On systems using `systemd-resolved`, `resolvectl` manages DNS resolution, caching, and per-interface DNS config.

```bash
# Show resolver status (servers, domains, protocols)
resolvectl status

# Query a hostname
resolvectl query example.com

# Flush DNS cache
sudo resolvectl flush-caches

# Show DNS cache statistics
resolvectl statistics

# Check current DNS servers
resolvectl dns
```

### DNS Configuration Files

| File | Purpose |
|------|---------|
| `/etc/resolv.conf` | System DNS resolver config (often managed by systemd-resolved or NetworkManager) |
| `/etc/hosts` | Static hostname mappings (checked before DNS) |
| `/etc/nsswitch.conf` | Name resolution order (files, dns, mdns, etc.) |
| `/etc/systemd/resolved.conf` | systemd-resolved configuration |

## NetworkManager

NetworkManager is the default network management tool on most desktop Arch installs. `nmcli` is its command-line interface — it manages WiFi, ethernet, VPN, and other connection types.

### Status and Listing

```bash
# Show connection status
nmcli general status

# List all configured connections
nmcli connection show

# List active connections
nmcli connection show --active

# Show device status (interface → connection mapping)
nmcli device status

# Show detailed device info
nmcli device show enp0s3
```

### WiFi

```bash
# List available WiFi networks
nmcli device wifi list

# Connect to WiFi
nmcli device wifi connect "SSID" password "password"

# Connect to hidden network
nmcli device wifi connect "SSID" password "password" hidden yes

# Disconnect
nmcli device disconnect wlan0
```

### Creating and Modifying Connections

```bash
# Create a static IP connection
nmcli connection add con-name "static-eth" ifname enp0s3 type ethernet \
    ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "8.8.8.8,8.8.4.4" \
    ipv4.method manual

# Modify an existing connection
nmcli connection modify "static-eth" ipv4.dns "1.1.1.1"

# Bring connection up / down
nmcli connection up "static-eth"
nmcli connection down "static-eth"

# Delete a connection
nmcli connection delete "static-eth"

# Reload connection files from disk
nmcli connection reload

# Interactive editor for complex changes
nmcli connection edit "static-eth"
```

### Inspecting Saved Credentials

```bash
# Show saved WiFi passwords
sudo grep -r psk= /etc/NetworkManager/system-connections/
```

## systemd-networkd

`systemd-networkd` is a lightweight network manager suitable for servers and minimal systems. Configuration is declarative via `.network` and `.netdev` files in `/etc/systemd/network/`.

### Basic Configuration

```ini
# /etc/systemd/network/20-wired.network
[Match]
Name=enp0s3

[Network]
DHCP=yes
# Or static:
# Address=192.168.1.100/24
# Gateway=192.168.1.1
# DNS=8.8.8.8
# DNS=8.8.4.4
```

### Bridge Example

Bridges connect multiple interfaces at layer 2. Define the virtual device with `.netdev` and its network settings with `.network`.

```ini
# /etc/systemd/network/25-bridge.netdev
[NetDev]
Name=br0
Kind=bridge

# /etc/systemd/network/25-bridge.network
[Match]
Name=br0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
```

### Managing systemd-networkd

```bash
# Enable and start
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved

# Check status of all interfaces
networkctl status
networkctl list

# Show detailed info for an interface
networkctl status enp0s3

# Reload config files without restarting
sudo networkctl reload
```

## nftables (Firewall)

`nftables` is the modern Linux firewall framework, replacing `iptables`. Rules are organized into tables, chains, and rules. The `inet` family handles both IPv4 and IPv6.

### Inspecting Rules

```bash
# Show current ruleset
sudo nft list ruleset

# Flush all rules
sudo nft flush ruleset

# List tables
sudo nft list tables

# List chains in a table
sudo nft list table inet filter
```

### Basic Stateful Firewall

This example allows established connections, loopback, ICMP, SSH, and HTTP/S — dropping everything else.

```conf
# /etc/nftables.conf
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Accept established/related connections
        ct state established,related accept

        # Accept loopback
        iifname "lo" accept

        # Accept ICMP (ping, etc.)
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # Accept SSH
        tcp dport 22 accept

        # Accept HTTP/HTTPS
        tcp dport { 80, 443 } accept

        # Log and drop everything else
        log prefix "[nft-drop] " drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

### Loading and Managing Rules

```bash
# Load nftables config from file
sudo nft -f /etc/nftables.conf

# Enable on boot
sudo systemctl enable --now nftables

# Add a rule interactively
sudo nft add rule inet filter input tcp dport 8080 accept

# Delete a rule (find handle number first, then delete by handle)
sudo nft -a list chain inet filter input
sudo nft delete rule inet filter input handle 12

# Block a specific IP
sudo nft add rule inet filter input ip saddr 10.0.0.5 drop

# Rate limiting (e.g., 3 new SSH connections per minute)
sudo nft add rule inet filter input tcp dport 22 ct state new limit rate 3/minute accept
```

## Network Diagnostics

Common tools for testing connectivity, tracing routes, and inspecting network behavior.

### Connectivity Testing

```bash
# Ping (ICMP echo)
ping -c 4 8.8.8.8
ping -c 4 -6 google.com          # IPv6

# Traceroute — show each hop to destination
traceroute google.com
traceroute -n google.com          # numeric only (faster)

# MTR — combines ping + traceroute in a live updating view
mtr google.com
mtr -r -c 10 google.com          # report mode (non-interactive)
```

### Port and Connection Testing

```bash
# Check if a port is open
nc -zv 192.168.1.1 22

# Scan a port range with timeout
nc -zv -w3 192.168.1.1 80-100

# Test TCP connection
curl -v telnet://192.168.1.1:22
```

### Public IP and Speed

```bash
# Check public IP
curl ifconfig.me
curl -4 icanhazip.com
curl -6 icanhazip.com

# Download speed test
curl -o /dev/null -w "%{speed_download}\n" https://speed.hetzner.de/100MB.bin
```

### Bandwidth and Packet Monitoring

```bash
# Show bandwidth usage per interface
ip -s link

# Monitor bandwidth in real-time per connection (install: pacman -S iftop)
sudo iftop -i enp0s3

# Capture packets (install: pacman -S tcpdump)
sudo tcpdump -i enp0s3 -c 100
sudo tcpdump -i enp0s3 port 80 -w capture.pcap

# ARP scan local network (install: pacman -S arp-scan)
sudo arp-scan --localnet

# Wake on LAN
wol AA:BB:CC:DD:EE:FF
```

## Useful Files

| Path | Purpose |
|------|---------|
| `/etc/hostname` | System hostname |
| `/etc/hosts` | Static host mappings |
| `/etc/resolv.conf` | DNS resolver configuration |
| `/etc/nsswitch.conf` | Name service switch configuration |
| `/etc/nftables.conf` | nftables firewall rules |
| `/etc/NetworkManager/` | NetworkManager config and connections |
| `/etc/systemd/network/` | systemd-networkd config files |
| `/etc/systemd/resolved.conf` | systemd-resolved configuration |
