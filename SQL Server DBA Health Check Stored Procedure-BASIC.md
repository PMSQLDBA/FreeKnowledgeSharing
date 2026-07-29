# SQL Server DBA Health Check Stored Procedure - BASIC

## Overview

`usp_DBA_HealthCheck` is a comprehensive SQL Server health monitoring stored procedure that performs 25 critical automated checks daily and delivers consolidated alerts via email. 
It provides detailed diagnostics across all critical database operations, performance metrics, and configuration compliance.

[SP Script] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_DBA_HealthCheck)

**Procedure Name:** `usp_DBA_HealthCheck`
**Drafted by:** Praveen Madupu
**Version:** 1.0
**Last Updated:** 28th July 2026 - Wednesday
**Compatibility:** SQL Server 2016 and later

---

## Quick Start

### Installation

1. Connect to your SQL Server in SSMS
2. Open `usp_DBA_HealthCheck.sql` [SP Script] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_DBA_HealthCheck)
3. Execute the script to create the procedure in the `master` database
4. Verify installation by running: `EXEC usp_DBA_HealthCheck;`

### Configuration

Edit these variables in the procedure (around line 20):

```sql
DECLARE @ProfileName NVARCHAR(128) = 'DBAAlerts';           -- Update Your mail profile
DECLARE @DBAEmailList NVARCHAR(MAX) = 'sqldbateam2026@gmail.com';  -- Update your Recipients list here
```

### Test Execution

```sql
USE master;
GO
EXEC usp_DBA_HealthCheck;
GO
```

---

## Features

### 25 Automated Health Checks

The procedure monitors:

**Critical Operations (5 checks)**
- SQL Server Agent job failures
- Database backup compliance
- SQL Server error log anomalies
- Database corruption detection (CHECKDB status)
- Disk space availability

**Performance & Resources (6 checks)**
- Automatic file growth events
- Active blocking and deadlocks
- Long-running queries
- CPU utilization
- Memory pressure and Page Life Expectancy
- Transaction log space usage

**Database Health & Availability (5 checks)**
- Suspect or offline databases
- Failed login attempts
- AlwaysOn Availability Group health
- Replication lag and pending commands
- TempDB usage analysis

**Diagnostics & Optimization (4 checks)**
- Wait statistics analysis
- Index fragmentation
- Missing index recommendations
- Query Store performance regression

**Configuration & Maintenance (4 checks)**
- Recent schema changes
- Configuration drift detection
- SQL Server service status
- Linked server connectivity

**Monitoring System (1 check)**
- Monitoring system health verification

### Consolidated Reporting

- **Main Issues Table:** All 25 checks with priority-based grouping
- **Backup Verification Details:** Dedicated table showing backup status per database
- **Database Corruption Details:** Dedicated table showing CHECKDB status per database
- **Real-time Console Output:** Progress tracking during execution
- **HTML Email Alerts:** Professional formatted reports sent only when issues found

### Priority Levels

- **CRITICAL:** Requires immediate action (red text in recommendation column)
- **HIGH:** Address within 4 hours
- **MEDIUM:** Address within business day
- **LOW:** Address during maintenance windows

---

## Configuration Details

### Database Mail Profile

Ensure Database Mail is configured:

```sql
-- Verify mail profile exists
SELECT * FROM msdb.dbo.sysmail_profile;

-- Create profile if needed (example)
EXECUTE msdb.dbo.sysmail_add_profile_sp
    @profile_name = 'DBAAlerts',
    @description = 'DBA Alerts Profile';
```

### Email Recipients

Multiple recipients supported (comma-separated):

```sql
@DBAEmailList = 'dba1@company.com,dba2@company.com,dba3@company.com'
```

### Scheduled Execution

Create SQL Agent jobs to run at 7 AM and 6 PM:

#### Job 1: DBA Health Check - 7 AM

```sql
EXEC msdb.dbo.sp_add_job
    @job_name = 'DBA_HealthCheck_7AM',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'DBA_HealthCheck_7AM',
    @step_name = 'Execute Health Check',
    @subsystem = 'TSQL',
    @command = 'EXEC usp_DBA_HealthCheck',
    @database_name = 'master';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Daily_7AM',
    @freq_type = 4,
    @freq_interval = 1,
    @active_start_time = 070000;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'DBA_HealthCheck_7AM',
    @schedule_name = 'Daily_7AM';
```

#### Job 2: DBA Health Check - 6 PM

```sql
EXEC msdb.dbo.sp_add_job
    @job_name = 'DBA_HealthCheck_6PM',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'DBA_HealthCheck_6PM',
    @step_name = 'Execute Health Check',
    @subsystem = 'TSQL',
    @command = 'EXEC usp_DBA_HealthCheck',
    @database_name = 'master';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Daily_6PM',
    @freq_type = 4,
    @freq_interval = 1,
    @active_start_time = 180000;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'DBA_HealthCheck_6PM',
    @schedule_name = 'Daily_6PM';
```

---

## Console Output Example

```
================================================================================
SQL SERVER DBA HEALTH CHECK - 2024-07-29 08:15:32.123
Server: SQLSERVER01
================================================================================

>>> SECTION 1: CRITICAL OPERATIONS
    Checking: Agent Jobs, Backups, Error Logs, Corruption, Disk Space
    Check 1: Agent Job Failures - Complete
    Check 2: Backup Verification - Complete
    Check 3: SQL Server Error Log - Complete
    Check 4: Database Corruption - Complete
    Check 5: Disk Space - Complete

>>> SECTION 2: PERFORMANCE & RESOURCE MANAGEMENT
    Checking: Auto-Growth, Blocking, Long Queries, CPU, Memory, Log Space
    Check 6: Auto-Growth Events - Complete
    Check 7: Blocking and Deadlocks - Complete
    ...

================================================================================
CONSOLIDATED REPORT SUMMARY
================================================================================
Total Issues Found: 5
  - CRITICAL: 2
  - HIGH:     2
  - MEDIUM:   1
  - LOW:      0

>>> CRITICAL ISSUES (Requires Immediate Action)
  [2] Backup Verification: Multiple databases require backup attention
  [5] Disk Space Monitoring: Drive F: Free Space 15% (10GB available)

>>> HIGH PRIORITY ISSUES (Address within 4 hours)
  [7] Blocking and Deadlocks: Session 45 blocking 3 other sessions
  [22] SQL Services Status: System database backup is older than 24 hours

>>> MEDIUM PRIORITY ISSUES (Address within business day)
  [10] Memory Pressure / PLE: Page Life Expectancy is 250 seconds

>>> BY SECTION SUMMARY
Section                         TotalIssues  Critical  High  Medium  Low
Critical Operations             2            2         0     0       0
Performance & Resources         2            0         2     0       0
Database Health                 1            0         0     1       0

>>> BACKUP VERIFICATION DETAILS
DatabaseName    FullBackup              LogBackup               RecoveryModel   Status                      Recommendation
OrdersDB        2024-07-20 18:30:00     2024-07-29 06:15:00     FULL            OK                          Backups are current. Test restore monthly.
ProductsDB      Never                   N/A                     SIMPLE          CRITICAL - Never backed up  Backup immediately. Database has never been backed up.

>>> DATABASE CORRUPTION CHECK DETAILS
DatabaseName    LastCheckDB         AgeDays     Status                  Recommendation
SalesDB         2024-07-15          14          WARNING - Stale         CHECKDB is 14 days old. Run CHECKDB within this week.
ReportsDB       Never               365         CRITICAL - Never run    CHECKDB has never been run. Schedule immediate execution.

================================================================================
Email notification sent successfully to: praveensqldba12@gmail.com
================================================================================
```

---

<img width="1676" height="752" alt="image" src="https://github.com/user-attachments/assets/ada1d0f7-f3ae-409b-b49f-eaf569d0bd97" />

<img width="1650" height="450" alt="image" src="https://github.com/user-attachments/assets/33e66dbd-ed22-4fca-8733-4bc152c58b4b" />


## Email Alert Format

Emails are sent only when CRITICAL or HIGH priority issues are detected.

### Email Subject Example
```
DBA Health Check Alert - 2 Critical, 2 High on SQLSERVER01 at 2024-07-29 08:15:32
```

### Email Body Sections

1. **Header:** Server name, execution timestamp
2. **Summary:** Count of issues by priority level
3. **Issues Table:** All HIGH and CRITICAL issues with:
   - Check number and name
   - Section/category
   - Priority level
   - Issue description
   - Job name (for job failures)
   - Job failure details (for job failures - in red)
   - Affected object
   - Recommended action (in red)
4. **Backup Verification Details Table:** Per-database backup status with recommendations (in red)
5. **Database Corruption Check Details Table:** Per-database CHECKDB status with recommendations (in red)

### Email Styling

- **Header:** Blue background (#2F5496) with white text
- **Table Headers:** Blue background (#4472C4) with white text
- **Data Rows:** White background
- **Recommendation Text:** Red color (#FF0000) for immediate visibility
- **Status Column:** Shows specific warnings (Critical, Overdue, Stale, etc.)

---

## The 25 Health Checks - Detailed

### CHECK 1: SQL Server Agent Job Failures

**What it monitors:** Failed, canceled, or stuck SQL Agent jobs in last 24 hours

**Columns in Report:**
- Job Name: Name of the failed job
- Job Failure Details: Specific error message from job history (red text)

**When it triggers:** Any job with status other than Success (1) in last 24 hours

**Recommendation:** Read error message, verify if transient, investigate recurring failures

### CHECK 2: Backup Verification

**What it monitors:** Full and transaction log backup recency

**Shows:** Last full backup date, last log backup date, recovery model, backup status, detailed recommendations

**When it triggers:**
- No full backup exists or full backup older than 1 day
- FULL recovery database with no log backups or log backups older than 4 hours

**Recommendations:** Kick off manual backups, establish automated backup schedule, test restore capability

### CHECK 3: SQL Server Error Log

**What it monitors:** Critical messages in SQL Server error log

**When it triggers:** More than 10 error entries in last 24 hours

**Look for:** Login failures, I/O errors, memory warnings, timeout issues

### CHECK 4: Database Corruption (CHECKDB)

**What it monitors:** DBCC CHECKDB status per database

**Shows:** Last CHECKDB run date, age in days, status warning, specific recommendation

**When it triggers:**
- Last CHECKDB older than 7 days
- CHECKDB has never been run

**Recommendations:** Run CHECKDB immediately for never-checked databases, weekly for stale checks

### CHECK 5: Disk Space

**What it monitors:** Available free space on all volumes with SQL Server files

**When it triggers:** Free space below 20%

**Recommendations:** Delete old backups, move files to larger drives, implement proactive alerting

### CHECK 6: Auto-Growth Events

**What it monitors:** Automatic file growth occurrences

**When it triggers:** More than 5 auto-growth events in 24 hours

**Impact:** Each growth event pauses queries while disk is zeroed

**Recommendations:** Pre-size files, use fixed growth values instead of percentage

### CHECK 7: Blocking and Deadlocks

**What it monitors:** Active session blocking chains

**When it triggers:** One session blocking multiple others

**Recommendations:** Identify head blocker, check execution plan, do not kill without investigation

### CHECK 8: Long-Running Queries

**What it monitors:** Queries running longer than 15 minutes

**When it triggers:** Any query exceeding 900 seconds runtime

**Recommendations:** Check for blocking, review execution plan, investigate after completion

### CHECK 9: CPU Utilization

**What it monitors:** SQL Server process CPU consumption

**When it triggers:** CPU sustained above 85%

**Recommendations:** Identify top CPU-consuming queries, assess scaling needs

### CHECK 10: Memory Pressure / PLE

**What it monitors:** Page Life Expectancy (buffer cache health)

**When it triggers:** PLE below 300 seconds

**Recommendations:** Find scan-heavy queries, add covering indexes, verify memory truly needed

### CHECK 11: Transaction Log Space

**What it monitors:** Log file usage in FULL recovery databases

**When it triggers:** Log space usage high

**Recommendations:** Take log backup immediately, implement regular log backup schedule

### CHECK 12: Suspect/Offline Databases

**What it monitors:** Database state

**When it triggers:** Any database not ONLINE

**Recommendations:** Stop everything and read error log before recovery attempts

### CHECK 13: Failed Login Attempts

**What it monitors:** Failed login spike detection

**When it triggers:** More than 10 failed logins in last hour

**Recommendations:** Check for brute-force attacks, verify application credentials

### CHECK 14: AlwaysOn AG Health

**What it monitors:** Availability Group synchronization status (if configured)

**When it triggers:** Replica health not HEALTHY

**Recommendations:** Check network latency, verify secondary resources

### CHECK 15: Replication Health

**What it monitors:** Replication pending commands (if configured)

**When it triggers:** Large backlog at subscriber

**Recommendations:** Verify distribution agent running, check subscriber blocking

### CHECK 16: TempDB Health

**What it monitors:** TempDB space usage

**When it triggers:** Usage exceeding 1GB

**Recommendations:** Identify consuming query, confirm multiple TempDB files configured

### CHECK 17: Wait Statistics

**What it monitors:** Top wait types by duration

**When it triggers:** Provides diagnostic insight (always runs)

**Interpretation:**
- PAGEIOLATCH: Disk I/O bottleneck
- CXPACKET: Parallelism tuning needed
- LCK_M_*: Blocking/deadlock issues
- CPU: CPU-bound workload

### CHECK 18: Index Fragmentation

**What it monitors:** Index fragmentation on tables over 1000 pages

**When it triggers:** Fragmentation over 30%

**Recommendations:** Schedule reorganize (10-30%) or rebuild (>30%) in maintenance window

### CHECK 19: Missing Indexes

**What it monitors:** High-impact missing index recommendations

**When it triggers:** Provides guidance (always runs)

**Caution:** Do not blindly create - cross-check against existing indexes

### CHECK 20: Schema Changes

**What it monitors:** Recent object modifications

**When it triggers:** Object modified in last 24 hours

**Recommendations:** Verify authorization, confirm change management process followed

### CHECK 21: Configuration Drift

**What it monitors:** Server configuration deviations from baseline

**When it triggers:** Max server memory set below 2048 MB

**Recommendations:** Verify intentional, check for recent patches/restores

### CHECK 22: SQL Services Status

**What it monitors:** System database backup age

**When it triggers:** System database backup older than 24 hours

**Recommendations:** Backup system databases, confirm no startup errors

### CHECK 23: Linked Server Connectivity

**What it monitors:** Configured linked servers

**When it triggers:** Provides current configuration status

**Recommendations:** Verify credentials current, check firewall/network

### CHECK 24: Query Store

**What it monitors:** Query Store performance trends (if enabled)

**When it triggers:** Provides high-resource consumer info

**Recommendations:** Force last good plan if regression detected

### CHECK 25: Monitoring System Health

**What it monitors:** Monitoring system itself

**When it triggers:** No alerts in last 24 hours

**Critical:** If this fails, you cannot trust other results

---

## Output Tables

### Issues Summary Table
Columns: #, Check Name, Section, Priority, Issue, Job Name, Job Failure Details, Affected Object, Recommended Action

- **Job Name & Job Failure Details:** Only populated for Check 1 (Agent job failures)
- **All other columns:** Populated for their respective checks

### Backup Verification Details Table
Columns: Database Name, Full Backup, Log Backup, Recovery Model, Status, Recommendation

- Shows only databases with backup issues
- Status column indicates specific problem (Never backed up, Overdue, OK, etc.)
- Recommendation column in red with actionable guidance

### Database Corruption Details Table
Columns: Database Name, Last CHECKDB, Age (Days), Status, Recommendation

- Shows only databases with stale or missing CHECKDB
- Age helps prioritize remediation
- Recommendation column in red with specific actions

---

## Customization

### Change Email Recipients

```sql
DECLARE @DBAEmailList NVARCHAR(MAX) = 'newdba@company.com,backup@company.com';
```

### Change Job Failure Lookup Window

Line 67-76: Change `DATEADD(HOUR, -24, GETDATE())` to different interval

### Change Backup SLA

Line 124-126: Modify days/hours thresholds

### Change Disk Space Threshold

Line 148: Change `< 20` to different percentage

### Change Long-Running Query Threshold

Line 255: Change `> 900` to different seconds

### Disable Email Alerts

Comment out the email sending section (around line 700-750)

---

## Troubleshooting

### Issue: Email Not Sending

```sql
-- Verify mail profile configured
SELECT * FROM msdb.dbo.sysmail_profile WHERE name = 'DBAAlerts';

-- Check mail history
SELECT TOP 20 sent_status, subject, send_request_date
FROM msdb.dbo.sysmail_sentitems
ORDER BY send_request_date DESC;

-- Check mail log for errors
SELECT TOP 20 mailitem_id, attendee_address, send_request_date, last_mod_date
FROM msdb.dbo.sysmail_event_log
WHERE event_type = 'failure'
ORDER BY last_mod_date DESC;

-- Send test email
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'DBAAlerts',
    @recipients = 'test@company.com',
    @subject = 'Test',
    @body = 'Test email';
```

### Issue: Procedure Takes Too Long

The procedure uses DMVs which are very efficient. Check:
- Is CHECKDB running simultaneously? (conflicts with DATABASEPROPERTYEX)
- Is server under heavy load?
- Try increasing schedule interval (less frequent)

### Issue: False Positives

Review thresholds for your environment. For example:
- New server with no backup history
- Development server with intentional SIMPLE recovery mode
- Long-running ETL process that is expected

Modify thresholds accordingly.

### Issue: No Issues Detected When Issues Exist

The procedure only reports configured thresholds. If an issue type isn't triggering:
1. Verify the threshold matches your SLA
2. Check if the check is conditional (some only run if feature is enabled)
3. Run the individual check query to debug

---

## Maintenance

### Weekly Review
- Scan email alerts for patterns
- Verify issues are being addressed per SLA
- Update team if baseline changes

### Monthly Review
- Analyze trends in alerts
- Adjust thresholds based on actual environment
- Test restore capability (especially for backups)
- Verify linked servers and mail profile still functional

### Quarterly Review
- Review all 25 checks for applicability
- Document any threshold changes
- Train new team members on alert interpretation
- Performance tune slow checks if needed

### Annual Review
- Assess overall health monitoring strategy
- Consider additional checks based on incidents
- Update contact lists and distribution groups
- Document lessons learned

---

## Performance Notes

- **Execution Time:** 30-60 seconds typically
- **Resource Impact:** Minimal (uses DMVs and catalog views only)
- **Network Impact:** Only sends email if issues found
- **Best Practice:** Schedule during non-peak hours, but 7 AM / 6 PM is standard

---

## SQL Server Versions

Tested and supported on:
- SQL Server 2016 Standard, Enterprise
- SQL Server 2017 Standard, Enterprise
- SQL Server 2019 Standard, Enterprise
- SQL Server 2022 Standard, Enterprise
- Azure SQL Database (limited checks)

---

## Dependencies

- **Database Mail:** Must be configured and operational
- **SQL Server Agent:** Required for scheduled execution
- **DMVs:** Standard DMVs available in all editions
- **Permissions:** Must run as sysadmin or user with access to:
  - sys.databases
  - sys.dm_exec_* views
  - msdb.dbo.sysjobs
  - msdb.dbo.backupset

---

## Support & Feedback

For issues or suggestions:
1. Review this README thoroughly
2. Check SQL Server error log for clues
3. Run procedure manually to test
4. Consult with database team
5. Review Anthropic DBA Health Check documentation

---

[SP Script] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_DBA_HealthCheck)

---

## License & Disclaimer

This stored procedure is provided as-is for internal use within your organization. 
Test thoroughly in non-production environments before deploying to production. 
Database Administration team is responsible for customization and maintenance.

No warranty or support is guaranteed. Use at your own risk.

---

**Last Updated:** July 29, 2026 - Wednesday
**Status:** Production Ready
