# Linux Network Configuration & Troubleshooting Guide

Complete guide for managing network settings on Ubuntu/Linux servers.

## Table of Contents

1. [Network Interface Management](#network-interface-management)
2. [Static IP Configuration](#static-ip-configuration)
3. [DNS Configuration](#dns-configuration)
4. [Network Diagnostics](#network-diagnostics)
5. [IP Routing](#ip-routing)
6. [Network Services](#network-services)
7. [VPN Configuration](#vpn-configuration)
8. [Proxy Configuration](#proxy-configuration)
9. [Network Performance Tuning](#network-performance-tuning)
10. [Troubleshooting](#troubleshooting)

---

## Network Interface Management

### View Network Interfaces

```bash
# Show all interfaces (modern)
ip link show

# Show interface details
ip addr show
ip -4 addr show          # IPv4 only
ip -6 addr show          # IPv6 only

# Legacy command
ifconfig -a
ifconfig eth0

# Show specific interface
ip addr show eth0

# Show interface statistics
ip -s link show

# Show route table
ip route show
route -n
```

### Bring Interface Up/Down

```bash
# Bring interface up
sudo ip link set eth0 up
sudo ifconfig eth0 up

# Bring interface down
sudo ip link set eth0 down
sudo ifconfig eth0 down

# Using systemd-networkd
sudo systemctl restart systemd-networkd
```

### Change MAC Address

```bash
# View current MAC
ip addr show eth0

# Change MAC (temporary - lost on reboot)
sudo ip link set dev eth0 down
sudo ip link set dev eth0 address 00:11:22:33:44:55
sudo ip link set dev eth0 up

# Make permanent (netplan)
sudo nano /etc/netplan/00-installer-config.yaml
# Add under interface:
#   macaddress: 00:11:22:33:44:55
# Then apply:
sudo netplan apply
```

### Set Temporary IP Address

```bash
# Add IP address (doesn't persist)
sudo ip addr add 192.168.1.100/24 dev eth0

# Remove IP address
sudo ip addr del 192.168.1.100/24 dev eth0

# Set as primary (replace existing)
sudo ip addr flush dev eth0
sudo ip addr add 192.168.1.100/24 dev eth0
```

---

## Static IP Configuration

### Netplan Configuration (Ubuntu 18.04+)

```bash
# List network interfaces
ip link show

# Edit netplan configuration
sudo nano /etc/netplan/00-installer-config.yaml

# Example static IP configuration:
# network:
#   version: 2
#   renderer: networkd
#   ethernets:
#     eth0:
#       dhcp4: no
#       addresses:
#         - 192.168.1.100/24
#       gateway4: 192.168.1.1
#       nameservers:
#         addresses: [8.8.8.8, 8.8.4.4]

# Apply changes
sudo netplan apply

# Validate configuration
sudo netplan validate

# Debug netplan
sudo netplan apply --debug
```

### Example: Static IP with Multiple Addresses

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
        - 192.168.1.101/24 # Secondary IP
        - 192.168.1.102/24 # Tertiary IP
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      dhcp6: false
      addresses6:
        - 2001:db8::1/64
```

### Example: DHCP Configuration

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: true
```

### Example: Multiple Interfaces

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    eth1:
      dhcp4: true
```

### Legacy Method: /etc/network/interfaces

```bash
# Edit interfaces file (older Ubuntu/Debian)
sudo nano /etc/network/interfaces

# Example:
# auto eth0
# iface eth0 inet static
#     address 192.168.1.100
#     netmask 255.255.255.0
#     gateway 192.168.1.1
#     dns-nameservers 8.8.8.8 8.8.4.4

# Restart networking
sudo systemctl restart networking
# or
sudo /etc/init.d/networking restart
```

---

## DNS Configuration

### View Current DNS

```bash
# View DNS configuration (modern)
resolvectl status
systemd-resolve --status

# View DNS servers
cat /etc/resolv.conf

# Query specific DNS
nslookup example.com 8.8.8.8
dig @8.8.8.8 example.com
```

### Configure DNS with Netplan

```bash
# Edit netplan config
sudo nano /etc/netplan/00-installer-config.yaml

# Example:
# network:
#   version: 2
#   ethernets:
#     eth0:
#       dhcp4: true
#       dhcp4-overrides:
#         use-dns: false
#       nameservers:
#         addresses:
#           - 8.8.8.8
#           - 8.8.4.4
#           - 1.1.1.1
#         search: [example.com, subdomain.example.com]

# Apply changes
sudo netplan apply
```

### Configure DNS with systemd-resolved

```bash
# Edit resolved configuration
sudo nano /etc/systemd/resolved.conf

# Example configuration:
# [Resolve]
# DNS=8.8.8.8 8.8.4.4 1.1.1.1
# FallbackDNS=9.9.9.9
# Domains=example.com subdomain.example.com
# LLMNR=no
# DNSSEC=no

# Restart systemd-resolved
sudo systemctl restart systemd-resolved

# Verify
resolvectl status
```

### Configure DNS in /etc/resolv.conf (temporary)

```bash
# View current DNS
cat /etc/resolv.conf

# Edit (temporary - will be overwritten by DHCP)
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.4.4" | sudo tee -a /etc/resolv.conf

# Make permanent by disabling DHCP DNS
sudo nano /etc/dhcp/dhclient.conf
# Add: supersede domain-name-servers 8.8.8.8, 8.8.4.4;
```

### DNS Testing

```bash
# Resolve hostname to IP
nslookup example.com
dig example.com
host example.com

# Reverse DNS lookup
nslookup 8.8.8.8
dig -x 8.8.8.8

# Specific DNS server
dig @8.8.8.8 example.com

# Get all DNS records
dig example.com ANY

# Get specific record types
dig example.com A      # IPv4 address
dig example.com AAAA   # IPv6 address
dig example.com MX     # Mail server
dig example.com CNAME  # Alias
dig example.com NS     # Nameserver
```

---

## Network Diagnostics

### Check Connectivity

```bash
# Ping host
ping -c 4 google.com

# Check routing path
traceroute example.com
tracert example.com    # Windows style

# Check if port is reachable
nc -zv example.com 80
telnet example.com 80
timeout 2 bash -c 'cat < /dev/null > /dev/tcp/example.com/80' && echo "Connected" || echo "Failed"
```

### View Connections

```bash
# Show listening ports
netstat -tulpn | grep LISTEN
ss -tulpn | grep LISTEN

# Show all connections
netstat -tulpn
ss -tulpn

# Show specific port
netstat -tulpn | grep :8080
ss -tulpn | grep :8080

# Show established connections
netstat -tulpn | grep ESTABLISHED
ss -tulpn | grep ESTABLISHED

# Show network statistics
netstat -s
ss -s
```

### Process to Port Mapping

```bash
# Find process using specific port
sudo lsof -i :8080
sudo netstat -tulpn | grep :8080

# List all connections by process
sudo netstat -tulpn
sudo ss -tulpn

# Show process info for connection
ps aux | grep processname
```

### Check Bandwidth Usage

```bash
# Monitor real-time traffic
ifstat
iftop -i eth0
nethogs

# Show interface statistics
ip -s link show
netstat -i

# Monitor with watch
watch -n 1 'ip -s link show'
```

### DNS Resolution Testing

```bash
# Resolve domain
nslookup example.com
dig example.com

# Check DNS propagation
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# Get all DNS info
dig example.com +noall +answer

# Trace DNS resolution
dig example.com +trace
```

---

## IP Routing

### View Routing Table

```bash
# Modern command
ip route show
ip route list

# Legacy command
route -n

# Verbose routing information
ip route show detail

# Show IPv6 routes
ip -6 route show
```

### Add Static Route

```bash
# Add route (temporary)
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0

# Add default gateway
sudo ip route add default via 192.168.1.1 dev eth0

# Add route with metric (priority)
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0 metric 100
```

### Delete Route

```bash
# Delete specific route
sudo ip route del 192.168.2.0/24

# Delete default route
sudo ip route del default via 192.168.1.1
```

### Make Routes Permanent (Netplan)

```bash
# Edit netplan
sudo nano /etc/netplan/00-installer-config.yaml

# Example with routes:
# network:
#   version: 2
#   ethernets:
#     eth0:
#       dhcp4: no
#       addresses:
#         - 192.168.1.100/24
#       gateway4: 192.168.1.1
#       routes:
#         - to: 192.168.2.0/24
#           via: 192.168.1.1
#         - to: 10.0.0.0/8
#           via: 192.168.1.2
#           metric: 100

# Apply
sudo netplan apply
```

### Policy Routing

```bash
# Add routing rule
sudo ip rule add from 192.168.1.0/24 table 100

# Add route to table
sudo ip route add 0.0.0.0/0 via 192.168.1.1 table 100

# List rules
ip rule list
```

---

## Network Services

### DHCP Server

```bash
# Install ISC DHCP server
sudo apt install isc-dhcp-server -y

# Configure DHCP
sudo nano /etc/dhcp/dhcpd.conf

# Start service
sudo systemctl start isc-dhcp-server

# Check status
sudo systemctl status isc-dhcp-server

# View DHCP leases
cat /var/lib/dhcp/dhcpd.leases
```

### DHCP Client

```bash
# Get IP via DHCP (temporary)
sudo dhclient eth0

# Release DHCP lease
sudo dhclient -r eth0

# Renew DHCP lease
sudo dhclient -r eth0
sudo dhclient eth0
```

### Bind (DNS Server)

```bash
# Install Bind
sudo apt install bind9 -y

# Configure Bind
sudo nano /etc/bind/named.conf

# Start service
sudo systemctl start bind9

# Test DNS
nslookup example.local localhost
dig @localhost example.local
```

### Dnsmasq (Simple DNS/DHCP)

```bash
# Install Dnsmasq
sudo apt install dnsmasq -y

# Configure
sudo nano /etc/dnsmasq.conf

# Start
sudo systemctl start dnsmasq

# Check status
sudo systemctl status dnsmasq
```

---

## VPN Configuration

### OpenVPN Installation

```bash
# Install OpenVPN
sudo apt install openvpn -y

# Install OpenVPN Easy-RSA (for certificates)
sudo apt install easy-rsa -y

# Copy EasyRSA
cp -r /usr/share/easy-rsa ~/easy-rsa

# Build certificate authority
cd ~/easy-rsa
./easyrsa init-pki
./easyrsa build-ca nopass
```

### OpenVPN Server Setup

```bash
# Generate server certificate
./easyrsa gen-req server nopass
./easyrsa sign-req server server

# Generate DH parameters
./easyrsa gen-dh

# Copy configs to OpenVPN
sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/
sudo cp ~/easy-rsa/pki/issued/server.crt /etc/openvpn/
sudo cp ~/easy-rsa/pki/private/server.key /etc/openvpn/
sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/

# Create server config
sudo nano /etc/openvpn/server.conf

# Start OpenVPN
sudo systemctl start openvpn@server

# Enable on boot
sudo systemctl enable openvpn@server
```

### OpenVPN Client

```bash
# Generate client key
cd ~/easy-rsa
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1

# Create client config
nano client.conf
# Example content:
# client
# proto udp
# remote your-server.com 1194
# ca ca.crt
# cert client1.crt
# key client1.key
# cipher AES-256-CBC
# verb 3

# Connect to VPN
sudo openvpn --config client.conf
```

### WireGuard Installation

```bash
# Install WireGuard
sudo apt install wireguard wireguard-tools -y

# Generate private/public keys
wg genkey | tee privatekey | wg pubkey > publickey

# Create interface configuration
sudo nano /etc/wireguard/wg0.conf
# [Interface]
# Address = 10.0.0.1/24
# SaveCounter = true
# PrivateKey = <content-of-privatekey>
# ListenPort = 51820
#
# [Peer]
# PublicKey = <client-public-key>
# AllowedIPs = 10.0.0.2/32

# Enable WireGuard
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# Check status
sudo wg show
```

---

## Proxy Configuration

### System-wide Proxy (Environment Variables)

```bash
# Set proxy (temporary)
export http_proxy=http://proxy.example.com:8080
export https_proxy=https://proxy.example.com:8080
export ftp_proxy=http://proxy.example.com:8080
export no_proxy="localhost,127.0.0.1,.example.com"

# Set without authentication
export HTTP_PROXY="http://proxy:8080"
export HTTPS_PROXY="https://proxy:8080"

# Set with authentication
export http_proxy="http://user:password@proxy:8080"

# Make permanent
sudo nano /etc/environment
# Add:
# http_proxy=http://proxy.example.com:8080
# https_proxy=https://proxy.example.com:8080
# no_proxy="localhost,127.0.0.1"

# Or edit /etc/profile.d/
sudo nano /etc/profile.d/proxy.sh
# export http_proxy=http://proxy.example.com:8080
```

### APT Proxy Configuration

```bash
# Create APT proxy config
sudo nano /etc/apt/apt.conf.d/proxy.conf

# Example:
# Acquire::http::Proxy "http://proxy.example.com:8080";
# Acquire::https::Proxy "https://proxy.example.com:8080";

# Or with authentication
# Acquire::http::Proxy "http://user:pass@proxy.example.com:8080";
```

### Squid Proxy Server

```bash
# Install Squid
sudo apt install squid -y

# Configure Squid
sudo nano /etc/squid/squid.conf

# Start Squid
sudo systemctl start squid

# Check logs
sudo tail -f /var/log/squid/access.log
```

---

## Network Performance Tuning

### View Network Buffers

```bash
# Show buffer settings
cat /proc/sys/net/ipv4/tcp_rmem
cat /proc/sys/net/ipv4/tcp_wmem

# Current values
sysctl -a | grep tcp

# Show network device settings
ethtool eth0
```

### Increase Buffer Sizes

```bash
# Temporary changes
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# Make permanent
sudo nano /etc/sysctl.conf
# Add:
# net.ipv4.tcp_rmem = 4096 87380 16777216
# net.ipv4.tcp_wmem = 4096 65536 16777216

# Apply
sudo sysctl -p
```

### TCP Optimization

```bash
# Enable TCP fast open
sudo sysctl -w net.ipv4.tcp_fastopen=3

# Increase TCP backlog
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8096

# Increase connection limit
sudo sysctl -w net.core.somaxconn=8096

# Reduce time-wait connections
sudo sysctl -w net.ipv4.tcp_fin_timeout=30

# Reuse connections
sudo sysctl -w net.ipv4.tcp_tw_reuse=1

# Make permanent
sudo nano /etc/sysctl.conf
# Add all settings
sudo sysctl -p
```

---

## Troubleshooting

### No Internet Connection

```bash
# Check interfaces
ip link show

# Bring interface up
sudo ip link set eth0 up

# Check IP address
ip addr show

# Check gateway
ip route show

# Test connectivity
ping 8.8.8.8

# Check DNS
dig google.com

# Check if service is running
systemctl status systemd-networkd
```

### DNS Not Working

```bash
# Check DNS servers
resolvectl status

# Test specific DNS
dig @8.8.8.8 google.com

# Check /etc/resolv.conf
cat /etc/resolv.conf

# Restart DNS resolver
sudo systemctl restart systemd-resolved

# Check DNS cache
systemd-resolve --statistics
systemd-resolve --flush-caches
```

### High Latency

```bash
# Check ping latency
ping -c 10 8.8.8.8 | tail -1

# Check path to host
traceroute -m 20 example.com

# Monitor packet loss
ping -c 100 google.com | tail -1

# Check network quality
mtr example.com
```

### Port Not Listening

```bash
# Check listening ports
ss -tulpn

# Verify service is running
systemctl status servicename

# Check firewall rules
sudo ufw status
sudo iptables -L -n

# Test port locally
nc -zv localhost 8080

# Check service logs
journalctl -u servicename -f
```

### Slow Transfer Speed

```bash
# Check interface speed
ethtool eth0 | grep Speed

# Change duplex mode (if needed)
sudo ethtool -s eth0 speed 1000 duplex full autoneg on

# Check interface errors
ip -s link show eth0

# Monitor bandwidth
iftop -i eth0
nethogs -i eth0
```

---

## Quick Reference

```bash
# Network info
ip addr show
ip route show
ss -tulpn

# Check connectivity
ping 8.8.8.8
nslookup example.com
traceroute example.com

# Manage interfaces
sudo ip link set eth0 up
sudo ip addr add 192.168.1.100/24 dev eth0

# DNS
resolvectl status
nslookup google.com
dig @8.8.8.8 example.com

# Troubleshooting
sudo netstat -tulpn
sudo lsof -i :8080
journalctl -u servicename -f
```
