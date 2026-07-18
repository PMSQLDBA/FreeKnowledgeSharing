Production-oriented **end-to-end T-SQL automation** for enabling **Transparent Data Encryption (TDE)** on a **SQL Server 2025 Standalone Instance** for database **AdventureWorks**.

This script includes:

* ✅ Pre-checks
* ✅ Verify prerequisites
* ✅ Take COPY_ONLY Full Backup
* ✅ Create Master Key (if not exists)
* ✅ Create Server Certificate (if not exists)
* ✅ Backup Certificate & Private Key
* ✅ Enable TDE
* ✅ Monitor Encryption Progress
* ✅ Validation
* ✅ Rollback Plan
* ✅ Post Implementation Checklist

---

# Step 1 - Variables

```sql
USE master;
GO

DECLARE @DatabaseName SYSNAME = 'AdventureWorks';

-- Backup Locations
DECLARE @FullBackupPath NVARCHAR(4000) =
'N:\SQLBackups\AdventureWorks_CopyOnly_PreTDE.bak';

DECLARE @CertificateBackupPath NVARCHAR(4000) =
'N:\SQLBackups\TDECert_Backup.cer';

DECLARE @PrivateKeyBackupPath NVARCHAR(4000) =
'N:\SQLBackups\TDECert_PrivateKey.pvk';

DECLARE @PrivateKeyPassword NVARCHAR(128) =
'StrongPassword@123456';

DECLARE @CertificateName SYSNAME =
'TDE_AdventureWorks_Certificate';
```

---

# Step 2 - Verify Database

```sql
SELECT
name,
state_desc,
recovery_model_desc,
is_encrypted
FROM sys.databases
WHERE name=@DatabaseName;
```

Expected:

```
ONLINE
FULL (recommended)
is_encrypted = 0
```

---

# Step 3 - Check Existing Database Encryption Keys

```sql
SELECT *
FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID(@DatabaseName);
```

Expected:

No rows.

---

# Step 4 - Take COPY_ONLY Full Backup

```sql
BACKUP DATABASE AdventureWorks
TO DISK='N:\SQLBackups\AdventureWorks_CopyOnly_PreTDE.bak'
WITH
COPY_ONLY,
CHECKSUM,
COMPRESSION,
INIT,
STATS=5;
GO
```

Verify backup:

```sql
RESTORE VERIFYONLY
FROM DISK='N:\SQLBackups\AdventureWorks_CopyOnly_PreTDE.bak';
GO
```

---

# Step 5 - Create Master Key (Only if Missing)

Check

```sql
SELECT *
FROM sys.symmetric_keys
WHERE symmetric_key_id=101;
```

If no rows:

```sql
CREATE MASTER KEY
ENCRYPTION BY PASSWORD='MasterKeyPassword@123456!';
GO
```

---

# Step 6 - Create TDE Certificate

Check:

```sql
SELECT *
FROM sys.certificates
WHERE name='TDE_AdventureWorks_Certificate';
```

If missing:

```sql
CREATE CERTIFICATE TDE_AdventureWorks_Certificate
WITH SUBJECT='AdventureWorks TDE Certificate',
EXPIRY_DATE='2036-12-31';
GO
```

---

# Step 7 - Backup Certificate Immediately

**Critical Step**

```sql
BACKUP CERTIFICATE TDE_AdventureWorks_Certificate

TO FILE='N:\SQLBackups\TDECert_Backup.cer'

WITH PRIVATE KEY
(
FILE='N:\SQLBackups\TDECert_PrivateKey.pvk',

ENCRYPTION BY PASSWORD='StrongPassword@123456'
);
GO
```

Store securely.

Without this certificate you **cannot restore** encrypted backups.

---

# Step 8 - Create Database Encryption Key

```sql
USE AdventureWorks;
GO

CREATE DATABASE ENCRYPTION KEY

WITH ALGORITHM = AES_256

ENCRYPTION BY SERVER CERTIFICATE
TDE_AdventureWorks_Certificate;
GO
```

---

# Step 9 - Enable Encryption

```sql
ALTER DATABASE AdventureWorks
SET ENCRYPTION ON;
GO
```

---

# Step 10 - Monitor Encryption Progress

```sql
SELECT

db_name(database_id) DatabaseName,

encryption_state,

percent_complete,

key_algorithm,

key_length

FROM sys.dm_database_encryption_keys;
GO
```

Encryption States

| State | Meaning                |
| ----- | ---------------------- |
| 0     | No Encryption          |
| 1     | Unencrypted            |
| 2     | Encryption in Progress |
| 3     | Encrypted              |
| 4     | Key Change             |
| 5     | Decryption in Progress |
| 6     | Protection Change      |

Wait until

```
State = 3
Percent = 0
```

---

# Step 11 - Validation

```sql
SELECT

name,

is_encrypted

FROM sys.databases

WHERE name='AdventureWorks';
```

Expected

```
is_encrypted = 1
```

---

Check encryption key

```sql
SELECT

db_name(database_id),

key_algorithm,

key_length,

encryptor_type,

encryption_state

FROM sys.dm_database_encryption_keys;
```

Expected

```
AES_256

Encrypted
```

---

# Step 12 - Take Post-TDE Backup

```sql
BACKUP DATABASE AdventureWorks

TO DISK='N:\SQLBackups\AdventureWorks_PostTDE_Full.bak'

WITH

CHECKSUM,

COMPRESSION,

INIT,

STATS=5;
GO
```

---

# Rollback Plan

## Option 1 (Immediately After Enabling)

Disable encryption

```sql
ALTER DATABASE AdventureWorks
SET ENCRYPTION OFF;
GO
```

Monitor

```sql
SELECT

db_name(database_id),

percent_complete,

encryption_state

FROM sys.dm_database_encryption_keys;
```

Wait until

```
encryption_state = 1
```

Drop Encryption Key

```sql
USE AdventureWorks;
GO

DROP DATABASE ENCRYPTION KEY;
GO
```

---

## Option 2 (Restore Pre-TDE Backup)

If rollback to original database state is required:

```sql
USE master;
GO

ALTER DATABASE AdventureWorks
SET SINGLE_USER
WITH ROLLBACK IMMEDIATE;
GO

RESTORE DATABASE AdventureWorks

FROM DISK='N:\SQLBackups\AdventureWorks_CopyOnly_PreTDE.bak'

WITH

REPLACE,

RECOVERY,

STATS=5;
GO

ALTER DATABASE AdventureWorks
SET MULTI_USER;
GO
```

---

## Option 3 (Certificate Cleanup)

Only if **no databases** use the certificate anymore:

```sql
DROP CERTIFICATE TDE_AdventureWorks_Certificate;
GO
```

If the master key was created solely for TDE and is no longer needed (rare in production):

```sql
DROP MASTER KEY;
GO
```

---

# Complete Flow

```
Health Check
      │
      ▼
COPY_ONLY Full Backup
      │
      ▼
Verify Backup
      │
      ▼
Create Master Key
      │
      ▼
Create Certificate
      │
      ▼
Backup Certificate
      │
      ▼
Create DEK
      │
      ▼
Enable Encryption
      │
      ▼
Monitor Progress
      │
      ▼
Validation
      │
      ▼
Post-TDE Full Backup
```

# Production Best Practices

* Always perform and verify a **COPY_ONLY** full backup before enabling TDE.
* **Immediately** back up the TDE certificate and private key, and store them in a secure, access-controlled location separate from the database backups.
* Take a **new full backup after TDE is enabled**. Although older (pre-TDE) backups remain restorable without the certificate, all backups taken after TDE is enabled require the certificate and private key for restore.
* Monitor the encryption scan via `sys.dm_database_encryption_keys`; large databases may take significant time to complete.
* Test restoring the encrypted backup on a non-production server by first restoring the certificate and private key.
* Ensure your disaster recovery documentation includes the certificate backup location, password escrow process, and restore procedures.
* Do not delete or rotate the certificate until you have confirmed that no encrypted databases or backups still depend on it.
