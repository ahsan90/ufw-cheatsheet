# Ubuntu VPS Server Setup Guide

Complete guide for configuring and securing a fresh Ubuntu VPS instance.

## Table of Contents

1. [Initial System Setup](#initial-system-setup)
2. [User & Permission Management](#user--permission-management)
3. [SSH Security](#ssh-security)
4. [Firewall Configuration](#firewall-configuration)
5. [Package Management](#package-management)
6. [Web Servers](#web-servers)
7. [Databases](#databases)
8. [Monitoring & Logs](#monitoring--logs)
9. [System Maintenance](#system-maintenance)

---

## Initial System Setup

### Update System

```bash
# Update package list
sudo apt update

# Upgrade all packages
sudo apt upgrade -y

# Full distribution upgrade (handles dependency changes)
sudo apt full-upgrade -y

# Auto-remove unused packages
sudo apt autoremove -y
sudo apt autoclean -y
```

### Check System Information

```bash
# System information
uname -a
lsb_release -a
hostnamectl

# Disk space
df -h
du -sh /*

# Memory usage
free -h

# CPU information
nproc
lscpu
```

### Set Hostname

```bash
# View current hostname
hostname

# Change hostname
sudo hostnamectl set-hostname new-hostname

# Edit hosts file
sudo nano /etc/hosts
# Add: 127.0.1.1 new-hostname

# Reboot to apply
sudo reboot
```

### Set Timezone

```bash
# List available timezones
timedatectl list-timezones

# Set timezone
sudo timedatectl set-timezone Asia/Dubai
# or
sudo timedatectl set-timezone America/New_York
sudo timedatectl set-timezone Europe/London

# Verify
timedatectl
```

### Configure Locale

```bash
# Generate locale
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8

# Verify
locale
```

### Create Swap (if needed)

```bash
# Check current swap
swapon --show

# Create 4GB swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent by adding to /etc/fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify
swapon --show
```

---

## User & Permission Management

### Create New User

```bash
# Create user
sudo adduser username

# Add user to sudo group
sudo usermod -aG sudo username

# Verify user exists
id username
```

### Switch Between Users

```bash
# Switch to another user
su - username

# Execute command as another user
sudo -u username command

# Check current user
whoami
id
```

### User Groups Management

```bash
# Create group
sudo groupadd groupname

# Add user to group
sudo usermod -aG groupname username

# Remove user from group
sudo deluser username groupname

# List all groups
getent group

# List groups for a user
groups username
```

### File Permissions

```bash
# Change file owner
sudo chown user:group filename

# Recursively change permissions
sudo chown -R user:group /path/to/directory

# Change file permissions
chmod 644 filename       # rw-r--r--
chmod 755 dirname        # rwxr-xr-x
chmod 700 privatefile    # rwx------

# Set default permissions for new files
umask 0022
```

### Disable Root Login (Security)

```bash
# Lock root account
sudo passwd -l root

# Unlock root (if needed)
sudo passwd -u root
```

---

## SSH Security

### Generate SSH Keys

```bash
# Generate RSA key (4096-bit)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Generate ED25519 key (recommended - faster & more secure)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# List keys
ls -la ~/.ssh/
```

### Copy SSH Key to Server

```bash
# From local machine - copy public key to server
ssh-copy-id -i ~/.ssh/id_rsa.pub username@server_ip

# Manual method (if ssh-copy-id unavailable)
cat ~/.ssh/id_rsa.pub | ssh username@server_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Set correct permissions
ssh username@server_ip 'chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```

### Configure SSH Daemon

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Key configuration options to set:
Port 22                              # Change default port for security
PermitRootLogin no                   # Disable root login
PasswordAuthentication no             # Disable password auth (use keys only)
PubkeyAuthentication yes              # Enable public key auth
StrictModes yes                       # Check key file permissions
MaxAuthTries 3                        # Max login attempts
MaxSessions 5                         # Max sessions per connection
X11Forwarding no                      # Disable X11 forwarding
PermitEmptyPasswords no               # No empty passwords
AllowUsers username1 username2        # Only allow specific users
```

### Apply SSH Changes

```bash
# Validate SSH config syntax
sudo sshd -t

# Restart SSH service
sudo systemctl restart ssh

# Check SSH status
sudo systemctl status ssh

# View SSH logs
sudo tail -f /var/log/auth.log
```

### SSH Connection Tips

```bash
# Connect to server
ssh -i ~/.ssh/id_rsa username@server_ip

# Connect to custom SSH port
ssh -i ~/.ssh/id_rsa -p 2222 username@server_ip

# Copy files via SCP
scp -i ~/.ssh/id_rsa -r local_folder username@server_ip:/remote/path

# Port forwarding
ssh -i ~/.ssh/id_rsa -L 8080:localhost:8080 username@server_ip
```

### SSH Config File

```bash
# Create SSH config for easier connections
cat >> ~/.ssh/config << EOF
Host myserver
    HostName server_ip
    User username
    Port 22
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking accept-new
EOF

# Usage
ssh myserver
```

---

## Firewall Configuration

### UFW (Uncomplicated Firewall)

```bash
# Install UFW
sudo apt install ufw -y

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
sudo ufw status verbose

# List rules with numbers
sudo ufw show added
```

### UFW Rules - Basic

```bash
# Allow SSH (CRITICAL - do this first!)
sudo ufw allow 22/tcp
# or with comment
sudo ufw allow 22/tcp comment 'SSH'

# Allow HTTP
sudo ufw allow 80/tcp

# Allow HTTPS
sudo ufw allow 443/tcp

# Allow specific IP
sudo ufw allow from 192.168.1.100 to any port 22

# Deny specific port
sudo ufw deny 23/tcp
```

### UFW Rules - Applications

```bash
# Allow SSH via app profile
sudo ufw allow OpenSSH

# List available app profiles
sudo ufw app list

# Allow specific app
sudo ufw allow "Nginx Full"
sudo ufw allow "Apache Full"
```

### UFW Rules - Advanced

```bash
# Allow port range
sudo ufw allow 6000:6100/tcp
sudo ufw allow 6000:6100/udp

# Delete rule
sudo ufw delete allow 80/tcp

# Reset firewall
sudo ufw reset

# Disable without disabling
sudo ufw disable
```

### UFW Policy Rules

```bash
# Set default incoming policy (default: deny)
sudo ufw default deny incoming

# Set default outgoing policy (default: allow)
sudo ufw default allow outgoing

# Set default routed policy
sudo ufw default deny routed
```

### UFW Logging

```bash
# Enable logging
sudo ufw logging on

# Set logging level
sudo ufw logging high

# View logs
sudo tail -f /var/log/ufw.log

# Disable logging
sudo ufw logging off
```

---

## Package Management

### APT (Advanced Package Tool)

```bash
# Update package list
sudo apt update

# Install package
sudo apt install package_name -y

# Install multiple packages
sudo apt install package1 package2 package3 -y

# Remove package (keep config)
sudo apt remove package_name -y

# Remove package (delete config)
sudo apt purge package_name -y

# Search for package
apt search package_name

# Show package info
apt show package_name

# List installed packages
apt list --installed
```

### Package Upgrade

```bash
# Upgrade specific package
sudo apt install --only-upgrade package_name

# Upgrade all packages (safe)
sudo apt upgrade -y

# Full upgrade (handles dependency changes)
sudo apt full-upgrade -y

# Check what will be upgraded
apt list --upgradable
```

### Dependency Management

```bash
# Fix broken dependencies
sudo apt --fix-broken install -y

# Fix missing dependencies
sudo apt --fix-missing install -y

# Clean up unused packages
sudo apt autoremove -y
sudo apt autoclean -y

# Clean apt cache
sudo apt clean
```

### Add PPA (Personal Package Archive)

```bash
# Install PPA
sudo apt-get install software-properties-common -y
sudo add-apt-repository ppa:user/ppa-name
sudo apt update

# Remove PPA
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update
```

### Snap Management (Modern Packaging)

```bash
# Install snap package
sudo snap install package_name

# Update snap
sudo snap refresh package_name

# List installed snaps
snap list

# Remove snap
sudo snap remove package_name
```

---

## Web Servers

### Nginx Installation & Configuration

```bash
# Install Nginx
sudo apt install nginx -y

# Check status
sudo systemctl status nginx

# Start/stop/restart Nginx
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Enable on boot
sudo systemctl enable nginx

# Reload configuration (without dropping connections)
sudo systemctl reload nginx

# Test Nginx configuration
sudo nginx -t
```

### Nginx Configuration

```bash
# Main config file
sudo nano /etc/nginx/nginx.conf

# Site configuration
sudo nano /etc/nginx/sites-available/default
sudo nano /etc/nginx/sites-available/mysite

# Enable site
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/

# Disable site
sudo rm /etc/nginx/sites-enabled/mysite

# List enabled sites
ls -la /etc/nginx/sites-enabled/

# Logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### Apache Installation & Configuration

```bash
# Install Apache
sudo apt install apache2 -y

# Check status
sudo systemctl status apache2

# Start/stop/restart Apache
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2

# Enable on boot
sudo systemctl enable apache2

# Reload configuration
sudo systemctl reload apache2

# Test Apache configuration
sudo apache2ctl -t
```

### Apache Configuration

```bash
# Main config file
sudo nano /etc/apache2/apache2.conf

# Site configuration
sudo nano /etc/apache2/sites-available/000-default.conf
sudo nano /etc/apache2/sites-available/mysite.conf

# Enable site
sudo a2ensite mysite

# Disable site
sudo a2dissite mysite

# Enable mod
sudo a2enmod rewrite
sudo a2enmod ssl

# List enabled mods
apache2ctl -M

# Logs
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log
```

---

## Databases

### PostgreSQL

```bash
# Install PostgreSQL
sudo apt install postgresql postgresql-contrib -y

# Check status
sudo systemctl status postgresql

# Access PostgreSQL
sudo -u postgres psql

# Create database
createdb mydbname

# Create user
createuser myuser

# Grant privileges
psql -U postgres -c "ALTER USER myuser WITH PASSWORD 'password';"
psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE mydbname TO myuser;"

# Backup database
pg_dump mydbname > backup.sql

# Restore database
psql mydbname < backup.sql
```

### MySQL/MariaDB

```bash
# Install MariaDB (MySQL fork)
sudo apt install mariadb-server -y

# Install MySQL
sudo apt install mysql-server -y

# Check status
sudo systemctl status mariadb
sudo systemctl status mysql

# Secure installation
sudo mysql_secure_installation

# Access MySQL
mysql -u root -p

# Create database
mysql -u root -p -e "CREATE DATABASE mydbname;"

# Create user
mysql -u root -p -e "CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'password';"

# Grant privileges
mysql -u root -p -e "GRANT ALL PRIVILEGES ON mydbname.* TO 'myuser'@'localhost';"
mysql -u root -p -e "FLUSH PRIVILEGES;"

# Backup database
mysqldump -u root -p mydbname > backup.sql

# Restore database
mysql -u root -p mydbname < backup.sql
```

### MongoDB

```bash
# Import MongoDB GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -

# Add MongoDB repo
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Install MongoDB
sudo apt update
sudo apt install -y mongodb-org

# Start MongoDB
sudo systemctl start mongod

# Check status
sudo systemctl status mongod

# Enable on boot
sudo systemctl enable mongod

# Access MongoDB shell
mongosh
```

### Redis

```bash
# Install Redis
sudo apt install redis-server -y

# Check status
sudo systemctl status redis-server

# Start Redis
sudo systemctl start redis-server

# Enable on boot
sudo systemctl enable redis-server

# Access Redis CLI
redis-cli

# Test Redis
redis-cli ping

# Configure Redis
sudo nano /etc/redis/redis.conf

# View Redis logs
sudo tail -f /var/log/redis/redis-server.log
```

---

## Monitoring & Logs

### System Monitoring

```bash
# Real-time system stats
top
# or
htop

# Disk usage
df -h
du -sh /*

# Memory usage
free -h

# Network statistics
netstat -tulpn
ss -tulpn

# Process list
ps aux | grep processname

# System load
uptime

# Network traffic
iftop
nethogs
```

### Log Management

```bash
# View system logs
journalctl -xe

# Follow system logs
journalctl -f

# View logs for specific service
journalctl -u nginx.service
journalctl -u postgresql.service

# View logs since specific time
journalctl --since "2024-01-01 10:00:00"

# View logs for specific priority
journalctl -p err

# View kernel logs
dmesg
dmesg | tail -20
```

### Logrotate Configuration

```bash
# View logrotate config
cat /etc/logrotate.conf

# Create custom logrotate rule
sudo nano /etc/logrotate.d/myapp

# Example logrotate config:
# /var/log/myapp.log {
#     daily
#     rotate 7
#     compress
#     delaycompress
#     missingok
#     notifempty
#     create 640 root adm
#     sharedscripts
# }

# Test logrotate
sudo logrotate -d /etc/logrotate.conf

# Force logrotate
sudo logrotate -f /etc/logrotate.conf
```

---

## System Maintenance

### Cron Jobs

```bash
# View crontab
crontab -l

# Edit crontab
crontab -e

# List all cron jobs
sudo find /etc/cron* -type f -exec ls -la {} \;

# Cron syntax: minute hour day month weekday command
# Example: Run script daily at 2 AM
# 0 2 * * * /path/to/script.sh

# Example: Run script every hour
# 0 * * * * /path/to/script.sh

# Example: Run script every 15 minutes
# */15 * * * * /path/to/script.sh
```

### Systemd Services

```bash
# List all services
systemctl list-units --type=service

# Start service
sudo systemctl start servicename

# Stop service
sudo systemctl stop servicename

# Restart service
sudo systemctl restart servicename

# Reload service config
sudo systemctl reload servicename

# Enable on boot
sudo systemctl enable servicename

# Disable on boot
sudo systemctl disable servicename

# Check status
sudo systemctl status servicename

# View service logs
journalctl -u servicename -f
```

### Disk Cleanup

```bash
# Remove old packages
sudo apt autoremove -y
sudo apt autoclean -y

# Clear apt cache
sudo apt clean

# Remove old snap versions (save space)
sudo snap set system refresh.retain=2

# Find large files
find / -type f -size +100M 2>/dev/null

# Cleanup systemd journal logs
sudo journalctl --vacuum=30d

# Remove old logs
sudo find /var/log -type f -name "*.log" -mtime +30 -delete
```

### System Backup

```bash
# Backup important files
tar -czf backup.tar.gz /home /etc /opt

# Backup to external drive
sudo rsync -av --delete /home/ /mnt/backup/home/

# Scheduled backup with cron
# 0 2 * * * tar -czf /backups/backup-$(date +\%Y\%m\%d).tar.gz /home /etc

# Restore from tar
tar -xzf backup.tar.gz -C /
```

### Reboot & Shutdown

```bash
# Schedule reboot
sudo reboot
sudo shutdown -r now          # Reboot immediately
sudo shutdown -r +5           # Reboot in 5 minutes
sudo shutdown -r 02:00        # Reboot at specific time

# Schedule shutdown
sudo shutdown -h now          # Shutdown immediately
sudo shutdown -h +2           # Shutdown in 2 hours
sudo shutdown -h 15:30        # Shutdown at specific time

# Cancel scheduled reboot/shutdown
sudo shutdown -c

# Restart specific service
sudo systemctl restart servicename
```

---

## Quick Reference: Common Port Numbers

```
SSH: 22
HTTP: 80
HTTPS: 443
FTP: 21
PostgreSQL: 5432
MySQL: 3306
Redis: 6379
MongoDB: 27017
Node.js: 3000 (default)
```

---

## Useful One-Liners

```bash
# Check all listening ports
sudo netstat -tulpn | grep LISTEN

# Kill process by name
pkill -f processname

# Kill process by port
sudo lsof -ti:8080 | xargs kill -9

# Find and replace in multiple files
find . -type f -name "*.txt" -exec sed -i 's/old/new/g' {} \;

# Generate random password
openssl rand -base64 32

# Check file size
du -h filename

# Monitor directory changes
watch -n 1 'ls -la /path/to/dir'

# Export environment variable
export VAR_NAME=value

# Make script executable
chmod +x script.sh

# Find recently modified files
find /path -type f -mtime -1

# Show disk I/O stats
iostat -x 1

# Network speed test
speedtest-cli
```
