# Standard Operating Procedure (SOP)

Reference:
https://www.dbi-services.com/blog/sql-server-2025-backups-to-secondary-replicas-of-an-availability-group/

Credits To: Stéphane Savorgnano

## SQL Server 2025 – Performing Native Backups on Secondary Replicas in Always On Availability Groups

**Technology:** SQL Server 2025 Enterprise Edition

**Applies To:** SQL Server 2025 (17.x) Always On Availability Groups

---

# 1. Purpose

This SOP describes the standard procedure for configuring and performing **Full, Differential, and Transaction Log backups on Secondary Replicas** in SQL Server 2025 Always On Availability Groups (AG).

SQL Server 2025 introduces a significant enhancement that allows **native Full and Differential backups** to be executed on secondary replicas in addition to Transaction Log backups. Previous SQL Server versions only supported **COPY_ONLY Full backups** and **Transaction Log backups** on secondary replicas. This enhancement reduces backup overhead on production primary replicas while improving overall resource utilization. ([TECHCOMMUNITY.MICROSOFT.COM][1])

---

# 2. Background

## Previous Behavior (SQL Server 2022 and Earlier)

Supported on Secondary Replica:

* COPY_ONLY Full Backup
* Transaction Log Backup

Not Supported:

* Regular Full Backup
* Differential Backup

As a result, DBAs were forced to:

* Execute Full backups on the Primary Replica
* Perform Differential backups only on the Primary
* Mix backup locations depending on backup type

---

## SQL Server 2025 Enhancement

SQL Server 2025 now supports:

| Backup Type            | Primary | Secondary |
| ---------------------- | ------- | --------- |
| Full Backup            | ✅       | ✅         |
| Differential Backup    | ✅       | ✅         |
| Transaction Log Backup | ✅       | ✅         |
| COPY_ONLY Full         | ✅       | ✅         |

This enables organizations to move nearly all backup workloads away from the production primary replica. ([TECHCOMMUNITY.MICROSOFT.COM][1])

---

# 3. Benefits

Implementing backups on secondary replicas provides:

* Reduced CPU utilization on the Primary Replica
* Lower disk I/O on production workloads
* Better OLTP performance
* Faster maintenance windows
* Better utilization of HA infrastructure
* Simplified backup architecture
* Increased availability during backup operations
* Reduced contention between user workloads and backup processes

---

# 4. Prerequisites

Before implementing:

### SQL Server Version

* SQL Server 2025 Enterprise Edition

### Always On Availability Groups

Configured and operational.

### Replica State

Secondary replica must be:

* SYNCHRONIZED
  or
* SYNCHRONIZING

The secondary replica must maintain communication with the primary replica during backup operations. ([Microsoft Learn][2])

---

# 5. Backup Preference Configuration

Configure the Availability Group to direct backup jobs toward secondary replicas.

Example:

```sql
ALTER AVAILABILITY GROUP [FinanceAG]
SET (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY
);
```

Available options:

* PRIMARY
* SECONDARY_ONLY
* SECONDARY
* NONE

Recommended:

```
SECONDARY
```

---

# 6. Configure Replica Backup Priority

Assign backup priority.

Example:

```sql
ALTER AVAILABILITY GROUP FinanceAG
MODIFY REPLICA ON
'SQLNODE02'
WITH (
    BACKUP_PRIORITY = 90
);
```

Higher priority replicas become preferred backup targets.

---

# 7. Verify Preferred Backup Replica

Run:

```sql
SELECT
    sys.fn_hadr_backup_is_preferred_replica('SalesDB') AS PreferredReplica;
```

Result:

```
1 = Preferred
0 = Not Preferred
```

Only execute backups when the function returns **1**.

---

# 8. Full Backup Procedure

Example:

```sql
BACKUP DATABASE SalesDB
TO DISK='D:\SQLBackups\SalesDB_FULL.bak'
WITH COMPRESSION,
CHECKSUM,
STATS = 10;
```

SQL Server 2025 now permits this command on a secondary replica without requiring `COPY_ONLY`. ([Microsoft Learn][2])

---

# 9. Differential Backup Procedure

Example:

```sql
BACKUP DATABASE SalesDB
TO DISK='D:\SQLBackups\SalesDB_DIFF.bak'
WITH DIFFERENTIAL,
COMPRESSION,
CHECKSUM,
STATS = 10;
```

This functionality is new in SQL Server 2025.

---

# 10. Transaction Log Backup

Example:

```sql
BACKUP LOG SalesDB
TO DISK='D:\SQLBackups\SalesDB_LOG.trn'
WITH COMPRESSION,
CHECKSUM,
STATS = 10;
```

Transaction log backups continue to maintain a consistent log chain regardless of whether they are taken on the primary or secondary replica. ([Microsoft Learn][2])

---

# 11. SQL Agent Job Logic

All AG replicas should contain identical backup jobs.

Example logic:

```sql
IF sys.fn_hadr_backup_is_preferred_replica('SalesDB') = 1
BEGIN

BACKUP DATABASE SalesDB
TO DISK='D:\SQLBackups\SalesDB_FULL.bak'
WITH COMPRESSION;

END
```

This prevents duplicate backups after failover.

---

# 12. Recommended Backup Schedule

| Backup          | Frequency        |
| --------------- | ---------------- |
| Full            | Weekly           |
| Differential    | Daily            |
| Transaction Log | Every 15 minutes |
| COPY_ONLY       | On Demand        |

---

# 13. Backup Verification

Always validate backups.

Example:

```sql
RESTORE VERIFYONLY
FROM DISK='D:\SQLBackups\SalesDB_FULL.bak';
```

Also periodically perform full restore testing on a non-production server.

---

# 14. Monitoring

Monitor backup history:

```sql
SELECT
database_name,
backup_start_date,
backup_finish_date,
type,
physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf
ON bs.media_set_id=bmf.media_set_id
ORDER BY backup_finish_date DESC;
```

Monitor replica synchronization:

```sql
SELECT
replica_server_name,
synchronization_state_desc
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar
ON drs.replica_id=ar.replica_id;
```

---

# 15. Best Practices

* Keep identical SQL Agent backup jobs on every replica.
* Always use `sys.fn_hadr_backup_is_preferred_replica()` before starting backups.
* Enable backup compression unless constrained by CPU.
* Use `CHECKSUM` to improve backup integrity.
* Validate backups with `RESTORE VERIFYONLY`.
* Perform regular test restores to ensure recoverability.
* Monitor synchronization health before running backups.
* Store backups on dedicated storage separate from database files.
* Review backup preferences after AG failover events.

---

# 16. Operational Considerations

* Secondary replicas must remain connected to the primary for backup coordination.
* Backups should only be performed on synchronized or synchronizing replicas.
* Restore operations cannot be executed directly against databases that are part of an Availability Group.
* Backup preferences guide where jobs should run but do not automatically move jobs; SQL Agent jobs must exist on all eligible replicas and evaluate the preferred replica function before executing. ([Microsoft Learn][2])

---

# 17. Rollback Procedure

If issues occur:

1. Change backup preference to **PRIMARY**.
2. Disable backup jobs on secondary replicas.
3. Enable backup jobs on the primary replica.
4. Verify successful backup completion.
5. Confirm backup history in `msdb`.

---

# 18. Troubleshooting

| Issue                   | Possible Cause              | Resolution                                                         |
| ----------------------- | --------------------------- | ------------------------------------------------------------------ |
| Backup job skipped      | Replica is not preferred    | Verify `sys.fn_hadr_backup_is_preferred_replica()`                 |
| Backup fails            | Replica not synchronized    | Check AG health and synchronization state                          |
| Full backup not running | Backup preference incorrect | Review `AUTOMATED_BACKUP_PREFERENCE` and replica `BACKUP_PRIORITY` |
| Missing backup history  | SQL Agent job disabled      | Ensure jobs are deployed and enabled on all replicas               |

---

# 19. Key Takeaways

SQL Server 2025 removes one of the longstanding limitations of Always On Availability Groups by allowing **regular Full and Differential backups on secondary replicas**, in addition to Transaction Log and COPY_ONLY backups. This enables DBAs to offload nearly all routine backup activity from the primary production server, improving workload performance while maintaining high availability and a simplified backup strategy. Successful implementation depends on configuring `AUTOMATED_BACKUP_PREFERENCE`, assigning appropriate `BACKUP_PRIORITY` values, deploying backup jobs across all replicas, and using `sys.fn_hadr_backup_is_preferred_replica()` to ensure backups execute only on the designated preferred replica. ([TECHCOMMUNITY.MICROSOFT.COM][1])

[1]: https://techcommunity.microsoft.com/blog/azuresqlblog/introducing-backups-on-secondary-for-sql-server-always-on-availability-groups-wi/4422167?utm_source=chatgpt.com "Introducing \"Backups on Secondary\" for SQL Server Always On Availability Groups with SQL Server 2025 | Microsoft Community Hub"
[2]: https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/active-secondaries-backup-on-secondary-replicas-always-on-availability-groups?view=sql-server-ver17&utm_source=chatgpt.com "Offload Backups to Secondary Availability Group Replica - SQL Server Always On | Microsoft Learn"

Reference: https://www.dbi-services.com/blog/sql-server-2025-backups-to-secondary-replicas-of-an-availability-group/
