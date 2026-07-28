MongoDB 8.0 Installation on Ubuntu 24.04

---

## Step 1: Create LVM Volumes

Check current disk:

```bash
sudo lsblk
sudo vgdisplay ubuntu-vg
```

Create logical volumes:

```bash
sudo lvcreate -L 40G -n mongodb-data ubuntu-vg
sudo lvcreate -L 8G -n mongodb-journal ubuntu-vg
sudo lvcreate -L 3G -n mongodb-logs ubuntu-vg
sudo lvcreate -L 10G -n mongodb-backup ubuntu-vg
```

Verify volumes:

```bash
sudo lvdisplay
```

---

## Step 2: Format and Mount Volumes

Format all volumes:

```bash
sudo mkfs.ext4 /dev/ubuntu-vg/mongodb-data
sudo mkfs.ext4 /dev/ubuntu-vg/mongodb-journal
sudo mkfs.ext4 /dev/ubuntu-vg/mongodb-logs
sudo mkfs.ext4 /dev/ubuntu-vg/mongodb-backup
```

Create mount points:

```bash
sudo mkdir -p /mnt/data/mongodb
sudo mkdir -p /mnt/journal/mongodb
sudo mkdir -p /mnt/logs/mongodb
sudo mkdir -p /mnt/backup/mongodb
```

Add to /etc/fstab:

```bash
sudo tee -a /etc/fstab > /dev/null <<EOF
/dev/ubuntu-vg/mongodb-data /mnt/data/mongodb ext4 defaults 0 2
/dev/ubuntu-vg/mongodb-journal /mnt/journal/mongodb ext4 defaults 0 2
/dev/ubuntu-vg/mongodb-logs /mnt/logs/mongodb ext4 defaults 0 2
/dev/ubuntu-vg/mongodb-backup /mnt/backup/mongodb ext4 defaults 0 2
EOF
```

Mount all volumes:

```bash
sudo mount -a
```

Verify mounts:

```bash
df -h
```

---

## Step 3: Install MongoDB

Update system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install prerequisites:

```bash
sudo apt install -y gnupg curl
```

Import MongoDB GPG key:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
```

Add MongoDB repository:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

Update package index:

```bash
sudo apt update
```

Install MongoDB:

```bash
sudo apt install -y mongodb-org
```

---

## Step 4: Configure MongoDB

Create MongoDB configuration file:

```bash
sudo tee /etc/mongod.conf > /dev/null <<EOF
# MongoDB configuration file
storage:
  dbPath: /mnt/data/mongodb
systemLog:
  destination: file
  logAppend: true
  path: /mnt/logs/mongodb/mongod.log
net:
  port: 27017
  bindIp: 0.0.0.0
security:
  authorization: enabled
EOF
```

---

## Step 5: Set Permissions and Start MongoDB

Set ownership:

```bash
sudo chown -R mongodb:mongodb /mnt/data/mongodb /mnt/logs/mongodb /mnt/backup/mongodb
sudo chmod 755 /mnt/data/mongodb /mnt/logs/mongodb /mnt/backup/mongodb
sudo chmod 777 /mnt/backup/mongodb
```

Start MongoDB:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

Verify status:

```bash
sudo systemctl status mongod
```

---

## Step 6: Create Admin User

Connect to MongoDB:

```bash
mongosh
```

Switch to admin database:

```bash
use admin
```

Create admin user:

```bash
db.createUser({ user: "mongoadmin", pwd: "Today@123", roles: ["root"] })
```

Exit:

```bash
exit
```

---

## Step 7: Enable Authentication

Restart MongoDB to apply authentication:

```bash
sudo systemctl restart mongod
```

Verify it's running:

```bash
sudo systemctl status mongod
```

---

## Step 8: Create Backup User

Connect as admin:

```bash
mongosh -u mongoadmin -p "Today@123" --authenticationDatabase admin
```

Create backup user:

```bash
use admin
db.createUser({ user: "backupuser", pwd: "Today@123", roles: ["backup", "restore"] })
exit
```

---

## Step 9: Configure Firewall

Enable UFW:

```bash
sudo ufw enable
```

Allow SSH (replace 192.168.0.73 with your Windows IP):

```bash
sudo ufw allow from 192.168.0.73 to any port 22
```

Allow MongoDB (replace 192.168.0.73 with your Windows IP):

```bash
sudo ufw allow from 192.168.0.73 to any port 27017
```

Verify firewall:

```bash
sudo ufw status
```

---

## Step 10: Create Backup Script

Create backup script:

```bash
cat > /home/ubuntu/backup-mongodb.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/mnt/backup/mongodb"
DATE=$(date +%F)
mongodump --host localhost --port 27017 --username backupuser --password "Today@123" --authenticationDatabase admin --out $BACKUP_DIR/mongodb-$DATE
echo "Backup completed: $BACKUP_DIR/mongodb-$DATE"
EOF
```

Make executable:

```bash
chmod +x /home/ubuntu/backup-mongodb.sh
```

Test backup:

```bash
/home/ubuntu/backup-mongodb.sh
```

Verify backup:

```bash
ls -la /mnt/backup/mongodb/
```

---

## Step 11: Schedule Daily Backups

Open crontab:

```bash
crontab -e
```

Add this line (backups at 2 AM daily):

```
0 2 * * * /home/ubuntu/backup-mongodb.sh
```

Save and exit.

Verify cron job:

```bash
crontab -l
```

---

## Step 12: Create Admin Connection Script

Create convenience script:

```bash
sudo cat > /usr/local/bin/mongoadmin << 'SCRIPT_EOF'
#!/bin/bash
# MongoDB Admin Connection Script

MONGO_USER="mongoadmin"
MONGO_PASS="Today@123"
MONGO_HOST="localhost"
MONGO_PORT="27017"
MONGO_AUTH_DB="admin"

mongosh -u "$MONGO_USER" -p "$MONGO_PASS" --host "$MONGO_HOST" --port "$MONGO_PORT" --authenticationDatabase "$MONGO_AUTH_DB" --db admin
SCRIPT_EOF
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/mongoadmin
```

Use it:

```bash
mongoadmin
```

---

## Verification

Test admin connection:

```bash
mongoadmin
```

Inside MongoDB shell:

```bash
db.getName()
show dbs
exit
```

Test remote connection from Windows:

```powershell
Test-NetConnection -ComputerName 192.168.0.145 -Port 27017
```

---

## Complete Setup Summary

- MongoDB 8.0 installed and running
- Four separate LVM volumes configured
- Authentication enabled with two users
- Firewall configured for remote access
- Automated daily backups scheduled
- Admin convenience script created

Your MongoDB production-ready environment is installed and ready to use.
