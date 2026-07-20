# SQL Server Transparent Data Encryption (TDE) SOP

**Version:** 1.0 | **Date:** July 2026 | **Author:** Praveen Madupu

---

## Table of Contents

1. Overview
2. Prerequisites
3. Why Database Master Key is NOT Visible in GUI
4. Pre-Implementation Backups
5. Enable TDE - Step by Step
6. Monitor & Verify
7. Rollback Procedure
8. Maintenance & Monitoring
9. Troubleshooting

---

## 1. Overview

Transparent Data Encryption (TDE) encrypts SQL Server database files at rest. 
This SOP provides step-by-step guidance for enabling TDE with proper backup procedures.

**Key Points:**
- Encrypts data at rest (disk level)
- Does NOT encrypt data in transit (use SSL/TLS separately)
- Requires SQL Server 2016 SP2+
- Background process - can take hours for large databases

---

## 2. Prerequisites

### System Requirements:
- SQL Server 2016 SP2 or later
- Database in FULL recovery mode
- 1.5x database size available for backups

### Permissions:
- CONTROL SERVER on instance
- CONTROL DATABASE on target database

---

## 3. Why Database Master Key is NOT Visible in GUI

### THE CRITICAL FACT:

**Database Master Key (##MS_DatabaseMasterKey##) is NOT a Symmetric Key - it is a SYSTEM OBJECT. 
SQL Server intentionally hides it from SSMS Object Explorer for security.**

### Encryption Hierarchy:

| Level | Key Name | Scope | Visibility in SSMS |
|-------|----------|-------|-------------------|
| 1 | Service Master Key | Instance-level | YES - Visible |
| 2 | Database Master Key | Database-level (System) | NO - Hidden |
| 3 | Database Encryption Key | Database-level | YES - Via DMV |

### Why Hidden?

System objects (prefix ##) are managed by SQL Server internally. 
They are hidden to prevent accidental modification or deletion. 
The DMK exists and works correctly - it's just not visible in the GUI.

### Verify DMK Exists (T-SQL):

```sql
SELECT * FROM sys.system_objects WHERE name LIKE '%DatabaseMasterKey%';
SELECT name, is_master_key_encrypted_by_server FROM sys.databases WHERE name = DB_NAME();
```

---

## 4. Pre-Implementation Backups

**CRITICAL: Take backups in this exact order BEFORE enabling TDE**

### Step 1: Backup Service Master Key

```sql
BACKUP SERVICE MASTER KEY 
TO FILE = 'C:\SQLBackups\SMK.key'
ENCRYPTION BY PASSWORD = 'ComplexPassword123!';
```

### Step 2: Backup Database Master Key

```sql
USE YourDatabase;
GO

BACKUP MASTER KEY 
TO FILE = 'C:\SQLBackups\DMK.key'
ENCRYPTION BY PASSWORD = 'ComplexPassword123!';
```

### Step 3: Create & Backup TDE Certificate

```sql
USE master;
GO

CREATE CERTIFICATE TDE_Certificate
WITH SUBJECT = 'TDE Certificate';

BACKUP CERTIFICATE TDE_Certificate
TO FILE = 'C:\SQLBackups\TDE_Certificate.cer'
WITH PRIVATE KEY (
    FILE = 'C:\SQLBackups\TDE_Certificate.pvk',
    ENCRYPTION BY PASSWORD = 'ComplexPassword123!'
);
```

### Step 4: Full Database Backup

```sql
ALTER DATABASE YourDatabase SET RECOVERY FULL;

BACKUP DATABASE YourDatabase
TO DISK = 'C:\SQLBackups\YourDatabase_PreTDE.bak'
WITH COMPRESSION;
```

---

## 5. Enable TDE - Step by Step

### Step 1: Verify Certificate

```sql
USE master;
SELECT name FROM sys.certificates WHERE name = 'TDE_Certificate';
```

### Step 2: Create Database Encryption Key

```sql
USE YourDatabase;
GO

CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Certificate;
```

### Step 3: Enable TDE

```sql
ALTER DATABASE YourDatabase SET ENCRYPTION ON;
```

### Step 4: Monitor Progress

```sql
SELECT 
    encryption_state,
    percent_complete
FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID('YourDatabase');

-- encryption_state = 3 means ENCRYPTED
-- Run repeatedly until percent_complete = 100
```

---

## 6. Monitor & Verify

### Check Encryption Status

```sql
SELECT 
    DB_NAME(database_id) AS Database_Name,
    encryption_state,
    CASE 
        WHEN encryption_state = 0 THEN 'No encryption key'
        WHEN encryption_state = 1 THEN 'Unencrypted'
        WHEN encryption_state = 2 THEN 'Encryption in progress'
        WHEN encryption_state = 3 THEN 'Encrypted'
    END AS Status
FROM sys.dm_database_encryption_keys;
```

### Test Backup Works

```sql
BACKUP DATABASE YourDatabase
TO DISK = 'C:\SQLBackups\YourDatabase_PostTDE.bak'
WITH COMPRESSION;
```

### Verify Certificate

```sql
USE master;
SELECT name, thumbprint, expiry_date
FROM sys.certificates
WHERE name = 'TDE_Certificate';
```

---

## 7. Rollback Procedure

**IF CRITICAL ISSUES OCCUR:**

### Step 1: Disable TDE

```sql
ALTER DATABASE YourDatabase SET ENCRYPTION OFF;
```

### Step 2: Wait for Decryption

Monitor with this query (may take hours):

```sql
SELECT 
    encryption_state,
    percent_complete
FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID('YourDatabase');

-- Wait until encryption_state = 1
```

### Step 3: Drop Encryption Key

```sql
DROP DATABASE ENCRYPTION KEY;
```

---

## 8. Maintenance & Monitoring

### Key Rotation (Every 12 Months)

```sql
ALTER DATABASE ENCRYPTION KEY
REGENERATE BY SERVER CERTIFICATE TDE_Certificate;
```

### Monitor Certificate Expiration

```sql
SELECT 
    name,
    expiry_date,
    DATEDIFF(day, GETDATE(), expiry_date) AS Days_Until_Expiry
FROM sys.certificates
WHERE name = 'TDE_Certificate';
```

---

## 9. Troubleshooting

| Issue | Solution |
|-------|----------|
| **CREATE DATABASE ENCRYPTION KEY fails** | Certificate not found. Verify: `SELECT * FROM sys.certificates;` |
| **Encryption scan at 99%** | Wait for active transactions. Check: `SELECT * FROM sys.dm_tran_active_transactions;` |
| **High CPU during scan** | Normal (3-5% overhead). Monitor tempdb - it auto-encrypts too. |
| **Backup fails after TDE** | Service account lacks permissions. Grant write access to backup folder. |

---

## Important Limitations (Microsoft Documentation)

### FILESTREAM NOT Encrypted
- FILESTREAM data is NOT encrypted by TDE
- Use BitLocker or EFS for FILESTREAM security

### tempdb Auto-Encryption
- tempdb is automatically encrypted when ANY user database is encrypted
- This affects all databases with ~3-5% CPU overhead

### Instant File Initialization
- Unavailable when TDE is enabled
- New database files will initialize slower (zero-filled, not instant)

### SQL Server 2019+ Advanced Features
- Encryption scan can be suspended and resumed
- Use: `ALTER DATABASE YourDatabase SET ENCRYPTION SUSPEND/RESUME;`

---

**END OF SOP**

Reference: 
https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption?view=sql-server-ver17
