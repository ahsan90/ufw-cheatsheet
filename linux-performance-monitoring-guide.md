# Linux Performance Monitoring & Optimization Guide

Complete guide for monitoring and optimizing Linux server performance.

## Table of Contents

1. [System Monitoring](#system-monitoring)
2. [CPU Monitoring](#cpu-monitoring)
3. [Memory Management](#memory-management)
4. [Disk I/O Analysis](#disk-io-analysis)
5. [Process Management](#process-management)
6. [System Logs](#system-logs)
7. [Performance Tuning](#performance-tuning)
8. [Service Optimization](#service-optimization)
9. [Monitoring Tools](#monitoring-tools)
10. [Troubleshooting Performance Issues](#troubleshooting-performance-issues)

---

## System Monitoring

### Basic System Information

```bash
# System information
uname -a
lsb_release -a
hostnamectl

# CPU information
lscpu
nproc  # Number of processors
cat /proc/cpuinfo

# Memory information
free -h
free -m
cat /proc/meminfo

# Disk space
df -h
df -i  # Inode usage
du -sh /*

# Uptime and load
uptime
cat /proc/loadavg
```

### System Load Explanation

```bash
# uptime command shows: load average: 1.5, 2.3, 3.1
# These are 1-minute, 5-minute, 15-minute averages
# On 4-core system: 4.0 = 100% utilization

# Calculate max load (should be < number of cores)
nproc  # Shows core count
uptime | awk '{print $(NF-2), $(NF-1), $NF}'
```

### Real-time System Monitoring

```bash
# Top - process monitor
top
top -u username
top -p pid1,pid2

# Htop - interactive process monitor
htop
htop -u username
htop -p pid

# Atop - advanced process/system monitor
atop

# Glances - comprehensive monitoring
glances
```

---

## CPU Monitoring

### CPU Usage

```bash
# Current CPU usage by process
top -b -n 1 | head -20

# CPU usage per core
mpstat -P ALL 1

# Watch CPU usage
watch -n 1 'top -bn1 | head -15'

# CPU usage per process (top 10)
ps aux --sort=-%cpu | head -11

# Real-time CPU usage
dstat -c
```

### CPU Affinity

```bash
# Run process on specific CPU core
taskset -c 0 mycommand  # Run on CPU 0
taskset -c 0,2 mycommand  # Run on CPUs 0 and 2

# Change process CPU affinity
taskset -pc 0,1 <PID>

# View CPU affinity of process
taskset -pc <PID>
```

### CPU Throttling and Frequency

```bash
# View CPU frequency scaling
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq

# Set CPU governor (powersave, performance, ondemand, conservative)
echo "performance" | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set all CPUs
for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo "performance" | sudo tee $i
done

# View available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
```

### CPU Interrupts

```bash
# View interrupts
cat /proc/interrupts

# View soft interrupts
cat /proc/softirqs

# Monitor interrupts
watch -n 1 'cat /proc/interrupts'
```

---

## Memory Management

### Memory Usage Analysis

```bash
# Memory usage overview
free -h
free -m
free -g

# Detailed memory info
cat /proc/meminfo

# Memory usage per process (top 10)
ps aux --sort=-%mem | head -11

# Memory usage in human-readable format
ps aux | awk '{print $6/1024 "MB\t" $11}' | sort -rn | head -10

# Virtual memory usage
vmstat 1 5
```

### Cache and Buffer

```bash
# View cache and buffer usage
free -h
# Buffers = disk read cache
# Cached = page cache

# Clear cache (careful - may impact performance temporarily)
sync  # Flush buffers to disk first
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'

# Clear specific caches
sudo sh -c 'echo 1 > /proc/sys/vm/drop_caches'  # Clear page cache
sudo sh -c 'echo 2 > /proc/sys/vm/drop_caches'  # Clear slab objects
```

### Swap Management

```bash
# View swap usage
swapon --show
free -h

# Check swap in use
cat /proc/swaps

# Create swap file (if needed)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent in /etc/fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Adjust swappiness (0-100)
# Higher = more swap, Lower = less swap
cat /proc/sys/vm/swappiness
sudo sysctl -w vm.swappiness=10
# Make permanent
sudo nano /etc/sysctl.conf
# vm.swappiness = 10
sudo sysctl -p
```

### Memory Pages

```bash
# Monitor page faults
cat /proc/vmstat | grep fault

# Real-time memory paging
vmstat 1

# Memory mapping
pmap -x <PID>
```

---

## Disk I/O Analysis

### Disk Usage

```bash
# Disk usage summary
df -h

# Disk usage by directory
du -sh /*
du -sh ./*  # Current directory

# Top 10 largest files
find / -type f -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -10

# Disk usage by partition
lsblk -d -o NAME,SIZE

# Inode usage
df -i
du -i --max-depth=1 /
```

### Disk I/O Monitoring

```bash
# Real-time disk I/O
iostat -x 1
iostat -xd 1  # Device stats only

# Block device statistics
cat /proc/diskstats

# Watch disk I/O
watch -n 1 'iostat -x'

# Per-process I/O statistics
iotop
iotop -u username
iotop -p pid

# Disk performance
fio --name=test --ioengine=libaio --iodepth=16 --rw=read --bs=4k --direct=1 --size=1G --numjobs=1
```

### Disk Latency

```bash
# Check disk latency
cat /proc/diskstats | grep sda

# Monitor latency with blktrace
sudo blktrace -d /dev/sda -o - | blkparse -i -

# Simple latency check
sudo iostat -x 1
# Look at 'await' column (milliseconds)
```

### Filesystem Health

```bash
# Check filesystem
sudo fsck -n /dev/sda1  # Dry run

# Check disk errors (SMART)
sudo smartctl -a /dev/sda
sudo smartctl -t short /dev/sda  # Run test

# Mount filesystem read-only
sudo mount -o remount,ro /

# Repair filesystem (must be unmounted)
sudo fsck.ext4 -y /dev/sda1
```

---

## Process Management

### View Processes

```bash
# All processes
ps aux
ps aux | grep processname

# Process tree
pstree
pstree -p  # With PID
pstree -u  # With user

# Processes using specific port
sudo lsof -i :8080
ss -tulpn | grep :8080

# Process info
ps -o pid,ppid,cmd,etime,user -p <PID>
```

### CPU and Memory by Process

```bash
# Sort by CPU usage
ps aux --sort=-%cpu | head -10

# Sort by memory usage
ps aux --sort=-%mem | head -10

# Sorted by elapsed time
ps aux --sort=-etime | head -10

# Show CPU time per process
ps aux | awk '{print $3, $6, $11}' | sort -rn
```

### Process Priority (Nice)

```bash
# View process priority
ps -l
# NI column = nice value (-20 to 19)

# Start process with nice value
nice -n 10 mycommand  # Lower priority
nice -n -10 mycommand  # Higher priority (need sudo)

# Change running process priority
renice -n 5 -p <PID>
renice -n -5 -p <PID>  # Increase priority

# Change user's all processes
renice -n 5 -u username
```

### Control Process

```bash
# Pause process
kill -STOP <PID>

# Resume process
kill -CONT <PID>

# Send signals
kill -TERM <PID>  # Graceful shutdown
kill -9 <PID>     # Force kill
kill -HUP <PID>   # Reload config
kill -USR1 <PID>  # Custom signal

# Kill by name
pkill -f processname
pkill -u username  # Kill user's processes
```

---

## System Logs

### View System Logs

```bash
# System journal (systemd-journald)
journalctl

# Recent logs
journalctl -n 100
journalctl -f  # Follow new logs

# Since specific time
journalctl --since "2024-01-01 00:00:00"
journalctl --since "1 hour ago"

# Specific service
journalctl -u nginx.service
journalctl -u postgresql.service -f

# Specific priority
journalctl -p err
journalctl -p warning
journalctl -p debug

# Specific unit and priority
journalctl -u mysql.service -p err

# By priority levels: emerg, alert, crit, err, warning, notice, info, debug
```

### Traditional Log Files

```bash
# System log
tail -f /var/log/syslog
tail -f /var/log/messages  # RedHat/CentOS

# Kernel log
dmesg
tail -f /var/log/kern.log

# Authentication log
tail -f /var/log/auth.log

# Service logs
tail -f /var/log/nginx/error.log
tail -f /var/log/mysql/error.log
tail -f /var/log/postgresql/postgresql.log
```

### Parse and Search Logs

```bash
# Count log entries
grep "error" /var/log/syslog | wc -l

# Find specific errors
grep -i "fatal\|error\|warning" /var/log/syslog | tail -20

# Show context
grep -A 5 -B 5 "error" /var/log/syslog

# Time range
grep "Jan 15" /var/log/syslog | head -20

# Real-time search
tail -f /var/log/syslog | grep "error"
```

### Log Rotation

```bash
# View logrotate configuration
cat /etc/logrotate.conf
ls -la /etc/logrotate.d/

# Create custom logrotate rule
sudo nano /etc/logrotate.d/myapp

# Example:
# /var/log/myapp.log {
#   daily
#   rotate 14
#   compress
#   delaycompress
#   notifempty
#   create 640 www-data www-data
#   sharedscripts
#   postrotate
#     systemctl reload myapp
#   endscript
# }

# Test logrotate
sudo logrotate -d /etc/logrotate.conf

# Force rotation
sudo logrotate -f /etc/logrotate.conf
```

---

## Performance Tuning

### Kernel Parameters

```bash
# View all kernel parameters
sysctl -a

# Modify kernel parameter (temporary)
sudo sysctl -w vm.swappiness=10
sudo sysctl -w net.core.somaxconn=65535

# Make permanent
sudo nano /etc/sysctl.conf
# Add: vm.swappiness = 10
sudo sysctl -p  # Reload

# View limits for specific user/process
ulimit -a
```

### File Descriptor Limits

```bash
# View current limit
ulimit -n
cat /proc/sys/fs/file-max

# Increase limit for session
ulimit -n 65536

# Make permanent for user
sudo nano /etc/security/limits.conf
# Add:
# username soft nofile 65536
# username hard nofile 65536

# Make permanent system-wide
sudo nano /etc/sysctl.conf
# Add: fs.file-max = 2097152
sudo sysctl -p
```

### Network Tuning

```bash
# TCP window scaling
sysctl net.ipv4.tcp_window_scaling

# TCP timestamps
sysctl net.ipv4.tcp_timestamps

# Increase TCP backlog
sudo sysctl -w net.core.somaxconn=65535
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# Connection tracking
sysctl net.nf_conntrack_max

# Configure in /etc/sysctl.conf
sudo nano /etc/sysctl.conf
# net.core.somaxconn = 65535
# net.ipv4.tcp_max_syn_backlog = 65535
# net.ipv4.tcp_tw_reuse = 1
# net.ipv4.tcp_fin_timeout = 30
```

### I/O Scheduler

```bash
# View available schedulers
cat /sys/block/sda/queue/scheduler

# Set scheduler (temporary)
echo deadline | sudo tee /sys/block/sda/queue/scheduler

# Make permanent
sudo nano /etc/default/grub
# Add: elevator=deadline
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Scheduler options:
# deadline - good for databases
# cfq - default, fair for multi-user
# noop - simple, good for SSD
```

---

## Service Optimization

### Service Management

```bash
# List all services
systemctl list-units --type=service
systemctl list-units --type=service --all

# Service status
systemctl status nginx

# Start/stop/restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx

# Enable on boot
sudo systemctl enable nginx

# Check if running
sudo systemctl is-active nginx
sudo systemctl is-enabled nginx
```

### View Service Startup Time

```bash
# Show system startup time
systemd-analyze

# Show service startup order
systemd-analyze critical-chain

# Show unit startup time
systemd-analyze blame | head -20

# Show dependencies
systemctl list-dependencies nginx
```

### Service Resource Limits

```bash
# Edit service file
sudo systemctl edit nginx

# Add resource limits:
# [Service]
# MemoryLimit=512M
# MemoryMax=1G
# CPUQuota=50%

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart nginx

# View applied limits
systemctl show -p MemoryLimit nginx
```

---

## Monitoring Tools

### Install Monitoring Tools

```bash
# Install htop
sudo apt install htop -y

# Install iostat
sudo apt install sysstat -y

# Install iotop
sudo apt install iotop -y

# Install nethogs (network monitor)
sudo apt install nethogs -y

# Install dstat
sudo apt install dstat -y

# Install glances (comprehensive)
sudo apt install glances -y

# Install nmon (performance tool)
sudo apt install nmon -y
```

### Munin (Server Monitoring)

```bash
# Install Munin
sudo apt install munin munin-node -y

# Configure
sudo nano /etc/munin/munin.conf

# Start service
sudo systemctl start munin-node
sudo systemctl enable munin-node

# Web interface
# http://localhost/munin/
```

### Collectd (Metric Collector)

```bash
# Install
sudo apt install collectd collectd-utils -y

# Configure
sudo nano /etc/collectd/collectd.conf

# Start
sudo systemctl start collectd
sudo systemctl enable collectd

# View metrics
# Uses RRD files in /var/lib/collectd/rrd/
```

### Prometheus (Metrics Collection)

```bash
# Install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.x.x/prometheus-2.x.x.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
cd prometheus-*/

# Start Prometheus
./prometheus --config.file=prometheus.yml

# Web interface
# http://localhost:9090/
```

---

## Troubleshooting Performance Issues

### High CPU Usage

```bash
# Identify high CPU process
top -b -n 1 | head -15
ps aux --sort=-%cpu | head -10

# Check process details
ps -o pid,cmd,etime,time -p <PID>

# Check if process is in loop
strace -p <PID>  # Trace system calls

# Reduce priority if safe
renice -n 10 -p <PID>

# Kill if necessary
kill -9 <PID>
```

### High Memory Usage

```bash
# Identify high memory process
ps aux --sort=-%mem | head -10

# Check memory details
cat /proc/meminfo
free -h

# View memory map of process
pmap -x <PID>

# Check for memory leaks
valgrind --leak-check=full ./myapp

# Increase swap if needed
# See "Swap Management" section
```

### Disk I/O Bottleneck

```bash
# Check disk I/O
iostat -x 1
iotop

# Look for:
# - High %util (utilization)
# - High await (latency)
# - Many r/s or w/s (operations)

# Find high I/O process
iotop -b -n 1

# Check disk health
sudo smartctl -a /dev/sda
```

### Memory Leak Detection

```bash
# Monitor memory growth
watch -n 1 'ps aux | grep processname'

# Use valgrind (development)
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# Check for zombie processes
ps aux | grep " <defunct>"

# Kill zombie processes (kill parent)
ps aux | grep Z  # Find zombie's parent PID
kill -9 <PARENT_PID>
```

### Load Average High

```bash
# Check what's causing load
top
htop
uptime

# Identify slow processes
ps aux --sort=-etime | head -10

# Check for I/O wait
iostat -x 1

# Check system calls
strace -p <PID> -c

# May need to scale resources or optimize code
```

---

## Quick Diagnostics Script

```bash
#!/bin/bash
# Quick system diagnostics

echo "=== SYSTEM LOAD ==="
uptime

echo -e "\n=== CPU USAGE ==="
ps aux --sort=-%cpu | head -6

echo -e "\n=== MEMORY USAGE ==="
free -h
ps aux --sort=-%mem | head -6

echo -e "\n=== DISK USAGE ==="
df -h
du -sh /* 2>/dev/null | sort -rh | head -10

echo -e "\n=== DISK I/O ==="
iostat -x 1 2 | tail -5

echo -e "\n=== TOP PROCESSES ==="
top -b -n 1 | head -15

echo -e "\n=== NETWORK CONNECTIONS ==="
ss -s

echo -e "\n=== SYSTEM LOGS ==="
journalctl -n 10 -p err
```
