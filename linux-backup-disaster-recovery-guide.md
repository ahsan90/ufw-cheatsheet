# Linux Backup & Disaster Recovery Guide

Complete guide for backing up and recovering data on Ubuntu/Linux systems.

## Table of Contents

1. [Backup Strategies](#backup-strategies)
2. [Backup Tools](#backup-tools)
3. [Full System Backup](#full-system-backup)
4. [File and Directory Backup](#file-and-directory-backup)
5. [Database Backup](#database-backup)
6. [Automated Backups](#automated-backups)
7. [Remote Backups](#remote-backups)
8. [Disaster Recovery](#disaster-recovery)
9. [Restore Procedures](#restore-procedures)
10. [Testing and Verification](#testing-and-verification)

---

## Backup Strategies

### Backup Types

**Full Backup**

- Complete copy of all data
- Time-consuming but simplest to restore
- Use weekly or monthly

**Incremental Backup**

- Only changes since last backup
- Faster and less storage
- Need all incremental backups to restore

**Differential Backup**

- Changes since last full backup
- Balance between full and incremental
- Need full + latest differential

**Backup 3-2-1 Rule**

- 3 copies of important data
- 2 different storage media
- 1 copy offsite

---

## Backup Tools

### Install Common Backup Tools

```bash
# Install tar, gzip, bzip2
sudo apt install tar gzip bzip2 -y

# Install rsync
sudo apt install rsync -y

# Install Duplicity (encrypted backups)
sudo apt install duplicity -y

# Install Bacula (enterprise backup)
sudo apt install bacula -y

# Install Amanda (network backup)
sudo apt install amanda-client amanda-server -y

# Install Borgbackup (deduplication)
sudo apt install borgbackup -y

# Install restic (modern backup)
sudo apt install restic -y
```

---

## Full System Backup

### Backup System with Tar

```bash
# Create full system backup (excluding some directories)
sudo tar --exclude=/proc \
         --exclude=/sys \
         --exclude=/dev \
         --exclude=/run \
         --exclude=/mnt \
         --exclude=/media \
         --exclude=/tmp \
         --exclude=/var/tmp \
         --exclude=/boot/grub \
         -czf /backup/system-backup-$(date +%Y%m%d).tar.gz /

# Backup specific directories
tar -czf backup.tar.gz /home /etc /opt

# Backup with split files (for size limit)
tar -czf - /home /etc | split -b 500M - /backup/backup.tar.gz.

# Create backup with progress
tar -czf backup.tar.gz --verbose /home /etc

# Estimate backup size
du -sh /home /etc /opt
tar -czf - /home /etc /opt | wc -c  # Actual compressed size
```

### Backup System with Rsync

```bash
# Full system backup (safe method)
sudo rsync -a --delete --exclude=/proc \
           --exclude=/sys \
           --exclude=/dev \
           --exclude=/run \
           / /mnt/backup/

# Incremental backup
rsync -a --delete --backup --backup-dir=../backup-$(date +%Y%m%d) \
      --exclude-from=exclude.txt \
      /home/ /mnt/backup/home/

# Progress indication
rsync -a --progress /home /mnt/backup/

# Dry run (see what would be copied)
rsync -a -v --dry-run /home /mnt/backup/
```

### Clone Disk

```bash
# Clone entire disk
sudo dd if=/dev/sda of=/backup/sda-backup.img bs=4M status=progress

# Clone partition
sudo dd if=/dev/sda1 of=/backup/sda1-backup.img bs=4M status=progress

# Clone and compress
sudo dd if=/dev/sda bs=4M | gzip -c > /backup/sda-backup.img.gz

# Restore from image
sudo dd if=/backup/sda-backup.img of=/dev/sda bs=4M status=progress

# More efficient cloning with ddrescue
sudo ddrescue /dev/sda /backup/sda-backup.img
```

---

## File and Directory Backup

### Basic File Backup

```bash
# Copy files
cp -r /home/user/documents /backup/documents

# Copy with progress (GNU cp)
cp -r -v /home/user/documents /backup/documents

# Archive single directory
tar -czf /backup/documents-$(date +%Y%m%d).tar.gz /home/user/documents

# Archive and encrypt
tar -czf - /home/user/documents | openssl enc -aes-256-cbc -out backup.tar.gz.enc

# List tar contents
tar -tzf backup.tar.gz

# Extract specific file from tar
tar -xzf backup.tar.gz path/to/file
```

### Selective Backup

```bash
# Backup files modified in last 7 days
find /home/user -type f -mtime -7 -exec tar -czf backup-7days.tar.gz {} \;

# Backup files by extension
tar -czf backup-docs.tar.gz /home --include="*.pdf" --include="*.doc*" --exclude="*"

# Backup excluding certain patterns
tar -czf backup.tar.gz /home --exclude="*.tmp" --exclude="*.cache" /home

# Show what would be backed up
tar -tzf backup.tar.gz | head -50
```

### Rsync for File Backup

```bash
# Simple rsync backup
rsync -av /home/user /backup/user

# Backup with deletion (mirror)
rsync -av --delete /home/user /backup/user

# Backup with exclusions
rsync -av --exclude='.git' --exclude='node_modules' \
      --exclude='*.tmp' /home/user /backup/user

# Bandwidth limit
rsync -av --bwlimit=1000 /home /backup

# Checksum verification
rsync -av --checksum /home /backup

# Progress and stats
rsync -av --progress --stats /home /backup
```

---

## Database Backup

### PostgreSQL Backup

```bash
# Backup single database
pg_dump dbname > backup.sql

# Backup with compression
pg_dump dbname | gzip > backup.sql.gz

# Backup all databases
pg_dumpall > all-databases.sql

# Backup with custom format (better for large DB)
pg_dump -Fc dbname > backup.dump

# Backup with table of contents
pg_dump -Fc -v dbname > backup.dump

# Restore from SQL
psql dbname < backup.sql

# Restore from dump
pg_restore -d dbname backup.dump

# Restore specific table
pg_restore -d dbname -t tablename backup.dump
```

### MySQL/MariaDB Backup

```bash
# Backup single database
mysqldump -u root -p dbname > backup.sql

# Backup all databases
mysqldump -u root -p --all-databases > all-databases.sql

# Backup with compression
mysqldump -u root -p dbname | gzip > backup.sql.gz

# Backup with progress
mysqldump -u root -p --progress dbname > backup.sql

# Backup with binary logs (point-in-time recovery)
mysqldump -u root -p --single-transaction --flush-logs \
         --master-data=2 dbname > backup.sql

# Restore from backup
mysql -u root -p dbname < backup.sql

# Restore specific table
mysql -u root -p dbname < backup_table.sql
```

### MongoDB Backup

```bash
# Backup single database
mongodump -d dbname -o /backup/mongo/

# Backup all databases
mongodump -o /backup/mongo/

# Backup with compression
mongodump -d dbname --archive=backup.archive --gzip

# Restore from backup
mongorestore /backup/mongo/

# Restore from archive
mongorestore --archive=backup.archive --gzip

# Backup remote MongoDB
mongodump --uri="mongodb://user:pass@host:27017/dbname" -o /backup/
```

### Redis Backup

```bash
# Manual Redis backup (copies dump.rdb)
redis-cli BGSAVE

# Get backup file location
redis-cli CONFIG GET dbfilename
redis-cli CONFIG GET dir

# Backup RDB file
sudo cp /var/lib/redis/dump.rdb /backup/dump.rdb

# Backup with AOF (Append Only File)
redis-cli CONFIG SET appendonly yes

# Backup via BGREWRITEAOF
redis-cli BGREWRITEAOF

# Restore by copying dump.rdb back and restarting
sudo cp /backup/dump.rdb /var/lib/redis/
sudo systemctl restart redis-server
```

---

## Automated Backups

### Cron Job Backup

```bash
# Edit crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * tar -czf /backup/backup-$(date +\%Y\%m\%d).tar.gz /home

# Weekly backup every Sunday at 3 AM
0 3 * * 0 tar -czf /backup/weekly-$(date +\%Y\%m\%d).tar.gz /home

# Monthly backup on 1st at 4 AM
0 4 1 * * tar -czf /backup/monthly-$(date +\%Y\%m\%d).tar.gz /home

# Daily database backup
0 3 * * * mysqldump -u root -ppassword dbname | gzip > /backup/db-$(date +\%Y\%m\%d).sql.gz

# Backup script
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### Create Backup Script

```bash
#!/bin/bash
# /opt/scripts/backup.sh

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/backup.log"

# Create backup directory
mkdir -p $BACKUP_DIR

# Function to log messages
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# Backup directories
BACKUP_SOURCES=("/home" "/etc" "/opt")

for source in "${BACKUP_SOURCES[@]}"; do
    if [ -d "$source" ]; then
        log_message "Starting backup of $source"
        tar -czf "$BACKUP_DIR/backup-$(basename $source)-$DATE.tar.gz" "$source"
        if [ $? -eq 0 ]; then
            log_message "Successfully backed up $source"
        else
            log_message "ERROR: Failed to backup $source"
        fi
    fi
done

# Backup databases
log_message "Starting database backup"
mysqldump -u root -ppassword dbname | gzip > "$BACKUP_DIR/db-$DATE.sql.gz"

# Delete old backups (keep last 30 days)
log_message "Cleaning old backups"
find $BACKUP_DIR -name "backup-*.tar.gz" -mtime +30 -delete

log_message "Backup process completed"
```

### Systemd Timer for Backups

```bash
# Create service file
sudo nano /etc/systemd/system/backup.service

# [Unit]
# Description=System Backup
# After=network.target
#
# [Service]
# Type=oneshot
# ExecStart=/opt/scripts/backup.sh
# User=root

# Create timer file
sudo nano /etc/systemd/system/backup.timer

# [Unit]
# Description=Daily Backup Timer
# Requires=backup.service
#
# [Timer]
# OnCalendar=daily
# OnCalendar=*-*-* 02:00:00
# Persistent=true
#
# [Install]
# WantedBy=timers.target

# Enable and start timer
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

# Check timer status
systemctl list-timers
systemctl status backup.timer
```

---

## Remote Backups

### Backup to Remote Server (SSH)

```bash
# Backup over SSH
tar -czf - /home | ssh user@remote-server "cat > /backup/backup-$(date +%Y%m%d).tar.gz"

# Restore from remote
ssh user@remote-server "cat /backup/backup.tar.gz" | tar -xzf -

# Rsync over SSH
rsync -av -e ssh /home user@remote-server:/backup/

# Rsync with SSH on custom port
rsync -av -e "ssh -p 2222" /home user@remote-server:/backup/
```

### Backup to S3/Cloud

```bash
# Install AWS CLI
sudo apt install awscli -y

# Configure AWS credentials
aws configure

# Backup to S3
tar -czf - /home | aws s3 cp - s3://my-bucket/backups/backup-$(date +%Y%m%d).tar.gz

# Backup with duplicity (encrypted)
duplicity /home s3://s3.amazonaws.com/my-bucket/backup

# List S3 backups
aws s3 ls s3://my-bucket/backups/

# Restore from S3
aws s3 cp s3://my-bucket/backups/backup.tar.gz - | tar -xzf -
```

### Backup to FTP

```bash
# Backup via FTP
tar -czf /tmp/backup.tar.gz /home

# Upload via FTP script
(echo "user username password";
 echo "put /tmp/backup.tar.gz";
 echo "bye") | ftp ftp-server.com

# Or use lftp
lftp ftp-server.com -e "put /tmp/backup.tar.gz; quit"
```

### Backup to NAS/Network Storage

```bash
# Mount NAS via NFS
sudo mount -t nfs nas-server:/export/backup /mnt/nas

# Backup to NAS
rsync -av /home /mnt/nas/backup

# Mount NAS via SMB/CIFS
sudo mount -t cifs //nas-server/backup /mnt/nas -o username=user,password=pass

# Unmount when done
sudo umount /mnt/nas
```

---

## Disaster Recovery

### Prepare for Disaster Recovery

```bash
# Create recovery boot device (USB)
sudo dd if=ubuntu-22.04-live-server-amd64.iso of=/dev/sdb bs=4M status=progress

# Create recovery image with grub
sudo grub-mkrescue -o /tmp/recovery.iso /boot

# Document system configuration
sudo hostnamectl > /backup/system-config.txt
sudo ip addr show >> /backup/system-config.txt
sudo cat /etc/fstab >> /backup/system-config.txt
sudo dmidecode >> /backup/system-config.txt

# Export partition info
sudo sfdisk -d /dev/sda > /backup/sda-partition-table.txt
sudo fdisk -l > /backup/partition-info.txt

# Export GRUB config
sudo grub-mkconfig -o /backup/grub-config.cfg
```

### Recovery Checklist

```
[ ] Boot from recovery media
[ ] Connect to recovery environment
[ ] Mount backup storage
[ ] Restore filesystem/partitions
[ ] Restore system files
[ ] Restore databases
[ ] Restore user data
[ ] Verify data integrity
[ ] Test system functionality
[ ] Update backup status
[ ] Document recovery process
[ ] Post-incident review
```

---

## Restore Procedures

### Restore from Tar Backup

```bash
# List backup contents
tar -tzf backup.tar.gz | head -20

# Extract entire backup
tar -xzf backup.tar.gz -C /

# Extract specific files/directories
tar -xzf backup.tar.gz -C / home/user/documents

# Extract to different location
tar -xzf backup.tar.gz -C /tmp

# Verbose extraction (see progress)
tar -xzvf backup.tar.gz -C /
```

### Restore from Rsync Backup

```bash
# List backup contents
ls -la /backup

# Restore entire directory
rsync -av /backup/home/ /home

# Restore specific files
rsync -av /backup/home/user/documents /home/user

# Restore with verification
rsync -av --checksum /backup/home /
```

### Restore Single File

```bash
# Find file in backup
tar -tzf backup.tar.gz | grep filename

# Extract single file
tar -xzf backup.tar.gz path/to/file

# Restore to original location
tar -xzf backup.tar.gz -C / home/user/file.txt
```

### Restore System from Image

```bash
# Boot from recovery media

# List disks
lsblk

# Restore disk image
sudo dd if=/backup/sda-backup.img of=/dev/sda bs=4M status=progress

# Or with compression
sudo gunzip -c /backup/sda-backup.img.gz | sudo dd of=/dev/sda bs=4M status=progress

# Restore partition
sudo dd if=/backup/sda1-backup.img of=/dev/sda1 bs=4M status=progress
```

---

## Testing and Verification

### Verify Backup Integrity

```bash
# List tar contents without extracting
tar -tzf backup.tar.gz | wc -l

# Check tar file integrity
tar -tzf backup.tar.gz > /dev/null && echo "Backup OK" || echo "Backup corrupted"

# Calculate checksum
md5sum backup.tar.gz > backup.md5
sha256sum backup.tar.gz > backup.sha256

# Verify checksum
md5sum -c backup.md5
sha256sum -c backup.sha256

# Test extraction
tar -tzf backup.tar.gz > /dev/null && tar -xzf backup.tar.gz -C /tmp
```

### Test Restore Procedure

```bash
# Restore to test directory
mkdir /tmp/restore-test
tar -xzf /backup/backup.tar.gz -C /tmp/restore-test

# Verify restored files
ls -la /tmp/restore-test
find /tmp/restore-test -type f | wc -l

# Compare with original
diff -r /home /tmp/restore-test/home

# Cleanup after test
rm -rf /tmp/restore-test
```

### Backup Monitoring

```bash
# Check backup sizes
du -sh /backup/backup-*.tar.gz

# Monitor backup progress
watch -n 5 'du -sh /backup'

# List recent backups
ls -lht /backup | head -10

# Find missing backups
# Create script to verify daily backups exist
```

### Backup Reporting

```bash
#!/bin/bash
# Create backup report

echo "=== Backup Report ==="
echo "Date: $(date)"
echo ""
echo "=== Backup Locations ==="
du -sh /backup/*
echo ""
echo "=== Total Backup Size ==="
du -sh /backup
echo ""
echo "=== Recent Backups ==="
ls -lt /backup | head -10
echo ""
echo "=== Backup Age ==="
find /backup -name "*.tar.gz" -exec ls -l {} \; | awk '{print $9, $6, $7, $8}'
```

---

## Quick Reference

```bash
# Full system backup
sudo tar -czf backup-$(date +%Y%m%d).tar.gz --exclude=/proc --exclude=/sys /

# Backup directories
tar -czf backup.tar.gz /home /etc /opt

# Database backup
mysqldump -u root -p dbname | gzip > backup.sql.gz
pg_dump dbname | gzip > backup.sql.gz

# Restore from tar
tar -xzf backup.tar.gz -C /

# Verify backup
tar -tzf backup.tar.gz > /dev/null && echo "OK"

# Remote backup
rsync -av -e ssh /home user@server:/backup/

# Create cron backup
crontab -e  # Add: 0 2 * * * /opt/scripts/backup.sh
```
