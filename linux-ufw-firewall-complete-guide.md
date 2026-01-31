# Linux UFW Firewall Complete Cheat Sheet

Complete reference guide for UFW (Uncomplicated Firewall) on Ubuntu/Linux systems.

## Table of Contents

1. [Installation & Basic Operations](#installation--basic-operations)
2. [Basic Rules](#basic-rules)
3. [Advanced Rules](#advanced-rules)
4. [Application Profiles](#application-profiles)
5. [Rules Management](#rules-management)
6. [Logging & Debugging](#logging--debugging)
7. [Policies & Defaults](#policies--defaults)
8. [IPv6 Configuration](#ipv6-configuration)
9. [Common Scenarios](#common-scenarios)
10. [Troubleshooting](#troubleshooting)

---

## Installation & Basic Operations

### Install UFW

```bash
# Check if UFW is installed
sudo apt list --installed | grep ufw

# Install UFW
sudo apt install ufw -y

# Verify installation
which ufw
ufw version
```

### Enable/Disable Firewall

```bash
# Enable firewall
sudo ufw enable

# Disable firewall (keeps configuration)
sudo ufw disable

# Reset firewall (delete all rules)
sudo ufw reset

# Reload firewall
sudo ufw reload
```

### Check Status

```bash
# Show status
sudo ufw status

# Show status with numbers (for rule deletion)
sudo ufw status numbered

# Show verbose status
sudo ufw status verbose

# Show added rules only
sudo ufw show added
```

---

## Basic Rules

### Allow Rules

```bash
# Allow incoming SSH
sudo ufw allow ssh
sudo ufw allow 22/tcp

# Allow HTTP traffic
sudo ufw allow http
sudo ufw allow 80/tcp

# Allow HTTPS traffic
sudo ufw allow https
sudo ufw allow 443/tcp

# Allow specific port
sudo ufw allow 8080

# Allow specific port with protocol
sudo ufw allow 8080/tcp
sudo ufw allow 8080/udp

# Allow with comment
sudo ufw allow 22/tcp comment 'SSH access'
sudo ufw allow 80/tcp comment 'HTTP web traffic'
```

### Deny Rules

```bash
# Deny incoming traffic on port
sudo ufw deny 23/tcp

# Deny SSH (not recommended!)
sudo ufw deny ssh

# Deny specific IP
sudo ufw deny from 192.168.1.100
```

### Reject Rules

```bash
# Reject instead of deny (sends rejection packet)
sudo ufw reject 23/tcp

# Reject from specific IP
sudo ufw reject from 192.168.1.100
```

### Allow from Specific IP

```bash
# Allow SSH from specific IP
sudo ufw allow from 192.168.1.100 to any port 22

# Allow specific port from specific IP
sudo ufw allow from 203.0.113.0/24 to any port 443

# Allow all traffic from specific IP
sudo ufw allow from 192.168.1.100

# Allow on specific interface
sudo ufw allow in on eth0 from 192.168.1.0/24
```

### Allow to Specific Interface

```bash
# Allow HTTP on eth0 interface only
sudo ufw allow in on eth0 to any port 80/tcp

# Allow HTTPS on wlan0 interface only
sudo ufw allow in on wlan0 to any port 443/tcp
```

---

## Advanced Rules

### Port Ranges

```bash
# Allow port range
sudo ufw allow 6000:6100/tcp
sudo ufw allow 6000:6100/udp

# Deny port range
sudo ufw deny 20:25/tcp

# Allow high ports for clients
sudo ufw allow 1024:65535/tcp
```

### Protocol Specific Rules

```bash
# Allow specific protocol
sudo ufw allow esp   # IPSec
sudo ufw allow gre   # GRE protocol

# Allow TCP only
sudo ufw allow 5432/tcp comment 'PostgreSQL'

# Allow UDP only
sudo ufw allow 53/udp comment 'DNS'

# Both TCP and UDP
sudo ufw allow 5353  # Both protocols
```

### Complex Rules

```bash
# Allow from network to specific port
sudo ufw allow from 192.168.0.0/16 to any port 3306

# Allow specific IP to specific port
sudo ufw allow from 10.0.0.50 to 10.0.0.1 port 443

# Deny all from subnet except one
sudo ufw deny from 192.168.1.0/24
sudo ufw allow from 192.168.1.100

# Allow from multiple networks
sudo ufw allow from 192.168.1.0/24 to any port 22
sudo ufw allow from 10.0.0.0/8 to any port 22
```

### CIDR Notation Examples

```bash
# Single IP
192.168.1.100/32

# Subnet (/24 = 254 hosts)
192.168.1.0/24

# Large network (/16 = 65,534 hosts)
10.0.0.0/16

# Class A (/8 = 16+ million hosts)
172.0.0.0/8
```

---

## Application Profiles

### List Available Profiles

```bash
# Show all available application profiles
sudo ufw app list

# Show profile details
sudo ufw app info "OpenSSH"
sudo ufw app info "Nginx Full"
sudo ufw app info "Apache Full"
```

### Common Application Profiles

```bash
# OpenSSH (SSH)
sudo ufw allow "OpenSSH"

# Nginx
sudo ufw allow "Nginx Full"      # HTTP + HTTPS
sudo ufw allow "Nginx HTTP"      # HTTP only
sudo ufw allow "Nginx HTTPS"     # HTTPS only

# Apache
sudo ufw allow "Apache Full"     # HTTP + HTTPS
sudo ufw allow "Apache"          # HTTP only
sudo ufw allow "Apache Secure"   # HTTPS only

# Mail
sudo ufw allow "Postfix"
sudo ufw allow "Dovecot"
sudo ufw allow "Dovecot POP3"
sudo ufw allow "Dovecot IMAP"

# FTP
sudo ufw allow "OpenSSH"
sudo ufw allow "Samba"
```

### Create Custom Application Profile

```bash
# Create custom profile
sudo nano /etc/ufw/applications.d/myapp

# Example content:
# [MyApp]
# title=My Application
# description=Custom application profile
# ports=8000,8001/tcp|9000/udp

# Reload UFW to recognize new profile
sudo ufw reload

# Use profile
sudo ufw allow "MyApp"
```

---

## Rules Management

### Delete Rules

```bash
# Delete by rule number (use "status numbered" first)
sudo ufw delete 5

# Delete by rule specification
sudo ufw delete allow 22/tcp
sudo ufw delete allow ssh
sudo ufw delete deny from 192.168.1.100

# Delete rule by index (interactive)
sudo ufw delete allow from 192.168.1.0/24 to any port 443
```

### Insert Rules

```bash
# Insert rule at specific line number
sudo ufw insert 1 allow ssh
sudo ufw insert 2 allow http

# Useful when order matters (first match wins)
```

### View All Rules with Numbers

```bash
# Show rules with numbers for easy deletion
sudo ufw status numbered

# Output example:
#      To                         Action      From
#      --                         ------      ----
# [ 1] 22/tcp                     ALLOW IN    Anywhere
# [ 2] 80/tcp                     ALLOW IN    Anywhere
# [ 3] 443/tcp                    ALLOW IN    Anywhere
```

### Disable/Re-enable Rules

```bash
# UFW doesn't have disable rule feature
# Alternative: delete and recreate when needed

# Or comment out in config file
sudo nano /etc/ufw/user.rules
```

### Show Configuration Files

```bash
# Show before/after rules
sudo cat /etc/ufw/before.rules
sudo cat /etc/ufw/after.rules

# Show custom rules
sudo cat /etc/ufw/user.rules
sudo cat /etc/ufw/user6.rules
```

---

## Logging & Debugging

### Enable Logging

```bash
# Enable firewall logging
sudo ufw logging on

# Disable logging
sudo ufw logging off

# Set logging level
sudo ufw logging high      # Log blocked packets
sudo ufw logging medium    # Log blocked packets and rate-limited connections
sudo ufw logging low       # Log packets on the default policy only
sudo ufw logging full      # Log all packets
```

### View Logs

```bash
# View UFW logs
sudo tail -f /var/log/ufw.log

# View specific number of lines
sudo tail -100 /var/log/ufw.log

# Search logs for specific IP
sudo grep "192.168.1.100" /var/log/ufw.log

# View logs for specific port
sudo grep ":22 " /var/log/ufw.log | tail -20

# Real-time log monitoring
sudo watch -n 1 'tail -20 /var/log/ufw.log'

# Count blocked connections
sudo grep "UFW BLOCK" /var/log/ufw.log | wc -l

# Find top blocked ports
sudo grep "UFW BLOCK" /var/log/ufw.log | awk '{print $NF}' | sort | uniq -c | sort -rn
```

### Test Rules

```bash
# Validate UFW configuration
sudo ufw show added

# Test if port is open (from another machine)
nmap -p 22 your-server-ip

# Test if port is open (local)
sudo netstat -tulpn | grep :22
sudo ss -tulpn | grep :22

# Try connecting to a port
telnet localhost 22
nc -zv localhost 22
```

---

## Policies & Defaults

### Default Policies

```bash
# View current default policies
sudo ufw show raw

# Set default incoming policy (drop or deny)
sudo ufw default deny incoming
sudo ufw default allow incoming  # Not recommended for security

# Set default outgoing policy
sudo ufw default allow outgoing
sudo ufw default deny outgoing   # May break system

# Set default routed policy (for router/gateway)
sudo ufw default deny routed
sudo ufw default allow routed
```

### Policy Chain Direction

```bash
# Incoming = packets coming TO the server
sudo ufw default deny incoming   # Drop all incoming by default
sudo ufw allow 22                # Allow SSH

# Outgoing = packets going FROM the server
sudo ufw default allow outgoing  # Allow all outgoing

# Routed = packets passing THROUGH the server
sudo ufw default deny routed     # Drop all routed packets
```

### Accept/Drop vs Deny/Reject

```bash
# ACCEPT = silently accept packet
# DROP = silently drop packet (stealth)
# DENY = send rejection packet back
# REJECT = send rejection packet back

# UFW uses:
# ALLOW = ACCEPT
# DENY = DROP (ignores)
# REJECT = sends rejection packet

# You cannot change this in UFW directly
# Use iptables for more control
```

---

## IPv6 Configuration

### Enable/Disable IPv6

```bash
# UFW IPv6 support (usually enabled by default)
sudo nano /etc/default/ufw

# Set IPV6=yes for IPv6 support
IPV6=yes

# Reload UFW
sudo ufw reload
```

### IPv6 Rules

```bash
# Allow SSH on IPv6
sudo ufw allow 22/tcp comment 'SSH'  # Applies to both IPv4 and IPv6

# Allow HTTP on IPv6
sudo ufw allow 80/tcp comment 'HTTP'

# IPv6 CIDR notation
sudo ufw allow from 2001:db8::/32 to any port 22

# Specific IPv6 address
sudo ufw allow from 2001:0db8:85a3::8a2e:0370:7334 to any port 22
```

### Check IPv6 Rules

```bash
# View all IPv6 rules
sudo ufw status | grep "v6"

# Check if IPv6 is properly working
ping6 localhost
ip -6 addr show

# Check listening IPv6 ports
sudo netstat -tulpn | grep ":::"
sudo ss -tulpn | grep ":::"
```

---

## Common Scenarios

### Scenario 1: Web Server with SSH

```bash
# Enable firewall
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp comment 'SSH'

# Allow HTTP
sudo ufw allow 80/tcp comment 'HTTP'

# Allow HTTPS
sudo ufw allow 443/tcp comment 'HTTPS'

# Verify
sudo ufw status numbered
```

### Scenario 2: Database Server (restricted access)

```bash
# Enable firewall
sudo ufw enable

# Allow SSH only from admin IP
sudo ufw allow from 192.168.1.100 to any port 22 comment 'Admin SSH'

# Allow database from web server network
sudo ufw allow from 192.168.1.0/24 to any port 5432 comment 'PostgreSQL'

# Deny all other access
sudo ufw default deny incoming
```

### Scenario 3: Application Server

```bash
# Enable firewall
sudo ufw enable

# Allow SSH from specific network
sudo ufw allow from 10.0.0.0/8 to any port 22 comment 'SSH'

# Allow application port from load balancer
sudo ufw allow from 10.0.0.0/8 to any port 3000 comment 'NodeJS App'

# Allow monitoring
sudo ufw allow from 10.0.0.50 to any port 9090 comment 'Prometheus'

# Verify
sudo ufw status
```

### Scenario 4: Docker Host

```bash
# Enable firewall
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp comment 'SSH'

# Allow Docker exposed ports (as needed)
sudo ufw allow 8000:8100/tcp comment 'Docker apps'
sudo ufw allow 8000:8100/udp comment 'Docker apps'

# Note: UFW may not work perfectly with Docker
# See troubleshooting section for Docker-specific fixes
```

### Scenario 5: VPN Server (OpenVPN)

```bash
# Enable firewall
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp comment 'SSH'

# Allow OpenVPN
sudo ufw allow 1194/udp comment 'OpenVPN'

# Allow OpenVPN control channel over TCP (alternative)
sudo ufw allow 1194/tcp comment 'OpenVPN TCP'

# Enable IP forwarding for VPN clients
sudo nano /etc/sysctl.conf
# Uncomment: net.ipv4.ip_forward=1
# Uncomment: net.ipv6.conf.all.forwarding=1
sudo sysctl -p
```

### Scenario 6: Mail Server

```bash
# Enable firewall
sudo ufw enable

# SSH
sudo ufw allow 22/tcp comment 'SSH'

# SMTP
sudo ufw allow 25/tcp comment 'SMTP'
sudo ufw allow 587/tcp comment 'SMTP Submission'

# POP3
sudo ufw allow 110/tcp comment 'POP3'
sudo ufw allow 995/tcp comment 'POP3S'

# IMAP
sudo ufw allow 143/tcp comment 'IMAP'
sudo ufw allow 993/tcp comment 'IMAPS'

# HTTP/HTTPS for webmail
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
```

---

## Troubleshooting

### UFW Not Working

```bash
# Check if UFW is active
sudo ufw status

# Check if service is running
sudo systemctl status ufw

# Restart UFW service
sudo systemctl restart ufw

# Check UFW logs
sudo journalctl -u ufw.service -f

# Check iptables (kernel firewall)
sudo iptables -L -n | head -20
```

### Port Still Blocked After Allowing

```bash
# Verify rule was added
sudo ufw status numbered

# Check if service is actually listening on port
sudo netstat -tulpn | grep :8080
sudo ss -tulpn | grep :8080

# Check if service is running
sudo systemctl status servicename

# Reload UFW
sudo ufw reload

# Check if iptables rules exist
sudo iptables -L -n | grep 8080
```

### IPv6 Issues

```bash
# Check if IPv6 is enabled in UFW
grep IPV6= /etc/default/ufw

# Enable IPv6
sudo sed -i 's/IPV6=no/IPV6=yes/' /etc/default/ufw
sudo ufw reload

# Check IPv6 connectivity
ping6 localhost
```

### Docker Issues with UFW

```bash
# UFW may conflict with Docker's iptables rules
# Solution 1: Add rules before Docker startup
sudo ufw allow from 172.17.0.0/16  # Docker default network

# Solution 2: Edit UFW before rules
sudo nano /etc/ufw/before.rules
# Add before the COMMIT line:
# -A ufw-before-input -i docker0 -j ACCEPT
# -A ufw-before-output -o docker0 -j ACCEPT

# Solution 3: Restart Docker after UFW reload
sudo systemctl restart docker
```

### Performance Issues

```bash
# Check number of rules (many rules = slower)
sudo ufw show added | wc -l

# Optimize by ordering rules (most common first)
sudo ufw status numbered

# Consider moving to iptables for complex rules
```

### Accidentally Blocked SSH

```bash
# If locked out via SSH, use console/physical access
# Or use cloud provider's terminal

# Add SSH rule (via console)
sudo ufw allow 22/tcp

# Re-enable UFW
sudo ufw enable
```

### Reset and Reconfigure

```bash
# Backup current rules
sudo cp /etc/ufw/user.rules /etc/ufw/user.rules.backup

# Reset firewall
sudo ufw reset

# Reconfigure from scratch
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
# ... add other rules
```

---

## Quick Reference Commands

```bash
# Enable/disable
sudo ufw enable
sudo ufw disable
sudo ufw reload

# Status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Allow/deny
sudo ufw allow 22/tcp
sudo ufw deny 23/tcp
sudo ufw allow from 192.168.1.0/24

# Manage rules
sudo ufw delete allow 22
sudo ufw insert 1 allow ssh
sudo ufw show added

# Logging
sudo ufw logging on
sudo tail -f /var/log/ufw.log

# Reset
sudo ufw reset
```

---

## References

- UFW Man Page: `man ufw`
- UFW Tutorial: https://help.ubuntu.com/community/UFW
- IPTables Rules: `/etc/ufw/before.rules`
