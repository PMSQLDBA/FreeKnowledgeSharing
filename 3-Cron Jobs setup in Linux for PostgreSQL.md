**Cron Jobs setup in Linux** using your PostgreSQL backup example.
---
# Cron Jobs Basics in Linux

## 1. What is Cron?

**Cron** is a Linux scheduler used to run commands or scripts automatically at a specific time.

Example use cases:

| Use Case        | Example                              |
| --------------- | ------------------------------------ |
| Database backup | Run `pg_dump` daily at 2 AM          |
| Log cleanup     | Delete old backup files every Sunday |
| Monitoring      | Run health check every 6 hours       |
| Maintenance     | Vacuum/analyze database monthly      |

---

# 2. Cron Syntax Format

A cron entry has **5 schedule fields** followed by the command.

```bash
* * * * * command_to_run
```

| Field Position | Meaning      | Allowed Values |
| -------------- | ------------ | -------------- |
| 1              | Minute       | 0–59           |
| 2              | Hour         | 0–23           |
| 3              | Day of Month | 1–31           |
| 4              | Month        | 1–12           |
| 5              | Day of Week  | 0–7            |

Important note:

```bash
0 = Sunday
7 = Sunday
1 = Monday
2 = Tuesday
```

---

# 3. Basic Cron Examples

## Run every day at 2 AM

```bash
0 2 * * * /path/to/script.sh
```

Meaning:

| Field        | Value | Meaning       |
| ------------ | ----: | ------------- |
| Minute       |     0 | At minute 0   |
| Hour         |     2 | At 2 AM       |
| Day of Month |     * | Every day     |
| Month        |     * | Every month   |
| Day of Week  |     * | Every weekday |

---

## Run every 6 hours

```bash
0 */6 * * * /path/to/script.sh
```

This runs at:

```text
12 AM
6 AM
12 PM
6 PM
```

---

## Run every Sunday at 1 AM

```bash
0 1 * * 0 /path/to/script.sh
```

Meaning:

| Field        | Value | Meaning     |
| ------------ | ----: | ----------- |
| Minute       |     0 | At minute 0 |
| Hour         |     1 | At 1 AM     |
| Day of Month |     * | Any day     |
| Month        |     * | Any month   |
| Day of Week  |     0 | Sunday only |

---

## Run first day of every month at 3 AM

```bash
0 3 1 * * /path/to/script.sh
```

Meaning:

| Field        | Value | Meaning     |
| ------------ | ----: | ----------- |
| Minute       |     0 | At minute 0 |
| Hour         |     3 | At 3 AM     |
| Day of Month |     1 | First day   |
| Month        |     * | Every month |
| Day of Week  |     * | Any weekday |

---

# 4. PostgreSQL User Cron

For PostgreSQL backups, it is better to schedule the cron under the **postgres OS user**.

## Verify cron entries for postgres user

```bash
sudo crontab -u postgres -l
```

If no cron exists, you may see:

```text
no crontab for postgres
```

That is normal.

---

## Edit cron for postgres user

```bash
sudo crontab -u postgres -e
```

This opens the cron file for the postgres user.

Add your schedule at the bottom.

Example:

```bash
0 2 * * * /pgbackup/scripts/full_backup.sh
```

Save and exit.

---

# 5. Important Cron Commands

| Requirement               | Command                       |
| ------------------------- | ----------------------------- |
| View current user cron    | `crontab -l`                  |
| Edit current user cron    | `crontab -e`                  |
| View postgres user cron   | `sudo crontab -u postgres -l` |
| Edit postgres user cron   | `sudo crontab -u postgres -e` |
| Remove current user cron  | `crontab -r`                  |
| Remove postgres user cron | `sudo crontab -u postgres -r` |

Be careful with:

```bash
crontab -r
```

It removes all cron entries for that user.

---

# 6. PostgreSQL Backup Script Example

Create backup script:

```bash
sudo mkdir -p /pgbackup/scripts
sudo nano /pgbackup/scripts/full_backup.sh
```

Add this script:

```bash
#!/bin/bash

BACKUP_DIR="/pgbackup/logical"
DATE=$(date +%F_%H%M%S)
DB_NAME="postgres"

mkdir -p "$BACKUP_DIR"

pg_dump -U postgres -d "$DB_NAME" -F c -f "$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

find "$BACKUP_DIR" -type f -name "*.dump" -mtime +7 -delete
```

Save and exit.

Give ownership to postgres:

```bash
sudo chown -R postgres:postgres /pgbackup/scripts
sudo chown -R postgres:postgres /pgbackup/logical
```

Give execute permission:

```bash
sudo chmod +x /pgbackup/scripts/full_backup.sh
```

Test manually first:

```bash
sudo -u postgres /pgbackup/scripts/full_backup.sh
```

Check backup file:

```bash
ls -lh /pgbackup/logical
```

---

# 7. Add Backup Script to Cron

Edit postgres cron:

```bash
sudo crontab -u postgres -e
```

Add this line:

```bash
0 2 * * * /pgbackup/scripts/full_backup.sh
```

Verify:

```bash
sudo crontab -u postgres -l
```

Expected output:

```bash
0 2 * * * /pgbackup/scripts/full_backup.sh
```

---

# 8. Recommended Cron Entry with Log File

Better version:

```bash
0 2 * * * /pgbackup/scripts/full_backup.sh >> /pgbackup/logical/full_backup.log 2>&1
```

Meaning:

| Part                                | Meaning                               |
| ----------------------------------- | ------------------------------------- |
| `>>`                                | Append output to log file             |
| `/pgbackup/logical/full_backup.log` | Log file path                         |
| `2>&1`                              | Capture both normal output and errors |

---

# 9. Common Cron Schedules Table

| Requirement                      | Cron Schedule      |
| -------------------------------- | ------------------ |
| Every minute                     | `* * * * *`        |
| Every 5 minutes                  | `*/5 * * * *`      |
| Every 15 minutes                 | `*/15 * * * *`     |
| Every hour                       | `0 * * * *`        |
| Every 6 hours                    | `0 */6 * * *`      |
| Every day at 2 AM                | `0 2 * * *`        |
| Every day at 11:30 PM            | `30 23 * * *`      |
| Every Sunday at 1 AM             | `0 1 * * 0`        |
| Every Monday at 4 AM             | `0 4 * * 1`        |
| First day of every month at 3 AM | `0 3 1 * *`        |
| Last day of month                | Needs script logic |
| Every January 1st at midnight    | `0 0 1 1 *`        |

---

# 10. Cron Special Keywords

Cron also supports special shortcuts.

| Shortcut   | Meaning                |
| ---------- | ---------------------- |
| `@reboot`  | Run when server starts |
| `@hourly`  | Run once every hour    |
| `@daily`   | Run once every day     |
| `@weekly`  | Run once every week    |
| `@monthly` | Run once every month   |
| `@yearly`  | Run once every year    |

Example:

```bash
@daily /pgbackup/scripts/full_backup.sh
```

Example run after server reboot:

```bash
@reboot /pgbackup/scripts/startup_check.sh
```

---

# 11. Cron Environment Issue

Cron does not load the full user environment like your normal terminal.

So commands like this may fail:

```bash
pg_dump
```

Better to use full path:

```bash
/usr/bin/pg_dump
```

Find path:

```bash
which pg_dump
```

Example output:

```bash
/usr/bin/pg_dump
```

Then update script:

```bash
/usr/bin/pg_dump -U postgres -d postgres -F c -f "$BACKUP_FILE"
```

---

# 12. Best Practice Cron Format for PostgreSQL Backups

Recommended cron entry:

```bash
0 2 * * * /bin/bash /pgbackup/scripts/full_backup.sh >> /pgbackup/logical/full_backup.log 2>&1
```

This is better because:

| Benefit               | Explanation                    |
| --------------------- | ------------------------------ |
| Uses `/bin/bash`      | Ensures script runs using bash |
| Uses full script path | Avoids path issue              |
| Logs output           | Helps troubleshooting          |
| Runs as postgres      | Correct OS-level permission    |

---

# 13. Check Cron Service Status

Cron service must be running.

On Ubuntu/Debian:

```bash
sudo systemctl status cron
```

Start cron:

```bash
sudo systemctl start cron
```

Enable cron on boot:

```bash
sudo systemctl enable cron
```

Restart cron:

```bash
sudo systemctl restart cron
```

---

# 14. Check Cron Logs

On Ubuntu:

```bash
grep CRON /var/log/syslog
```

Example:

```bash
sudo grep CRON /var/log/syslog
```

Check latest cron activity:

```bash
sudo grep CRON /var/log/syslog | tail -50
```

---

# 15. PostgreSQL Backup Cron Example — Final Setup

## Step 1: Create folders

```bash
sudo mkdir -p /pgbackup/scripts
sudo mkdir -p /pgbackup/logical
sudo chown -R postgres:postgres /pgbackup
```

## Step 2: Create backup script

```bash
sudo nano /pgbackup/scripts/full_backup.sh
```

Paste:

```bash
#!/bin/bash

BACKUP_DIR="/pgbackup/logical"
DATE=$(date +%F_%H%M%S)
DB_NAME="postgres"
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

mkdir -p "$BACKUP_DIR"

/usr/bin/pg_dump -U postgres -d "$DB_NAME" -F c -f "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "$(date): Backup completed successfully: $BACKUP_FILE"
else
    echo "$(date): Backup failed"
    exit 1
fi

find "$BACKUP_DIR" -type f -name "*.dump" -mtime +7 -delete
```

## Step 3: Give permission

```bash
sudo chown postgres:postgres /pgbackup/scripts/full_backup.sh
sudo chmod +x /pgbackup/scripts/full_backup.sh
```

## Step 4: Test manually

```bash
sudo -u postgres /pgbackup/scripts/full_backup.sh
```

## Step 5: Add cron

```bash
sudo crontab -u postgres -e
```

Add:

```bash
0 2 * * * /bin/bash /pgbackup/scripts/full_backup.sh >> /pgbackup/logical/full_backup.log 2>&1
```

## Step 6: Verify cron

```bash
sudo crontab -u postgres -l
```

Expected:

```bash
0 2 * * * /bin/bash /pgbackup/scripts/full_backup.sh >> /pgbackup/logical/full_backup.log 2>&1
```

---

# 16. Troubleshooting Cron Jobs

| Issue                                 | Reason                | Fix                                               |
| ------------------------------------- | --------------------- | ------------------------------------------------- |
| Script works manually but not in cron | PATH issue            | Use full command paths                            |
| Backup file not created               | Permission issue      | Check ownership of backup folder                  |
| Cron not running                      | Cron service stopped  | Start cron service                                |
| No logs                               | Output not redirected | Add `>> logfile 2>&1`                             |
| pg_dump asks password                 | Authentication issue  | Configure `.pgpass` or local trust/peer carefully |
| Wrong timing                          | Server timezone issue | Check `timedatectl`                               |

---

# 17. Check Server Timezone

```bash
timedatectl
```

Example:

```text
Time zone: America/Chicago
```

Change timezone if needed:

```bash
sudo timedatectl set-timezone America/Chicago
```

---

# 18. Simple Cron Practice Examples

## Run script every minute for testing

```bash
* * * * * /bin/bash /pgbackup/scripts/test.sh >> /pgbackup/scripts/test.log 2>&1
```

Test script:

```bash
echo "$(date): Cron test successful" >> /pgbackup/scripts/test_output.log
```

After testing, remove the every-minute cron job.

---

# Final Recommended Cron for Your PostgreSQL Backup

```bash
sudo crontab -u postgres -e
```

Add:

```bash
0 2 * * * /bin/bash /pgbackup/scripts/full_backup.sh >> /pgbackup/logical/full_backup.log 2>&1
```

Verify:

```bash
sudo crontab -u postgres -l
```

Check logs:

```bash
tail -100 /pgbackup/logical/full_backup.log
```

Check backup files:

```bash
ls -lh /pgbackup/logical
```
