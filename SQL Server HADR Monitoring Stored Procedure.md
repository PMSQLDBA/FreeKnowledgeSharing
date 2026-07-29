# SQL Server HADR Monitoring Stored Procedure

## Overview

`usp_HADR_HealthCHECK` is a specialized SQL Server monitoring stored procedure that monitors four critical data availability and disaster recovery technologies: 
Log Shipping, Database Mirroring, Replication, and AlwaysOn Availability Groups. 

It delivers consolidated HTML email alerts when issues are detected.

**Procedure Name:** `usp_HADR_HealthCHECK`

**Drafted by:** Praveen Madupu

**Version:** 1.0

**Last Updated:** 29th July 2026 - Wednesday

**Compatibility:** SQL Server 2016 and later

**Status:** Production Ready

[**Script**] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_HADR_HealthCHECK)

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Features](#features)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Usage](#usage)
6. [Monitored Technologies](#monitored-technologies)
7. [Email Alerts](#email-alerts)
8. [Console Output](#console-output)
9. [Scheduling](#scheduling)
10. [Troubleshooting](#troubleshooting)
11. [FAQ](#faq)

---

## Quick Start

### Prerequisites

- SQL Server 2016 or later
- SQL Server Management Studio (SSMS)
- Database Mail configured and operational
- sysadmin or equivalent permissions
- SQL Server Agent running (for scheduled execution)

### Installation (5 minutes)

1. **Download the file:**
   ```
   usp_HADR_HealthCHECK_CORRECTED.sql  --> [**Script**] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_HADR_HealthCHECK)
   ```

2. **Connect to SQL Server:**
   - Open SQL Server Management Studio
   - Connect to the server you want to monitor
   - Select the `master` database

3. **Create the procedure:**
   - Open the SQL file
   - Execute the entire script
   - Result: "Command(s) completed successfully"

4. **Verify installation:**
   ```sql
   USE master;
   GO
   SELECT * FROM sys.procedures WHERE name = 'usp_HADR_HealthCHECK';
   GO
   ```

5. **Test execution:**
   ```sql
   USE master;
   GO
   EXEC usp_HADR_HealthCHECK;
   GO
   ```

### First Time Setup (10 minutes)

1. **Configure email settings in the procedure:**
   - Line 10: Set email profile name
   - Line 11: Set recipient email address(es)

2. **Edit the procedure:**
   ```sql
   ALTER PROCEDURE usp_HADR_HealthCHECK
   AS
   BEGIN
       DECLARE @ProfileName NVARCHAR(128) = 'DBAAlerts';      -- Change this
       DECLARE @DBAEmailList NVARCHAR(MAX) = 'your@email.com'; -- Change this
   ```

3. **Verify Database Mail is operational:**
   ```sql
   -- Test email
   EXEC msdb.dbo.sp_send_dbmail
       @profile_name = 'DBAAlerts',
       @recipients = 'your@email.com',
       @subject = 'Test Email',
       @body = 'This is a test email';
   ```

---

## Features

### Four Availability Technologies Monitored

#### 1. Log Shipping
- Verifies backup jobs are configured
- Detects missing backup job configuration
- Identifies when backup job is not set up

**Status Indicators:**
- ✅ OK - Backup job configured
- ⚠️ WARNING - No backup job configured

#### 2. Database Mirroring
- Monitors mirror state in real-time
- Tracks principal and mirror server status
- Detects connection issues

**Status Indicators:**
- ✅ OK - Synchronized/Hardened state
- ⚠️ WARNING - Synchronizing (data in transit)
- 🔴 CRITICAL - Suspended or Disconnected

#### 3. Replication
- Monitors subscription initialization status
- Tracks active subscriptions
- Identifies uninitialized or inactive subscriptions

**Status Indicators:**
- ✅ OK - Active (status = 2)
- ⚠️ WARNING - Subscribed but not active (status = 1)
- ⚠️ WARNING - Uninitialized (status = 0)

#### 4. AlwaysOn Availability Groups
- Monitors replica connection status
- Tracks synchronization health
- Identifies offline or unhealthy replicas

**Status Indicators:**
- ✅ OK - Replica healthy and connected
- ⚠️ WARNING - Partially healthy
- 🔴 CRITICAL - Disconnected or not healthy

### Smart Alerts

- **Conditional Reporting:** Only reports issues that exist
- **Graceful Degradation:** Skips sections where features aren't configured
- **HTML Email:** Professional formatted alerts with red-text recommendations
- **Console Output:** Real-time progress tracking during execution
- **No False Positives:** Only alerts on actual problems

---

## Installation

### Step 1: Verify Prerequisites

```sql
-- Check SQL Server version
SELECT @@VERSION;

-- Check if Database Mail is configured
SELECT * FROM msdb.dbo.sysmail_profile;

-- Check if SQL Server Agent is running
SELECT status, status_desc FROM sys.dm_server_services WHERE servicename LIKE '%SQL Server Agent%';
```

### Step 2: Create the Procedure

```sql
USE master;
GO
-- Paste the entire usp_HADR_HealthCHECK_CORRECTED.sql file here
-- ...
GO
```

### Step 3: Verify Creation

```sql
USE master;
GO
EXEC sys.sp_help 'usp_HADR_HealthCHECK';
GO
```

### Step 4: Configure Email

Edit lines 10-11:

```sql
DECLARE @ProfileName NVARCHAR(128) = 'DBAAlerts';
DECLARE @DBAEmailList NVARCHAR(MAX) = 'dba-team@company.com';
```

---

## Configuration

### Email Profile Setup

#### Option A: Using Existing Mail Profile

If you already have Database Mail configured:

```sql
-- View existing profiles
SELECT name FROM msdb.dbo.sysmail_profile;

-- Use in procedure
DECLARE @ProfileName NVARCHAR(128) = 'YourExistingProfileName';
```

#### Option B: Create New Mail Profile

```sql
-- Step 1: Create account
EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'DBAAlerts_Account',
    @email_address = 'sqlserver-alerts@company.com',
    @mailserver_name = 'smtp.company.com',
    @port = 25;

-- Step 2: Create profile
EXEC msdb.dbo.sysmail_add_profile_sp
    @profile_name = 'DBAAlerts',
    @description = 'DBA Alerts Profile';

-- Step 3: Add account to profile
EXEC msdb.dbo.sysmail_add_profileaccount_sp
    @profile_name = 'DBAAlerts',
    @account_name = 'DBAAlerts_Account',
    @sequence_number = 1;

-- Step 4: Grant public access
EXEC msdb.dbo.sysmail_add_principalprofile_sp
    @profile_name = 'DBAAlerts',
    @principal_name = 'public',
    @is_default = 1;
```

### Multiple Email Recipients

```sql
DECLARE @DBAEmailList NVARCHAR(MAX) = 'dba1@company.com;dba2@company.com;dba3@company.com';
```

### Customize Email Sender

Edit the procedure to add sender information:

```sql
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = @ProfileName,
    @recipients = @DBAEmailList,
    @subject = @EmailSubject,
    @body = @EmailBody,
    @body_format = 'HTML',
    @from_address = 'sqlserver-alerts@company.com';
```

---

## Usage

### Manual Execution

Run on-demand to check current status:

```sql
USE master;
GO
EXEC usp_HADR_HealthCHECK;
GO
```

### Expected Console Output

```
================================================================================
SQL SERVER REPLICATION & AVAILABILITY MONITORING - 2026-07-28 23:34:20.697
Server: PMSQLDBA
================================================================================

>>> SECTION 1: LOG SHIPPING MONITORING
    Log Shipping: Not Configured

>>> SECTION 2: DATABASE MIRRORING MONITORING
    Database Mirroring: Not Configured

>>> SECTION 3: REPLICATION MONITORING
    Replication: Not Configured

>>> SECTION 4: ALWAYSON AVAILABILITY GROUP MONITORING
    AlwaysOn Availability Groups: Not Configured

================================================================================
REPLICATION & AVAILABILITY MONITORING SUMMARY
================================================================================
Log Shipping Issues: 0
Database Mirroring Issues: 0
Replication Issues: 0
AlwaysOn AG Issues: 0
Total Issues: 0

================================================================================
No issues found.
================================================================================
```

### Interpreting Output

**When features are not configured:**
```
Log Shipping: Not Configured
```
- This is normal if you don't use Log Shipping
- No email sent when no issues exist

**When issues are found:**
```
>>> LOG SHIPPING DETAILS
PrimaryDatabase    Status               AlertMessage
OrdersDB           WARNING - No Backup  Backup job not configured...
```
- Email sent to configured recipients
- HTML table with red-text recommendations

---

## Monitored Technologies

### 1. Log Shipping Monitoring

**What It Checks:**
- Backup job configuration status

**Columns in Report:**
- `PrimaryDatabase` - Name of primary database
- `Status` - OK or WARNING
- `AlertMessage` - Recommended action (in red)

**When It Triggers:**
- Backup job not configured
- Manual intervention needed

**Example Alert:**
```
Status: WARNING - No Backup Job
Action: Backup job not configured. Configure log shipping backup job.
```

---

### 2. Database Mirroring Monitoring

**What It Checks:**
- Mirror state (Suspended, Disconnected, Synchronizing, Synchronized)
- Principal and mirror server connectivity
- Real-time mirror status

**Columns in Report:**
- `DatabaseName` - Mirrored database name
- `MirrorState` - Current state
- `MirrorRole` - Principal or Mirror
- `PrincipalServer` - Principal server name
- `MirrorServer` - Mirror server name
- `AlertMessage` - Recommended action (in red)

**Mirror States:**
- **Suspended (Yellow):** Mirroring paused; session not active
- **Disconnected (Red):** Network issue between principal and mirror
- **Synchronizing (Yellow):** Data being sent to mirror; not yet fully synchronized
- **Synchronized (Green):** Principal and mirror fully synchronized
- **Hardened (Green):** Optimal state; fully synchronized

**When It Triggers:**
- Mirror state is Suspended
- Mirror state is Disconnected
- Mirror state is Synchronizing (data in transit)

**Example Alerts:**
```
CRITICAL: Mirroring suspended. Resume mirroring immediately.
CRITICAL: Mirroring disconnected. Check network connectivity.
WARNING: Mirroring synchronizing. Wait for synchronized state.
```

---

### 3. Replication Monitoring

**What It Checks:**
- Subscription status (Uninitialized, Subscribed, Active)
- Subscriber server and database status
- Publication subscription health

**Columns in Report:**
- `SubscriberServer` - Subscriber server name
- `SubscriptionDatabase` - Subscription database name
- `ReplicationStatus` - Uninitialized, Subscribed, or Active
- `AlertMessage` - Recommended action (in red)

**Subscription States:**
- **Uninitialized (0):** Not yet initialized with snapshot data
- **Subscribed (1):** Subscription active but not all transactions applied
- **Active (2):** Subscription active and current with publisher

**When It Triggers:**
- Subscription status is 0 (Uninitialized)
- Subscription status is 1 (Subscribed but not fully active)

**Example Alerts:**
```
WARNING: Subscription not initialized. Run snapshot and distribution agent.
ACTIVE: Subscription subscribed. Monitor for latency.
```

---

### 4. AlwaysOn Availability Group Monitoring

**What It Checks:**
- Replica connection state (Connected/Disconnected)
- Synchronization health (Healthy/Partially Healthy/Not Healthy)
- Replica operational status

**Columns in Report:**
- `AvailabilityGroupName` - Name of AG
- `ReplicaName` - Replica server name
- `ReplicaRole` - Replica designation
- `ConnectedState` - Connected or Disconnected
- `SyncHealth` - Health status
- `AlertMessage` - Recommended action (in red)

**Health States:**
- **HEALTHY (Green):** Replica healthy and synchronized
- **PARTIALLY_HEALTHY (Yellow):** Some replicas or databases not synchronized
- **NOT_HEALTHY (Red):** Replica failed or unable to communicate

**When It Triggers:**
- Replica connection state is Disconnected (0)
- Synchronization health is not HEALTHY
- Any replica health issue

**Example Alerts:**
```
CRITICAL: Replica disconnected. Check network and rejoin cluster.
CRITICAL: Replica not healthy. Check error log.
WARNING: Some replicas not synchronized.
```

---

## Email Alerts

### When Emails Are Sent

Emails are sent **only** when issues are detected. If all systems are healthy, no email is sent.

### Email Subject Line

```
SQL Server Replication & Availability Alert - 3 Issue(s) on PMSQLDBA
```

### Email Body Structure

1. **Header Section**
   - Server name
   - Execution timestamp

2. **Summary Section**
   - Total issues found
   - Count by technology type

3. **Issue Tables** (if applicable)
   - Log Shipping issues (if any)
   - Mirroring issues (if any)
   - Replication issues (if any)
   - AlwaysOn AG issues (if any)

4. **Footer Section**
   - Automated alert notice
   - Support contact information

### Email Styling

- **Header:** Dark blue background (#2F5496) with white text
- **Table Headers:** Blue background (#4472C4) with white text
- **Data Rows:** White background for readability
- **Alert Messages:** Red text (#FF0000) for immediate visibility

### Example Email Content

```
SQL Server Replication & Availability Monitoring Alert

Server: PMSQLDBA
Execution Time: 2026-07-28 23:34:20

Summary
Total Issues Found: 1
Log Shipping Issues: 0
Database Mirroring Issues: 1
Replication Issues: 0
AlwaysOn AG Issues: 0

Database Mirroring Monitoring
Database      State          Role         Principal      Mirror         Alert
SalesDB       Disconnected   Mirror       PRIMARY01      SECONDARY01    CRITICAL: Mirroring disconnected...
```

---

## Console Output

### Output Sections

The procedure displays real-time progress in SQL Server Management Studio "Messages" tab:

#### Header Section
```
================================================================================
SQL SERVER REPLICATION & AVAILABILITY MONITORING - 2026-07-28 23:34:20.697
Server: PMSQLDBA
================================================================================
```

#### Progress Section
```
>>> SECTION 1: LOG SHIPPING MONITORING
    Log Shipping: Not Configured

>>> SECTION 2: DATABASE MIRRORING MONITORING
    Database Mirroring Details Collected: 1 issue found
    
>>> SECTION 3: REPLICATION MONITORING
    Replication: Not Configured
    
>>> SECTION 4: ALWAYSON AVAILABILITY GROUP MONITORING
    AlwaysOn Availability Groups: Not Configured
```

#### Summary Section
```
================================================================================
REPLICATION & AVAILABILITY MONITORING SUMMARY
================================================================================
Log Shipping Issues: 0
Database Mirroring Issues: 1
Replication Issues: 0
AlwaysOn AG Issues: 0
Total Issues: 1
```

#### Details Tables (only if issues found)
```
>>> DATABASE MIRRORING DETAILS
DatabaseName    MirrorState    MirrorRole    PrincipalServer    MirrorServer    AlertMessage
SalesDB         Disconnected   Mirror        PRIMARY01          SECONDARY01     CRITICAL: Mirroring...
```

#### Completion Section
```
================================================================================
Email notification sent successfully to: dba-team@company.com
================================================================================
```

---

## Scheduling

### Schedule with SQL Server Agent

Create two jobs: one in morning, one in evening for comprehensive monitoring.

#### Job 1: Morning Check (7:00 AM)

```sql
USE msdb;
GO

-- Create job
EXEC sp_add_job
    @job_name = 'Replication_Availability_Monitor_7AM',
    @enabled = 1,
    @description = 'Monitor Log Shipping, Mirroring, Replication, and AlwaysOn AG at 7 AM';

-- Create step
EXEC sp_add_jobstep
    @job_name = 'Replication_Availability_Monitor_7AM',
    @step_name = 'Execute Monitoring',
    @subsystem = 'TSQL',
    @command = 'EXEC master.dbo.usp_HADR_HealthCHECK',
    @database_name = 'master';

-- Create schedule
EXEC sp_add_schedule
    @schedule_name = 'Daily_7AM',
    @freq_type = 4,           -- Daily
    @freq_interval = 1,        -- Every day
    @active_start_time = 070000; -- 7:00 AM

-- Attach schedule to job
EXEC sp_attach_schedule
    @job_name = 'Replication_Availability_Monitor_7AM',
    @schedule_name = 'Daily_7AM';

-- Add job to server
EXEC sp_add_jobserver
    @job_name = 'Replication_Availability_Monitor_7AM',
    @server_name = N'(local)';
```

#### Job 2: Evening Check (6:00 PM)

```sql
USE msdb;
GO

-- Create job
EXEC sp_add_job
    @job_name = 'Replication_Availability_Monitor_6PM',
    @enabled = 1,
    @description = 'Monitor Log Shipping, Mirroring, Replication, and AlwaysOn AG at 6 PM';

-- Create step
EXEC sp_add_jobstep
    @job_name = 'Replication_Availability_Monitor_6PM',
    @step_name = 'Execute Monitoring',
    @subsystem = 'TSQL',
    @command = 'EXEC master.dbo.usp_HADR_HealthCHECK',
    @database_name = 'master';

-- Create schedule
EXEC sp_add_schedule
    @schedule_name = 'Daily_6PM',
    @freq_type = 4,           -- Daily
    @freq_interval = 1,        -- Every day
    @active_start_time = 180000; -- 6:00 PM

-- Attach schedule to job
EXEC sp_attach_schedule
    @job_name = 'Replication_Availability_Monitor_6PM',
    @schedule_name = 'Daily_6PM';

-- Add job to server
EXEC sp_add_jobserver
    @job_name = 'Replication_Availability_Monitor_6PM',
    @server_name = N'(local)';
```

### Verify Scheduled Jobs

```sql
-- View job schedule
SELECT 
    j.name AS JobName,
    s.name AS ScheduleName,
    CASE s.freq_type
        WHEN 1 THEN 'Once'
        WHEN 4 THEN 'Daily'
        WHEN 8 THEN 'Weekly'
        WHEN 16 THEN 'Monthly'
        WHEN 32 THEN 'Monthly Relative'
        WHEN 64 THEN 'At SQL Server Agent Start'
        WHEN 128 THEN 'On Idle'
    END AS FrequencyType,
    CONVERT(NVARCHAR(8), s.active_start_time) AS StartTime
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
INNER JOIN msdb.dbo.sysschedules s ON js.schedule_id = s.schedule_id
WHERE j.name LIKE 'Replication_Availability%'
ORDER BY j.name;
```

### Adjust Frequency

Run every hour (instead of 7 AM and 6 PM):

```sql
EXEC sp_add_schedule
    @schedule_name = 'Every_Hour',
    @freq_type = 4,              -- Daily
    @freq_interval = 1,           -- Every day
    @freq_subday_type = 4,        -- Hour
    @freq_subday_interval = 1;    -- Every 1 hour
```

---

## Troubleshooting

### Issue 1: Email Not Sending

**Symptom:** Procedure runs but no email received

**Diagnosis:**
```sql
-- Check if mail profile exists
SELECT * FROM msdb.dbo.sysmail_profile WHERE name = 'DBAAlerts';

-- Check recent mail history
SELECT TOP 20 
    sent_status, 
    subject, 
    send_request_date
FROM msdb.dbo.sysmail_sentitems
ORDER BY send_request_date DESC;

-- Check for mail errors
SELECT TOP 20 
    mailitem_id,
    event_type,
    log_date,
    description
FROM msdb.dbo.sysmail_event_log
WHERE event_type = 'failure'
ORDER BY log_date DESC;
```

**Solution:**
1. Verify mail profile exists and is configured correctly
2. Test email manually:
   ```sql
   EXEC msdb.dbo.sp_send_dbmail
       @profile_name = 'DBAAlerts',
       @recipients = 'your@email.com',
       @subject = 'Test Email',
       @body = 'This is a test';
   ```
3. Check SMTP server connectivity
4. Verify recipient email address is correct

### Issue 2: "Log Shipping: Not Configured"

**Symptom:** Procedure shows "Log Shipping: Not Configured" even though it's set up

**Cause:** Log Shipping monitoring checks for backup jobs only

**Solution:**
- Log Shipping monitoring is basic in this version
- Verify backup jobs exist in SQL Server Agent
- This is expected behavior - procedure only reports if backup job not found

### Issue 3: "Invalid object name 'msdb.dbo.MSreplication_subscriptions'"

**Symptom:** Error when running procedure on server without replication

**Status:** FIXED in current version

**What changed:**
- Procedure now checks if replication is configured before querying
- Shows "Replication: Not Configured" instead of throwing error
- Gracefully handles missing features

### Issue 4: Procedure Runs Slowly

**Symptom:** Procedure takes longer than expected to complete

**Typical Execution Times:**
- No issues found: 5-10 seconds
- With issues found: 10-20 seconds
- First run: 15-30 seconds

**Optimization:**
1. Run during non-peak hours
2. Check server CPU and disk utilization
3. Verify DMV queries aren't blocked by other processes

### Issue 5: Too Many Alert Emails

**Symptom:** Receiving duplicate or excessive alerts

**Solution:**
1. Adjust scheduled job frequency
2. Add condition to only run if previous job completed
3. Create alert suppression logic for known maintenance windows

```sql
-- Example: Only send alerts during business hours
IF DATEPART(HOUR, GETDATE()) BETWEEN 6 AND 22
BEGIN
    -- Send email logic here
END
```

---

## FAQ

### Q: What SQL Server versions does this support?

**A:** SQL Server 2016 and later (2016, 2017, 2019, 2022)

### Q: Can I run this on Azure SQL Database?

**A:** No. Azure SQL Database doesn't support:
- Database Mirroring (deprecated)
- Log Shipping (requires SQL Server Agent)
- Replication (limited support)

Use Azure SQL Database with AlwaysOn AG only.

### Q: Will this SP affect production performance?

**A:** No. The procedure:
- Uses only DMVs (no locks)
- Runs read-only queries
- Typical execution: 10-20 seconds
- Negligible I/O impact
- Safe to run hourly

### Q: Can I modify the email format?

**A:** Yes. Edit the email body construction around line 250-290:

```sql
SET @EmailBody = '<html><head><style>...'
```

### Q: How do I add more email recipients?

**A:** Use semicolon-separated email addresses:

```sql
DECLARE @DBAEmailList NVARCHAR(MAX) = 'dba1@company.com;dba2@company.com;dba3@company.com';
```

### Q: What if I don't want emails, just console output?

**A:** Comment out the email section:

```sql
-- Email sending logic (lines 270-285)
/*
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = @ProfileName,
        @recipients = @DBAEmailList,
        @subject = @EmailSubject,
        @body = @EmailBody,
        @body_format = 'HTML';
*/
```

### Q: Can I customize alert thresholds?

**A:** The current version uses fixed thresholds based on Microsoft best practices:
- Log Shipping: Backup job configured
- Mirroring: State not Suspended/Disconnected
- Replication: Subscription status = 2 (Active)
- AlwaysOn: Health = HEALTHY and Connected

To customize, edit the WHERE clauses in each SELECT statement.

### Q: How do I disable monitoring for a specific feature?

**A:** Comment out the entire section, e.g.:

```sql
-- Disable Log Shipping monitoring
/*
    PRINT '>>> SECTION 1: LOG SHIPPING MONITORING';
    IF EXISTS (SELECT 1 FROM msdb.dbo.log_shipping_primary_databases)
    BEGIN
        ...
    END
*/
```

### Q: What permissions does the procedure require?

**A:** 
- VIEW SERVER STATE (for DMVs)
- READ msdb database
- EXECUTE permission on sp_send_dbmail

Recommended: Run as sysadmin or user with similar permissions

### Q: Can I schedule this to run on a remote server?

**A:** Yes. From any server with SQL Server Agent:

```sql
-- Target remote server
EXEC sp_add_jobstep
    @job_name = 'Remote_Monitoring',
    @step_name = 'Monitor Target Server',
    @subsystem = 'TSQL',
    @command = 'EXEC [TargetServerName].master.dbo.usp_HADR_HealthCHECK',
    @server_name = 'TargetServerName';
```

### Q: Where are the alerts stored?

**A:** Alerts are:
- Sent via email (HTML format)
- Stored in msdb.dbo.sysmail_sentitems (email history)
- Displayed in SSMS Messages tab (console output)

No database table storage - alerts are transient.

---

## Maintenance Schedule

### Daily
- Review alert emails for patterns
- Verify all features operational
- Check if jobs completed successfully

### Weekly
- Analyze alert trends
- Update recipient email list if needed
- Test email delivery

### Monthly
- Test failover procedures for critical systems
- Verify backup restore capability
- Review and update configuration if needed
- Document any manual fixes applied

### Quarterly
- Review procedure for SQL Server updates
- Evaluate if monitoring thresholds still appropriate
- Train new team members on alert interpretation
- Update this documentation with any changes

---

## Support & Contact

For issues or questions:

1. Review this README thoroughly
2. Check SQL Server error log for related messages
3. Run procedure manually to test:
   ```sql
   EXEC master.dbo.usp_HADR_HealthCHECK;
   ```
4. Consult with your database administration team
5. Review SQL Server system health DMVs

---

## License & Disclaimer

This stored procedure is provided as-is for internal use within your organization. 

**Disclaimer:**
- Test thoroughly in non-production environments before deployment
- Database Administration team is responsible for customization and maintenance
- No warranty or support is guaranteed
- Use at your own risk
- Review and comply with your organization's change management policies

---

## Related Documentation

- [Microsoft Log Shipping Documentation](https://learn.microsoft.com/en-us/sql/database-engine/log-shipping/about-log-shipping-sql-server?view=sql-server-ver17)
- [Microsoft Database Mirroring Documentation](https://learn.microsoft.com/en-us/sql/database-engine/database-mirroring/database-mirroring-sql-server)
- [Microsoft Replication Documentation](https://learn.microsoft.com/en-us/sql/relational-databases/replication/sql-server-replication)
- [Microsoft AlwaysOn AG Documentation](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server)

---

**Last Updated:** July 29, 2026
**Status:** Production Ready
**Drafted by:** Praveen Madupu
