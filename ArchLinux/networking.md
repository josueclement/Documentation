# Networking

## ip (iproute2)

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

# Add an IP address
sudo ip addr add 192.168.1.100/24 dev enp0s3

# Remove an IP address
sudo ip addr del 192.168.1.100/24 dev enp0s3

# Bring interface up / down
sudo ip link set enp0s3 up
sudo ip link set enp0s3 down

# Show link status
ip link show

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

# Show route to a specific destination
ip route get 8.8.8.8

# Show ARP / neighbor table
ip neigh

# Flush addresses on an interface
sudo ip addr flush dev enp0s3

# Show interface statistics
ip -s link show enp0s3

# Show all interfaces in brief
ip -br addr
ip -br link

# Set MTU
sudo ip link set enp0s3 mtu 9000

# Add a VLAN interface
sudo ip link add link enp0s3 name enp0s3.100 type vlan id 100
```

## ss (Socket Statistics)

```bash
# Show all listening TCP ports
ss -tlnp

# Show all listening UDP ports
ss -ulnp

# Show all connections (TCP + UDP)
ss -tunap

# Show established connections only
ss -t state established

# Show connections to a specific port
ss -tnp sport = :443
ss -tnp dport = :443

# Show connections by process name
ss -tnp | grep nginx

# Show socket summary statistics
ss -s

# Show timer information
ss -to

# Filter by address
ss -tn dst 192.168.1.0/24

# Show Unix sockets
ss -xlnp

# Flags reference:
# -t  TCP          -u  UDP
# -l  listening    -a  all (listening + non-listening)
# -n  numeric      -p  show process
# -r  resolve      -s  summary
```

## DNS

```bash
# Resolve hostname
dig example.com
dig example.com +short

# Query specific record types
dig example.com MX
dig example.com AAAA
dig example.com TXT
dig example.com NS

# Use a specific DNS server
dig @8.8.8.8 example.com

# Reverse DNS lookup
dig -x 8.8.8.8

# Trace resolution path
dig +trace example.com

# Show all records
dig example.com ANY +noall +answer

# Simple lookup with host
host example.com
host -t MX example.com

# nslookup
nslookup example.com
nslookup -type=ns example.com

# systemd-resolve status
resolvectl status
resolvectl query example.com

# Flush DNS cache (systemd-resolved)
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
| `/etc/hosts` | Static hostname mappings |
| `/etc/nsswitch.conf` | Name resolution order |
| `/etc/systemd/resolved.conf` | systemd-resolved configuration |

## NetworkManager

```bash
# Show connection status
nmcli general status

# List all connections
nmcli connection show

# List active connections
nmcli connection show --active

# Show WiFi networks
nmcli device wifi list

# Connect to WiFi
nmcli device wifi connect "SSID" password "password"

# Connect to hidden network
nmcli device wifi connect "SSID" password "password" hidden yes

# Disconnect
nmcli device disconnect wlan0

# Show device status
nmcli device status

# Show device details
nmcli device show enp0s3

# Create a static connection
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

# Reload connection files
nmcli connection reload

# Interactive editor
nmcli connection edit "static-eth"

# Show saved WiFi passwords
sudo grep -r psk= /etc/NetworkManager/system-connections/
```

## systemd-networkd

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

```bash
# Enable and start
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved

# Check status
networkctl status
networkctl list

# Show detailed info for an interface
networkctl status enp0s3

# Reload config
sudo networkctl reload
```

## nftables (Firewall)

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

```conf
# /etc/nftables.conf — basic stateful firewall
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Accept established/related connections
        ct state established,related accept

        # Accept loopback
        iifname "lo" accept

        # Accept ICMP
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

```bash
# Load nftables config
sudo nft -f /etc/nftables.conf

# Enable on boot
sudo systemctl enable --now nftables

# Add a rule interactively
sudo nft add rule inet filter input tcp dport 8080 accept

# Delete a rule (find handle first)
sudo nft -a list chain inet filter input
sudo nft delete rule inet filter input handle 12

# Block a specific IP
sudo nft add rule inet filter input ip saddr 10.0.0.5 drop

# Rate limiting
sudo nft add rule inet filter input tcp dport 22 ct state new limit rate 3/minute accept
```

## Network Diagnostics

```bash
# Ping
ping -c 4 8.8.8.8
ping -c 4 -6 google.com          # IPv6

# Traceroute
traceroute google.com
traceroute -n google.com          # numeric only

# MTR (combines ping + traceroute)
mtr google.com
mtr -r -c 10 google.com          # report mode

# Check if a port is open
nc -zv 192.168.1.1 22
nc -zv -w3 192.168.1.1 80-100    # port range scan, 3s timeout

# Test TCP connection
curl -v telnet://192.168.1.1:22

# Check public IP
curl ifconfig.me
curl -4 icanhazip.com
curl -6 icanhazip.com

# Download speed test
curl -o /dev/null -w "%{speed_download}\n" https://speed.hetzner.de/100MB.bin

# Show bandwidth usage per interface
ip -s link

# Monitor bandwidth in real-time (install: pacman -S iftop)
sudo iftop -i enp0s3

# Capture packets
sudo tcpdump -i enp0s3 -c 100
sudo tcpdump -i enp0s3 port 80 -w capture.pcap

# ARP scan local network
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
