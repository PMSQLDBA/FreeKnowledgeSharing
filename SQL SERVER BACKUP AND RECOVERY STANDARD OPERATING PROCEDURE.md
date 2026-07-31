**SQL SERVER BACKUP AND RECOVERY STANDARD OPERATING PROCEDURE**

**OVERVIEW**

This SOP covers SQL Server backup and recovery operations. Your database has a recovery model. 
That model determines what your transaction log retains, which backup types are possible, and how much data you can lose in a disaster. 
Understanding recovery models and backup contents is the difference between a restore strategy and a folder of hopeful backup files.

**RECOVERY MODELS**

Every SQL Server database has a recovery model set at creation time. Three models exist, each controlling exactly when the transaction log can be truncated and reused.

**SIMPLE RECOVERY MODEL**

In SIMPLE recovery, the transaction log truncates at each checkpoint. Checkpoints fire automatically during regular operations, recycling log space as they go.

- Log behavior: Checkpoints truncate the log only past the oldest active transaction
- What you get: Minimal administration, low log file growth, no point-in-time restore capability
- When to use: Development databases and non-critical reporting databases where losing all changes since the last full or differential backup is acceptable
- What you cannot do: Log backups are not allowed, point-in-time restore is not possible
- Log truncation trigger: Automatic checkpoint

**FULL RECOVERY MODEL**

In FULL recovery, the transaction log truncates only during a log backup. The database retains every transaction log record until a log backup carries it to safety.

- Pseudo-simple state: A database in FULL recovery without a full backup taken yet behaves exactly like SIMPLE with no log chain until that first full backup establishes the base
- Real FULL recovery: After the first full backup, the database starts holding log records and log_reuse_wait_desc shows "LOG_BACKUP", held log is the raw material for point-in-time restore
- Log behavior: Checkpoints do not truncate log space in FULL recovery after a base backup, only log backups release space for reuse
- What you get: Point-in-time restore capability, granular recovery control, ability to replay every transaction
- When to use: Production databases where losing changes is unacceptable, run FULL recovery with regular log backups tighter than your data-loss tolerance
- What you cannot do: You must take log backups on a regular schedule, without them the log grows until the disk fills or the database silently reverts to pseudo-simple behavior
- Log truncation trigger: Log backup only

**BULK_LOGGED RECOVERY MODEL**

BULK_LOGGED minimally logs certain bulk operations like SELECT INTO, BULK INSERT, and index builds. During these operations, the log writes extent references instead of individual row changes.

- Log savings during bulk operations: BULK_LOGGED writes much less log than FULL during large inserts, a million-row SELECT INTO uses 10 MB of log versus 453 MB under FULL
- The trade-off: Log backup after bulk operations must read actual extents from the data file, backup size ends up nearly as large as under FULL, you save log file space during the operation, not backup size
- Point-in-time limitations: You cannot restore to a point in time within the interval containing the bulk operation, for any restore touching a backup with bulk operations you must restore the entire backup
- When to use: BULK_LOGGED is a temporary state, switch into it for scheduled large loads, take log backups immediately before and after the bulk operation, switch back to FULL
- What to avoid: Never live in BULK_LOGGED permanently, every log backup carries the point-in-time asterisk
- Log truncation trigger: Log backup only

**RECOVERY MODEL DECISION FRAMEWORK**

Ask yourself three questions.

- First question: Can your business tolerate losing everything since the last full or differential backup? If yes, SIMPLE is honest and low-maintenance. If no, move to FULL with actual log backups on a schedule tighter than your tolerance for data loss.
- Second question: Do you have large bulk operations? If yes, plan to switch to BULK_LOGGED during those operations, take log backups before and after, do not stay in BULK_LOGGED continuously.
- Third question: If you run FULL recovery, do you actually have log backups running? FULL recovery without log backups is the worst of both worlds, the log grows until the disk fills or the database silently behaves like SIMPLE with no restore chain at all.

**BACKUP TYPES AND CONTENTS**

A backup file is not a copy of your MDF. It is a container with a specific contract. The container holds enough pages and enough log to reconstruct a transactionally consistent database as of the moment the backup finished.

**FULL BACKUP (TYPE D)**

- Content: Every allocated page in the database plus enough transaction log to make the restored copy consistent
- How it works: Data pages are read while transactions keep committing, the included log range reconciles it all at restore time
- Key LSN fields:
  - first_lsn: Where this backup starts in the log sequence
  - last_lsn: Where this backup ends
  - checkpoint_lsn: The checkpoint LSN where restore will begin
  - database_backup_lsn: The LSN of this backup for differentials to reference
- Restore time: A full backup alone is enough to restore a database to a consistent state as of the backup's end time

**DIFFERENTIAL BACKUP (TYPE I)**

- Content: Every extent modified since the last non-copy-only full backup, tracked by differential bitmap pages
- How it works: SQL Server maintains a bitmap of extents, each differential backup reads that bitmap and captures only changed extents
- Cumulative nature: Each differential contains everything since the base full, not just incremental changes from the previous differential, you restore the newest differential only, not all of them
- Key LSN field:
  - differential_base_lsn: Points to the first_lsn of the full backup this is based on
- Critical requirement: The full backup you plan to restore onto must match the differential_base_lsn, restore a differential onto the wrong full and SQL Server refuses
- Restore time: Restore the full backup, then the newest differential

**LOG BACKUP (TYPE L)**

- Content: The log records generated since the previous log backup
- Incremental nature: Log backups are the only incremental backup SQL Server has, each one is independent
- Chaining: Each log backup's first_lsn equals the previous log backup's last_lsn, gaps between this link means the chain is broken and you cannot restore through that gap
- Point-in-time restore: Only log backups let you stop between backup boundaries, you can restore to any LSN within any log backup's range
- Critical fact: Log backups are independent of full backups, the full backup does not truncate the log or restart the log chain, a log backup taken hours before a full backup is still valid after the full

**LSN CHAINING**

Every backup has a specific start LSN and end LSN. When you restore, you follow an LSN timeline.

The chain works like this:

- Full backup starts at LSN X, ends at LSN Y
- Differential backup's base_lsn points to full's first_lsn
- First log backup's first_lsn begins where the previous backup ended
- Each log backup's first_lsn equals the prior log backup's last_lsn

A gap in this chain breaks it. If a log backup ends at LSN 100 and the next one starts at LSN 105, you have a gap. You cannot restore through that gap.

**BACKUP HISTORY IN MSDB**

Every backup records its LSNs in msdb.dbo.backupset.

Query the chain:

```
SELECT
  bs.backup_set_id,
  bs.type,
  bs.first_lsn,
  bs.last_lsn,
  bs.database_backup_lsn,
  bs.differential_base_lsn
FROM msdb.dbo.backupset AS bs
WHERE bs.database_name = 'your_database_name'
ORDER BY bs.backup_set_id;
```

Interpretation:

- type D is full
- type I is differential
- type L is log
- differential_base_lsn tells you which full it depends on
- first_lsn and last_lsn of each backup tell you if the chain is intact

**READING BACKUP FILES WITHOUT MSDB**

If you have a backup file but no msdb, the file still tells its story.

Use RESTORE HEADERONLY to read the file header:

```
RESTORE HEADERONLY FROM DISK = 'C:\path\to\backup.bak';
```

The output includes:

- BackupType: D, I, or L
- RecoveryModel: What model the source database used
- HasBulkLoggedData: Whether this backup has minimal logging
- IsCopyOnly: Whether this is a copy-only backup
- BeginsLogChain: Whether this backup starts a new log chain
- HasBackupChecksums: Whether the backup has checksums

Additional restoration commands:

- RESTORE FILELISTONLY: Shows what files are inside, for building MOVE clauses when restoring to a new location
- RESTORE LABELONLY: Shows media set information

**CHOOSING BACKUPS FOR YOUR STRATEGY**

- Start with recovery model choice: If SIMPLE is acceptable, you have fewer options, if FULL is required, commit to log backups on a schedule you can maintain
- Design for your tolerance for data loss: If losing an hour of data is acceptable, take log backups every 15 minutes, if losing a minute is not acceptable, take them every minute, if losing nothing is not acceptable, you need both log backups and a HA solution
- Test your restore path: Full backup plus differential plus log backups should restore to a known good point, restore missing log backups means restoring to the end of the last available backup, plan for that scenario
- Verify the chain before a disaster forces a restore: Query msdb to confirm LSN continuity, restore a test database from your backup set monthly, do not wait for a production failure to learn your backups are broken

**COMMON MISTAKES**

- Full recovery without log backups: This creates pseudo-simple behavior without the benefits of actual SIMPLE recovery, the log grows silently and you have no restore chain
- Living in BULK_LOGGED instead of using it as a temporary state: Every log backup carries the point-in-time asterisk
- Assuming a full backup truncates the log in FULL recovery: The log does not truncate until a log backup runs
- Restoring a differential from backup set A onto a full backup from backup set B because they have similar database names: Restore fails because the differential_base_lsn does not match, always verify LSNs match before a restore
- Not testing restores regularly: A backup that has never been restored is a backup you cannot trust

**MONITORING AND MAINTENANCE**

Check log_reuse_wait_desc regularly on databases in FULL recovery:

```
SELECT
  name,
  log_reuse_wait_desc
FROM sys.databases
WHERE name = 'your_database_name';
```

- If it says anything other than "LOG_BACKUP", investigate: REPLICATION, MIRRORING, or NOTHING indicate log backups are not running or not working
- Monitor backup job success: Backup failures should trigger alerts
- Track log file growth: Unexplained log growth often signals that log backups are not running or not truncating the log
- Verify differential bitmap after index maintenance or large loads:

```
SELECT
  CONVERT(decimal(5,2), (ps.in_row_used_page_count + ps.lob_used_page_count + ps.row_overflow_used_page_count) / 128.0) AS size_mb
FROM sys.dm_db_partition_stats AS ps
WHERE database_id = DB_ID() AND index_id IN (0,1)
ORDER BY ps.in_row_used_page_count DESC;
```

Large extents modified recently will show up in the next differential backup.

RESTORE SEQUENCE OVERVIEW

Restore from full only:

RESTORE DATABASE your_db FROM DISK = 'full.bak' WITH RECOVERY;

Restore from full plus newest differential:

RESTORE DATABASE your_db FROM DISK = 'full.bak' WITH NORECOVERY;
RESTORE DATABASE your_db FROM DISK = 'diff.bak' WITH RECOVERY;

Restore from full, differential, and log backups:

RESTORE DATABASE your_db FROM DISK = 'full.bak' WITH NORECOVERY;
RESTORE DATABASE your_db FROM DISK = 'diff.bak' WITH NORECOVERY;
RESTORE LOG your_db FROM DISK = 'log1.trn' WITH NORECOVERY;
RESTORE LOG your_db FROM DISK = 'log2.trn' WITH RECOVERY;

Use NORECOVERY on all backups except the final one. The final restore uses WITH RECOVERY to bring the database online.

For point-in-time restore, add the STOPAT clause to the final log restore:

RESTORE LOG your_db FROM DISK = 'log2.trn' WITH RECOVERY, STOPAT = '2024-01-15 14:30:00';

SQL Server applies all log records up to that point and stops.

VERSION REQUIREMENTS

Testing was done on SQL Server 2019 CU32 and SQL Server 2025 RTM. 
This SOP applies to those versions and intermediate versions. 
Consult Microsoft Learn for version-specific behaviors.

Microsoft Learn: Differential Backups (SQL Server)
Microsoft Learn: The Transaction Log (SQL Server)
Microsoft Learn: RESTORE statements (Transact-SQL)
