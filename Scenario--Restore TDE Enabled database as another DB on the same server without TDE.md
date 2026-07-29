# AdvworksDB Database Backup and Restore SOP

## Overview

This procedure backs up the AdvworksDB database from the QE server with Transparent Data Encryption (TDE) enabled and restores it on the same server with a new name. 
The restored database must not conflict with TDE settings.

Supported SQL Server versions: 2019, 2020, 2021, 2022, 2023, 2024, 2025.

## Prerequisites

- Access to QE server with SQL Server 2019 or higher
- Sufficient disk space for backup file (approximately 1.5 times the database size)
- SQL Server Management Studio (SSMS) or sqlcmd utility installed
- Backup location accessible by SQL Server service account
- Administrative privileges on the QE server

## Step 1: Prepare the Backup Location

1. Open File Explorer or connect to the QE server via RDP
2. Navigate to your backup directory (e.g., C:\Backups or \\backup_share\backups)
3. Verify the SQL Server service account has read and write permissions on this location
4. Confirm at least 200 GB of free space (adjust based on your database size)

## Step 2: Verify TDE Status on Source Database

1. Open SQL Server Management Studio and connect to the QE server
2. Expand Databases folder in Object Explorer
3. Right-click the AdvworksDB database and select Properties
4. Navigate to Options page and verify Encryption status shows "Enabled" or "On"
5. Note the database compatibility level (should be 130 or higher for SQL Server 2019+)

## Step 3: Create Full Database Backup

1. In SSMS, open a New Query window
2. Run the following command, replacing paths as needed:

```
BACKUP DATABASE [AdvworksDB] 
TO DISK = 'C:\Backups\AdvworksDB_Backup.bak' 
WITH INIT, COMPRESSION, CHECKSUM;
```

3. Wait for the backup to complete (you will see "Backup of database 'AdvworksDB' completed successfully")
4. Record the backup completion time and file size
5. Verify the backup file exists in your backup directory

Alternative: Right-click AdvworksDB database, select Tasks, then Back Up to open the backup dialog.

## Step 4: Verify Backup Integrity

1. Open a New Query window in SSMS
2. Run the following command to validate the backup:

```
RESTORE VERIFYONLY 
FROM DISK = 'C:\Backups\AdvworksDB_Backup.bak';
```

3. Confirm the message shows "The backup set is valid"
4. If the backup fails verification, stop and create a new backup

## Step 5: Check TDE Certificate on Target

1. Run this query to list all certificates on the QE server:

```
SELECT * FROM sys.certificates 
WHERE pvt_key_encryption_type = 'CERTIFICATE' 
OR pvt_key_encryption_type = 'PASSWORD';
```

2. Note the certificate name used by the AdvworksDB database
3. Verify the certificate exists (if not present, restore it from backup before proceeding)

## Step 6: Restore Database with New Name

1. Open a New Query window in SSMS connected to the QE server
2. First, get the logical file names from the backup:

```
RESTORE FILELISTONLY 
FROM DISK = 'C:\Backups\AdvworksDB_Backup.bak';
```

3. Note the LogicalName values for the data and log files
4. Run the restore command (replace file paths and logical names accordingly):

```
RESTORE DATABASE [AdvworksDB_New] 
FROM DISK = 'C:\Backups\AdvworksDB_Backup.bak' 
WITH MOVE 'AdvworksDB' TO 'C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA\AdvworksDB_New.mdf',
MOVE 'AdvworksDB_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA\AdvworksDB_New_log.ldf',
REPLACE, RECOVERY;
```

Note: Adjust logical file names and paths to match your environment. MSSQL15 = SQL Server 2019, MSSQL16 = SQL Server 2022, etc.

5. Wait for the restore to complete

## Step 7: Verify TDE on Restored Database

1. Run this query to check encryption status:

```
SELECT name, is_encrypted 
FROM sys.databases 
WHERE name = 'AdvworksDB_New';
```

2. If is_encrypted = 1, TDE is active on the restored database
3. Verify the certificate is correctly applied by running:

```
SELECT * FROM sys.certificates 
WHERE name IN (SELECT certificate_id FROM sys.dm_database_encryption_keys 
WHERE database_id = DB_ID('AdvworksDB_New'));
```

## Step 8: Disable TDE on AdvworksDB_New (If Required)

Only perform this step if the task specifies TDE must be disabled on the restored database.

1. Open a New Query window and run:

```
USE [AdvworksDB_New];
ALTER DATABASE [AdvworksDB_New] SET ENCRYPTION OFF;
```

2. Wait for encryption to complete (this may take several minutes for large databases)
3. Monitor progress with this query:

```
SELECT database_id, encryption_state 
FROM sys.dm_database_encryption_keys 
WHERE database_id = DB_ID('AdvworksDB_New');
```

Encryption_state values: 0 = No encryption, 1 = Unencrypted, 2 = Encryption in progress, 3 = Encrypted, 4 = Decryption in progress.

4. Once encryption_state = 1, run the verify query from Step 7 again to confirm is_encrypted = 0

## Step 9: Verify Restored Database Integrity

1. Right-click the AdvworksDB_New database in Object Explorer and select Properties
2. Navigate to Options and confirm database state is Online
3. Run a simple integrity check:

```
DBCC CHECKDB (AdvworksDB_New) WITH NO_INFOMSGS;
```

4. If no errors appear, the database is healthy

Alternative: Right-click AdvworksDB_New, select Tasks, then Maintenance Plans to run full database checks.

## Step 10: Test Database Connectivity

1. Open a New Query window
2. Connect to the AdvworksDB_New database using the dropdown in SSMS
3. Run a basic query to confirm access:

```
USE [AdvworksDB_New];
SELECT TOP 1 * FROM sys.objects;
```

4. Verify you receive results without errors

## Step 11: Document and Archive

1. Record the backup and restore completion times in your change log
2. Note the backup file location and size
3. Archive this SOP execution with the date and time stamps
4. Notify the Database Administration / SQL DBA Team of successful completion
5. Update your backup inventory with AdvworksDB_New database details

## Rollback Procedure

If the restore fails or data is corrupted:

1. Delete the AdvworksDB_New database from SSMS (right-click, Delete)
2. Verify the backup file is still intact
3. Contact the SQL DBA Team and repeat from Step 6 with an alternate backup or original AdvworksDB backup

## Troubleshooting

Issue: "The backup set is not valid"
Solution: Verify the backup file is not corrupted. Try creating a fresh backup from Step 3.

Issue: "Certificate not found" during restore
Solution: The TDE certificate must exist on the target server. Restore the certificate from the production database before restoring the backup.

Issue: "The file is in use" error during restore
Solution: Ensure no users are connected to AdvworksDB_New and restart the SQL Server service if needed.

Issue: Encryption state remains 2 (in progress) after extended time
Solution: Check SQL Server error log for issues. Monitor disk space and ensure no queries are running on the database.

## Notes

- This procedure uses COMPRESSION to reduce backup file size and improve performance
- CHECKSUM validates backup integrity during creation
- RECOVERY brings the database online immediately after restore
- TDE-encrypted backups can only be restored on servers with the correct certificate installed
- Do not rename AdvworksDB_New to AdvworksDB while the original AdvworksDB database still exists on the same server
- Schedule backups during low-usage periods to minimize performance impact

## Support

For issues or questions, contact the Database Administration / SQL DBA Team with:
- Exact error message and error number
- Time of failure
- SQL Server error log entries (if available)
- Current backup file location
