# Technical SOP: PostgreSQL Remote Connection Setup on Ubuntu Server

## 1. Purpose

This SOP explains how to configure PostgreSQL on Ubuntu to allow remote client connections from another machine in the same network.

In this lab:

| Item                   | Value                            |
| ---------------------- | -------------------------------- |
| PostgreSQL Server Name | UbuntuPGSQL / UbuntuRemoteServer |
| PostgreSQL Server IP   | `192.168.0.143`                  |
| Client Machine IP      | `192.168.0.216`                  |
| PostgreSQL Port        | `5432`                           |
| PostgreSQL User        | `postgres`                       |
| Database               | `postgres`                       |
| PostgreSQL Version     | PostgreSQL 18                    |

---

# 2. Issue Observed

PostgreSQL service is running successfully:

```bash
sudo systemctl status postgresql
```

Output shows:

```text
Active: active (exited)
```

PostgreSQL is also listening on port `5432`:

```bash
sudo ss -lntp | grep 5432
```

Output:

```text
LISTEN 0 200 0.0.0.0:5432
LISTEN 0 200 [::]:5432
```

This confirms PostgreSQL is listening on all IPv4 and IPv6 interfaces.

However, remote connection failed with this error:

```text
FATAL: no pg_hba.conf entry for host "192.168.0.216", user "postgres", database "postgres", SSL encryption

FATAL: no pg_hba.conf entry for host "192.168.0.216", user "postgres", database "postgres", no encryption
```

---

# 3. Root Cause

PostgreSQL has two important configuration files:

| File              | Purpose                                                              |
| ----------------- | -------------------------------------------------------------------- |
| `postgresql.conf` | Controls server-level settings like listen IP, port, memory, logging |
| `pg_hba.conf`     | Controls which client IPs/users/databases can connect                |

In this case:

```text
listen_addresses = '*'
```

is already working because PostgreSQL is listening on:

```text
0.0.0.0:5432
```

The real problem is:

```text
pg_hba.conf does not allow client IP 192.168.0.216
```

Important point:

You should not add only the PostgreSQL server IP `192.168.0.143` in `pg_hba.conf`.

You must allow the **client IP**:

```text
192.168.0.216
```

or allow the full subnet:

```text
192.168.0.0/24
```

---

# 4. Step-by-Step Fix

## Step 1: Verify PostgreSQL Server IP

Run on PostgreSQL server:

```bash
hostname -I
```

Expected:

```text
192.168.0.143
```

---

## Step 2: Verify PostgreSQL Service Status

```bash
sudo systemctl status postgresql
```

Expected:

```text
Active: active
```

For PostgreSQL package-based installation on Ubuntu, seeing `active (exited)` for the parent `postgresql.service` can be normal.

To verify actual PostgreSQL backend process, run:

```bash
ps -ef | grep postgres
```

---

## Step 3: Verify PostgreSQL Is Listening on Port 5432

```bash
sudo ss -lntp | grep 5432
```

Expected:

```text
LISTEN 0 200 0.0.0.0:5432
LISTEN 0 200 [::]:5432
```

Meaning:

| Output             | Meaning                                               |
| ------------------ | ----------------------------------------------------- |
| `0.0.0.0:5432`     | PostgreSQL accepts IPv4 connections on all interfaces |
| `[::]:5432`        | PostgreSQL accepts IPv6 connections                   |
| `postgres` process | Port is owned by PostgreSQL                           |

---

## Step 4: Find Active PostgreSQL Configuration Files

Run:

```bash
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
sudo -u postgres psql -c "SHOW data_directory;"
```

Expected examples:

```text
/etc/postgresql/18/main/postgresql.conf
/etc/postgresql/18/main/pg_hba.conf
/var/lib/postgresql/18/main
```

---

## Step 5: Verify `listen_addresses`

Open PostgreSQL configuration file:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Search inside nano:

```text
CTRL + W
listen_addresses
```

Set:

```conf
listen_addresses = '*'
```

Also verify port:

```conf
port = 5432
```

Save and exit:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 5. Configure `pg_hba.conf`

## Option 1: Allow Only One Client IP

This is more secure.

Open `pg_hba.conf`:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Add this line:

```conf
host    all             all             192.168.0.216/32          scram-sha-256
```

Meaning:

| Field              | Value             | Meaning                        |
| ------------------ | ----------------- | ------------------------------ |
| `host`             | TCP/IP connection | Allows remote TCP connection   |
| `all`              | Database          | Allows all databases           |
| `all`              | User              | Allows all PostgreSQL users    |
| `192.168.0.216/32` | Client IP         | Allows only this one client    |
| `scram-sha-256`    | Auth method       | Secure password authentication |

---

## Option 2: Allow Full LAN Subnet

For lab setup, this is easier.

```conf
host    all             all             192.168.0.0/24            scram-sha-256
```

Meaning this allows any machine from:

```text
192.168.0.1 to 192.168.0.254
```

Recommended for your current PostgreSQL lab:

```conf
host    all             all             192.168.0.0/24            scram-sha-256
```

Save and exit:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 6. Reload or Restart PostgreSQL

For `pg_hba.conf`, reload is enough:

```bash
sudo systemctl reload postgresql
```

For `postgresql.conf`, restart is required:

```bash
sudo systemctl restart postgresql
```

Recommended after both files are changed:

```bash
sudo systemctl restart postgresql
```

---

# 7. Validate `pg_hba.conf` Rules Loaded Correctly

Run:

```bash
sudo -u postgres psql -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules ORDER BY line_number;"
```

Check that your rule appears:

```text
host | all | all | 192.168.0.0/24 | scram-sha-256
```

Also check the `error` column. It should be empty.

---

# 8. Test Remote Connection

From client machine `192.168.0.216`, run:

```bash
psql -U postgres -d postgres -h 192.168.0.143 -p 5432
```

Correct syntax:

```bash
-U
```

Wrong syntax:

```bash
-u
```

Linux commands are case-sensitive.

Expected result:

```text
Password for user postgres:
```

After successful login:

```sql
SELECT version();
SELECT current_user;
SELECT current_database();
```

Exit psql:

```sql
\q
```

---

# 9. Reset PostgreSQL Password If Needed

If the connection reaches password prompt but password fails, reset password on the PostgreSQL server:

```bash
sudo -u postgres psql
```

Then run:

```sql
ALTER USER postgres WITH PASSWORD 'YourStrongPassword@123';
```

Exit:

```sql
\q
```

Then test again:

```bash
psql -U postgres -d postgres -h 192.168.0.143 -p 5432
```

---

# 10. Firewall Check

If `pg_hba.conf` is correct but connection still fails, check Ubuntu firewall.

```bash
sudo ufw status
```

If firewall is active, allow PostgreSQL port:

```bash
sudo ufw allow 5432/tcp
sudo ufw reload
```

Verify:

```bash
sudo ufw status numbered
```

---

# 11. Network Connectivity Test

From client machine, test PostgreSQL port.

## From Linux Client

```bash
nc -vz 192.168.0.143 5432
```

Or:

```bash
telnet 192.168.0.143 5432
```

## From Windows Client PowerShell

```powershell
Test-NetConnection 192.168.0.143 -Port 5432
```

Expected:

```text
TcpTestSucceeded : True
```

---

# 12. Common Errors and Fixes

## Error 1: Invalid option `-u`

Error:

```text
psql: invalid option -- 'u'
```

Cause:

```bash
psql -u postgres
```

is wrong.

Fix:

```bash
psql -U postgres
```

---

## Error 2: No `pg_hba.conf` entry

Error:

```text
FATAL: no pg_hba.conf entry for host "192.168.0.216"
```

Cause:

Client IP is not allowed in `pg_hba.conf`.

Fix:

```conf
host    all    all    192.168.0.216/32    scram-sha-256
```

or:

```conf
host    all    all    192.168.0.0/24      scram-sha-256
```

Then reload:

```bash
sudo systemctl reload postgresql
```

---

## Error 3: Connection refused

Error:

```text
connection refused
```

Possible causes:

| Cause                                  | Fix                                 |
| -------------------------------------- | ----------------------------------- |
| PostgreSQL not running                 | `sudo systemctl restart postgresql` |
| PostgreSQL not listening on network IP | Set `listen_addresses = '*'`        |
| Wrong port                             | Verify `port = 5432`                |
| Firewall blocking                      | Allow `5432/tcp`                    |

---

## Error 4: Password authentication failed

Error:

```text
FATAL: password authentication failed for user "postgres"
```

Fix:

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres WITH PASSWORD 'YourStrongPassword@123';
```

---

# 13. Final Working Configuration

## PostgreSQL Server

```text
Server IP: 192.168.0.143
Port: 5432
```

## `postgresql.conf`

```conf
listen_addresses = '*'
port = 5432
```

## `pg_hba.conf`

For lab subnet:

```conf
host    all             all             192.168.0.0/24            scram-sha-256
```

Or for single client:

```conf
host    all             all             192.168.0.216/32          scram-sha-256
```

## Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

## Test Connection

```bash
psql -U postgres -d postgres -h 192.168.0.143 -p 5432
```

---

# 14. Clean Command Checklist

Run these on PostgreSQL server:

```bash
hostname -I
sudo systemctl status postgresql
sudo ss -lntp | grep 5432
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
sudo -u postgres psql -c "SHOW listen_addresses;"
sudo -u postgres psql -c "SHOW port;"
```

Edit config:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Reload/restart:

```bash
sudo systemctl reload postgresql
sudo systemctl restart postgresql
```

Validate HBA rules:

```bash
sudo -u postgres psql -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules ORDER BY line_number;"
```

Test from client:

```bash
psql -U postgres -d postgres -h 192.168.0.143 -p 5432
```

---

# 15. Final Notes

Your PostgreSQL service and port binding are correct.

The root issue was:

```text
Client IP 192.168.0.216 was not allowed in pg_hba.conf
```

The fix is:

```conf
host    all    all    192.168.0.216/32    scram-sha-256
```

or for lab:

```conf
host    all    all    192.168.0.0/24      scram-sha-256
```

Then restart PostgreSQL and connect using:

```bash
psql -U postgres -d postgres -h 192.168.0.143 -p 5432
```
