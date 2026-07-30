# SQL SERVER DATABASE LOG SHIPPING REAL-TIME SCENARIO 100 FAQs

## Top 100 Frequently Asked Questions with Pointwise Answers (5-6 Points Each)

Based on Microsoft Official Documentation

---

## SECTION 1: FUNDAMENTALS AND CONCEPTS

### FAQ 1: What exactly is SQL Server Log Shipping?

- Automated disaster recovery solution that continuously backs up transaction logs from primary database and restores them on secondary databases
- Involves three continuous jobs: backup job creates transaction logs, copy job transfers files to secondary, restore job applies logs to secondary database
- Operates asynchronously with scheduled intervals between backup, copy, and restore operations, creating a time lag between primary and secondary updates
- Works at database level not server level, allowing selective protection of specific databases while others remain unprotected
- Provides database-level protection for disaster recovery and supports read-only access on secondary databases during intervals between restore jobs
- Requires SQL Server Agent to run jobs and shared network folder for log file transfer between servers

---

### FAQ 2: How does Log Shipping differ from Database Mirroring and AlwaysOn Availability Groups?

- Log shipping operates asynchronously with scheduled backup, copy, and restore jobs as multiple discrete steps with time delays
- Database mirroring operates synchronously or asynchronously with automatic failover capability depending on high-safety or high-performance mode
- AlwaysOn Availability Groups provides synchronous or asynchronous replication with automatic failover and multiple readable secondary replicas
- Log shipping requires manual failover intervention after failure, whereas mirroring and availability groups support automatic failover capability
- Log shipping works with any SQL Server edition including Standard Edition, while AlwaysOn requires Enterprise Edition
- Log shipping is simpler to configure and less resource intensive than both database mirroring and availability groups

---

### FAQ 3: What are the main components of a Log Shipping configuration?

- Primary database receives initial full and differential backups, then continues creating transaction log backups on recurring schedule
- Secondary databases receive and restore transaction log backups to stay synchronized with primary database
- Monitor server is optional and tracks status and history of all backup, copy, and restore operations across entire configuration
- SQL Server Agent jobs run on each server to automate backup, copy, and restore processes without manual intervention
- Monitor server records history in msdb tables and raises alerts if operations fail or exceed predefined threshold times
- Network shared folder facilitates copying log backup files from primary server to secondary servers

---

### FAQ 4: What recovery models are required for Log Shipping?

- Primary database requires FULL or BULK-LOGGED recovery model to maintain transaction log backups available for shipping
- SIMPLE recovery model purges transaction log after checkpoint operations, leaving nothing to ship, making it incompatible with log shipping
- Must change database recovery model from SIMPLE to FULL before enabling log shipping on that database
- Secondary databases follow recovery model of their backups during restoration process
- Recovery model setting determines how much transaction log history remains available for backup operations
- Cannot enable log shipping on database until recovery model is changed from SIMPLE to FULL or BULK-LOGGED

---

### FAQ 5: What are the licensing requirements for Log Shipping?

- Log shipping works with any edition of SQL Server from Standard Edition upward without requiring Enterprise Edition
- Monitor server can use SQL Server Express Edition without additional licensing costs for monitoring functionality only
- Monitor server functionality is identical across all editions, allowing cost optimization by using Express Edition for monitoring
- SQL Server Agent must be running on all servers involved and is included in standard SQL Server installations
- No additional license requirements or restrictions apply for using log shipping as disaster recovery solution
- Organizations can significantly reduce licensing costs by implementing log shipping instead of other high availability solutions

---

### FAQ 6: Can you use Log Shipping with Azure SQL databases?

- Azure SQL Database does not support traditional log shipping configuration using SQL Server Agent jobs
- Azure offers alternative disaster recovery solutions like automated backups with geo-replication and failover groups
- You can ship logs from on-premises SQL Server to Azure VM running SQL Server using standard log shipping
- If migrating to Azure cloud, failover groups and geo-replication are recommended approaches for Azure SQL Database
- For SQL Server on Azure VMs, log shipping works identically to on-premises SQL Server installations
- Documentation recommends evaluating Azure native solutions before implementing log shipping for cloud deployments

---

## SECTION 2: CONFIGURATION AND SETUP

### FAQ 7: What are the prerequisites before configuring Log Shipping?

- Create shared network folder accessible to both primary and secondary servers with appropriate NTFS permissions granted to service accounts
- Grant SQL Server service accounts on primary and secondary servers full control permissions to the shared backup folder
- Ensure both servers are on same network or connected via secure network links for reliable log file transfer
- Verify SQL Server Agent is running and set to start automatically on all involved servers before configuration
- Create secondary database by restoring full backup of primary database with NORECOVERY option to establish starting point
- Primary database must be in FULL or BULK-LOGGED recovery model before log shipping can be enabled

---

### FAQ 8: What are the step-by-step configuration tasks for Log Shipping?

- Perform full backup of primary database and restore it on secondary server with NORECOVERY option to create secondary database
- Enable log shipping on primary database by right-clicking database, selecting Properties, then navigating to Shipping section
- Configure backup job schedule specifying backup location on primary server and frequency of transaction log backups
- Add secondary databases and configure their restore job schedules with appropriate delays between backup and restore
- Set up monitoring by adding monitor server instance if desired for centralized status and alert tracking
- Configure backup and restore job history retention periods in monitoring settings to manage msdb database size
- Test configuration by allowing jobs to run and verify backup, copy, and restore operations complete successfully

---

### FAQ 9: How do you add a second or third secondary database to an existing Log Shipping configuration?

- Go to primary database properties and Shipping section in SQL Server Management Studio to add new secondary database
- Click "Add" button in Secondary Databases area to register new secondary database in configuration
- Specify secondary server instance name and secondary database name that will receive log backups
- Restore full backup of primary database to secondary server with NORECOVERY option before adding to configuration
- Configure copy job settings with network path to shared backup folder where primary backups are stored
- Configure restore job settings including schedule and restore delay period for the new secondary database
- New secondary database begins receiving log backups at next scheduled backup job on primary server

---

### FAQ 10: What is the role of the Monitor Server in Log Shipping?

- Monitor server is optional but highly recommended for production configurations to track backup, copy, and restore job status
- Maintains centralized history and status records in its msdb database rather than relying on local job histories only
- Can raise alerts if backup or restore jobs fail or if time threshold between operations exceeds defined limits
- Monitoring information is still stored locally on primary and secondary servers in addition to monitor server records
- Cannot be changed or reassigned after configuration without removing entire log shipping configuration first
- Using dedicated monitor instance improves operational visibility and provides consolidated alert management across log shipping infrastructure

---

### FAQ 11: Can Log Shipping use SQL authentication versus Windows authentication for shared folder access?

- Log shipping uses SQL Server service account credentials to access network shared folder for copying log backup files
- Whether Windows authentication or SQL authentication is used depends on service account configuration on servers
- For Windows service accounts, ensure account has full control permissions on shared folder in NTFS permissions
- For SQL authentication scenarios, service account still requires appropriate folder-level permissions to function
- Best practice is to use dedicated service account with minimal required permissions for security purposes
- Cannot specify different credentials per log shipping job; the service account handles all file operations uniformly

---

### FAQ 12: How does the database restore state affect secondary database usability?

- If secondary database is left in STANDBY mode, users can run read-only SELECT queries during intervals between restore operations
- If secondary database is left in NORECOVERY mode, it cannot be accessed by users at all, reserved only for log shipping restore operations
- STANDBY mode preserves log sequence number chain while allowing limited read access for reporting and queries
- NORECOVERY mode ensures database is always ready to receive next log restoration without additional preparation steps
- Switching between modes requires manual configuration of secondary database state during initial setup
- STANDBY mode useful for offloading reporting queries to secondary server, while NORECOVERY preferred when secondary is failover target only

---

## SECTION 3: OPERATIONS AND JOBS

### FAQ 13: What happens during the backup job on the primary server?

- Backup job executes at scheduled intervals to create transaction log backup of primary database for shipping
- Backup is written to local server and to configured shared network folder for copy job to pick up files
- Backup job logs completion status and backup file information to msdb database on primary and monitor servers if configured
- Backup job automatically deletes old backup files based on retention policy settings to manage storage space
- Job creates alert history records if previous log backups were not copied or restored within configured threshold time
- Backup job step also truncates transaction log to free up disk space used by completed transactions

---

### FAQ 14: What is the purpose of the copy job and where does it run?

- Copy job runs on each secondary server and transfers transaction log backup files from primary shared folder to secondary local backup folder
- Two-step approach of backup then copy improves performance by reducing network traffic during restore operations on secondary
- Copy job executes on configurable schedule, typically every 15 minutes by default in most environments
- Copy job logs its history to local secondary server msdb and to remote monitor server if configured in setup
- If copy jobs fail, network connectivity or permissions issues are most common causes requiring investigation
- Copy job uses SQL Server service account of secondary server to access network shared folder for file transfers

---

### FAQ 15: What is the restore job and why is a delay configurable?

- Restore job runs on each secondary server to apply copied transaction log backups to secondary database
- User-specified delay between backup and restore operations provides time window to recover from accidental data changes
- Delay is valuable for disaster recovery allowing recovery from accidental deletion before error replicates to secondary
- For example, if developer accidentally deletes data, delay gives time to discover error and retrieve unchanged data from secondary
- Delay measured in minutes allows secondary to intentionally lag behind primary based on business requirements
- Restore job cannot access secondary database while another restore is in progress, preventing concurrent access

---

### FAQ 16: Why does the secondary database become unavailable during log restoration?

- During restore operation, secondary database transitions from available to unavailable as SQL Server applies transaction log backup
- Users cannot query secondary database during active restore operations regardless of configured interval
- Unavailability duration depends on size of transaction log backup and available system resources on secondary server
- If secondary is configured in STANDBY mode, read access resumes after restore completes until next restore job begins
- If secondary is configured in NORECOVERY mode, database remains unavailable both during and after restoration
- Planning queries against STANDBY secondary databases requires knowledge of restore job schedule and duration

---

### FAQ 17: What happens if the backup job fails on the primary server?

- If backup job fails, no transaction log backups are created, halting all log shipping operations downstream immediately
- Backup failure is recorded in backup job history on primary and monitor servers for investigation purposes
- Monitor server raises alert if threshold time for backup operations is exceeded
- Copy and restore jobs on secondary servers receive no new files to process, causing secondary to fall out of sync progressively
- Common backup job failures include insufficient disk space, permissions issues on shared folder, or SQL Server Agent not running
- Must investigate and resolve root cause immediately because secondary database is no longer protecting against primary failure

---

### FAQ 18: What does it mean if copy job completes but restore job falls behind?

- Copy job successfully transfers log files but restore job lags, indicating secondary database is falling progressively out of sync
- Indicates either network delays, secondary server resource constraints, or excessive restore job interval scheduling
- For example, if backups occur every 5 minutes but restores run only every 30 minutes, secondary perpetually lags behind
- Check secondary server disk I/O, CPU, and memory resources during restore job execution to identify bottlenecks
- Verify secondary database restore job schedule and reduce interval if resources permit additional restore operations
- Monitor time delta between last backup time and last restore time to quantify lag and identify degradation trends

---

### FAQ 19: How do you monitor if Log Shipping jobs are running on schedule?

- Use Transaction Log Shipping Status report in SQL Server Management Studio by right-clicking primary server and selecting Reports
- Report displays status of all backup, copy, and restore jobs with timestamps of last execution for each database
- Query log shipping monitoring tables in msdb using sp_help_log_shipping_monitor stored procedure on monitor server
- Execute job history query using sp_help_jobhistory for specific log shipping jobs to review detailed status and errors
- Set up SQL Server Agent alerts configured to notify administrators if any log shipping job fails during execution
- Configure alerts if time threshold between backup and restore operations exceeds defined limits in your configuration
- Check Windows Event Viewer on all servers for SQL Server Agent errors or service failures that stop log shipping jobs

---

### FAQ 20: What is the maximum number of secondary databases in a single Log Shipping configuration?

- SQL Server documentation does not specify hard limit on number of secondary databases per primary database
- Practical limits depend on primary server resources, network bandwidth, and available backup storage capacity
- Each secondary database requires its own copy and restore jobs running on secondary server consuming resources
- Adding more secondary databases increases backup job duration and network traffic consumption proportionally
- Network bandwidth becomes limiting factor when shipping transaction logs to many geographically dispersed secondary servers
- Best practice is to test with actual workload to determine optimal number of secondaries for your environment

---

## SECTION 4: MONITORING AND ALERTS

### FAQ 21: What tables in msdb store Log Shipping configuration and history?

- Log shipping configuration stored in tables: log_shipping_primary_databases, log_shipping_primary_secondary, log_shipping_secondary, log_shipping_secondary_databases
- History tables include log_shipping_monitor_primary, log_shipping_monitor_secondary, log_shipping_monitor_error_detail, and log_shipping_monitor_history_detail
- Copy of monitoring information stored locally on primary and secondary servers even if remote monitor server is configured
- Monitor server maintains master copy of all history and status records for entire configuration
- Can query tables directly using T-SQL to build custom monitoring reports and dashboards for management visibility
- History records retained based on retention settings configured, typically 14 to 30 days in most environments

---

### FAQ 22: How do you enable alerts for Log Shipping job failures?

- Configure alerts using SQL Server Agent by creating jobs that execute sp_check_log_shipping_monitor_alert stored procedure
- This procedure compares current backup, copy, and restore status against defined thresholds to identify issues
- If threshold is exceeded, procedure raises alert with specific error messages identifying failure type
- Can configure separate alerts for backup, copy, and restore job failures with different threshold time settings
- Alerts can send email notifications, run response jobs, or execute custom T-SQL scripts based on configuration
- If no monitor server configured, alert jobs run locally on primary and secondary servers

---

### FAQ 23: What does the Transaction Log Shipping Status report show?

- Displays status of primary database and all associated secondary databases in single consolidated view for comparison
- Shows time of last backup, last copy, and last restore for each secondary database with timestamps
- Includes time gap between operations to highlight any synchronization delays or drift from primary
- Status indicators show whether each job completed successfully or failed during last execution cycle
- Displays which log files were last backed up, copied, and restored on each secondary for troubleshooting
- Includes alert history and monitor information if monitor server is configured for the log shipping setup

---

### FAQ 24: What is the recommended monitoring interval for Log Shipping?

- Configure SQL Server Agent alerts to check log shipping status every 5 to 15 minutes depending on criticality level
- If using monitor server, alert job runs on schedule you define, typically running frequently to catch issues quickly
- Backup job typically runs every 5 to 15 minutes to minimize data loss window and RPO for databases
- Copy job should run more frequently than backup job to prevent log file accumulation in shared folder
- Restore job should run frequently enough that secondary database lags minimally behind primary for acceptable RPO
- High-criticality applications warrant more frequent monitoring and shorter job intervals
- Less critical systems might accept longer intervals and wider synchronization gaps for resource optimization

---

### FAQ 25: How do you measure the lag between primary and secondary databases?

- Time delta calculation performed by subtracting last restore time from last backup time on primary server
- Transaction Log Shipping Status report displays this time difference in report output for operator review
- Query log_shipping_monitor_history_detail table to extract precise timestamps of each operation
- Lag represents unrecovered transactions if failover occurs at that moment, not actual data loss amount
- Lag of 30 minutes means secondary is 30 minutes behind primary in applying transactions from primary
- Shorter lags require more frequent backup and restore job scheduling, which increases resource usage
- Longer lags reduce resource usage but increase potential data loss if primary fails unexpectedly

---

### FAQ 26: What are common causes of copy job failures?

- Network connectivity issues between primary and secondary servers prevent copy job from accessing shared backup folder
- Permission issues occur when secondary server SQL Server service account lacks read permissions on primary shared folder
- Shared folder path changes or unmapped network drives cause copy jobs to fail immediately when path becomes invalid
- Disk space exhaustion on secondary server prevents new log files from being copied successfully
- Firewall rules blocking SMB protocol or other file sharing ports prevent network access to shared folder
- SQL Server Agent service not running on secondary server stops all scheduled copy jobs from executing
- Windows authentication failures or credential issues prevent network access with service account credentials

---

### FAQ 27: What are common causes of restore job failures?

- Insufficient disk space on secondary server prevents new log files from being restored to database
- Corrupted transaction log backup files cannot be restored and must be re-backed up from primary server
- Missing or damaged log files in copy destination prevent restore job from finding required backup files
- Database in use or locked state prevents restore job from acquiring exclusive access to database
- Corrupted secondary database pages or structures cause restore to fail with database integrity errors
- SQL Server service account permissions issues prevent restore job from reading log files or modifying database
- Restore job threshold time exceeded when previous restores take longer than scheduled interval duration

---

### FAQ 28: How do you configure email notifications for Log Shipping alerts?

- Enable Database Mail on SQL Server instance hosting alert job or monitor server for email delivery capability
- Create mail profile and account with SMTP server credentials in Database Mail configuration settings
- Configure SQL Server Agent mail profile by setting it in Agent properties for alert notifications
- Create alerts for log shipping jobs using sp_add_alert stored procedure with notification email operator specified
- Specify operator name and email address in SQL Server Agent Operators configuration for alert delivery
- Test email delivery by sending test message through Database Mail before alerts go to production
- Verify Database Mail and SQL Server Agent notification settings correctly saved across server restarts

---

## SECTION 5: FAILOVER AND DISASTER RECOVERY

### FAQ 29: What is the failover process when the primary server fails?

- Stop application connections and disable primary server network connection to prevent split-brain scenarios
- Verify secondary database has received all available log backups by checking last restored file timestamp
- Perform tail-log backup on primary server if still accessible and move this backup to secondary immediately
- Restore tail-log backup on secondary database with RECOVERY option to bring database online for applications
- Verify data integrity and transactions on secondary database after recovery before directing traffic to it
- Rename secondary server to match primary server name and SQL instance name if applications reference server names in connection strings
- Restore master and msdb databases from primary to secondary to sync logins, jobs, and linked servers

---

### FAQ 30: What is a tail-log backup and why is it critical during failover?

- Tail-log backup captures any committed transactions written to primary database since last transaction log backup
- These transactions might not have been backed up yet if primary server failed suddenly due to hardware or software failure
- Performing tail-log backup done with BACKUP LOG WITH NO_RECOVERY option before primary completely unavailable
- This backup must be copied to secondary server and restored before bringing secondary database online
- Restoring tail-log backup ensures secondary database includes all committed transactions from primary up to failure moment
- Without tail-log backup, you lose all transactions that occurred since last regular backup job completed
- Tail-log backup is final step in minimizing data loss during failover scenarios

---

### FAQ 31: Can you automate failover in Log Shipping or is it always manual?

- Log shipping does not support automatic failover like database mirroring or availability groups in any configuration
- All failover operations are manual and require administrator intervention and decision-making before execution
- If need automatic failover, should evaluate using database mirroring with high-safety mode or AlwaysOn Availability Groups
- Must manually execute failover steps and verify secondary database before redirecting applications to it
- Manual process introduces operational risk because failover decisions require human judgment and verification
- Organizations develop runbooks documenting failover procedure and perform regular failover drills to prepare teams
- Lack of automatic failover is significant limitation of log shipping for mission-critical systems requiring near-zero downtime

---

### FAQ 32: What metadata must be synchronized when failing over to a secondary?

- Server logins and passwords must be synchronized from primary to secondary server for applications to authenticate after failover
- Database users must be mapped to synchronized logins using sp_change_users_login procedure after failover
- SQL Agent jobs that reference the database must be recreated or updated on secondary server
- Linked servers configured on primary must be recreated on secondary with appropriate connection settings
- SSIS packages and Integration Services jobs must be deployed to secondary server for scheduled data loads
- Database-level permissions and role memberships must be verified on secondary database after failover
- Stored procedures referencing server-scoped objects must be recreated if those objects do not exist on secondary

---

### FAQ 33: How do you handle failback to the original primary after recovery?

- Original primary server must be verified for data corruption or hardware issues before accepting it back as primary database
- Restore original primary database with backup from secondary to ensure both databases are synchronized before failback
- Remove log shipping from secondary-turned-primary database using log shipping configuration removal wizard completely
- Reconfigure log shipping from scratch with original primary as primary and recovered server as new secondary database
- Reconfiguration necessary because log shipping configuration is directional and cannot be reversed automatically by system
- Plan failback during maintenance window to avoid application switching during recovery operations and testing
- Validate all failback operations in test environment before attempting in production to prevent issues

---

### FAQ 34: What is the Recovery Point Objective (RPO) in Log Shipping scenarios?

- RPO represents maximum amount of data willing to lose, measured as time since last backup of database
- In log shipping, RPO determined by backup job interval, typically ranging from 5 minutes to 30 minutes
- If backup job runs every 15 minutes, your RPO is 15 minutes in normal circumstances and operations
- System failure can result in loss of data beyond configured RPO if primary fails before scheduled backup completes
- Network or job failures can increase actual RPO beyond configured interval in practice
- Critical systems require shorter backup intervals to achieve smaller RPO values and data protection
- Less critical systems can accept longer intervals and larger RPO values to reduce resource overhead costs

---

### FAQ 35: What is the Recovery Time Objective (RTO) in Log Shipping scenarios?

- RTO represents maximum acceptable downtime from when primary fails until secondary is brought online for production
- Log shipping RTO depends on manual failover procedure duration, typically ranging from 15 minutes to 1 hour
- Additional time required for applications to detect primary failure and initiate failover procedures
- Administrator response time and decision-making add to actual RTO in real incidents and failures
- Testing failover drills reveals actual time required to execute all recovery steps in your environment
- If RTO requirement less than 1 hour, log shipping might not meet requirements and should evaluate automatic failover
- If RTO requirement is several hours or more, log shipping is generally adequate for requirements

---

### FAQ 36: What happens to in-flight transactions if the primary fails?

- Transactions in-flight at moment of primary failure are either fully committed or rolled back based on checkpoint state
- Transactions fully written to transaction log before failure are recovered during secondary database recovery
- Transactions partially written to transaction log when primary fails might be unrecoverable and lost
- Uncommitted transactions are automatically rolled back when database is recovered from backup
- Tail-log backup captures any committed transactions that might not have been backed up by last regular backup
- This is why tail-log backup is critical to minimizing data loss during failure scenarios
- Some data loss is inevitable in log shipping scenarios with asynchronous replication of database changes

---

### FAQ 37: How do you prevent accidental promotion of a secondary database?

- Implement strict change control procedures requiring approval before any failover operation on secondary database
- Document current primary server name and connection details in protected central location accessible to authorized staff
- Use read-only access on STANDBY secondary databases to prevent accidental modifications by users or applications
- Restrict permissions on secondary servers to prevent operators from making configuration changes without authorization
- Implement monitoring alerts that notify administrators immediately if log shipping is stopped or removed unexpectedly
- Create database copy-protected backups or tape storage to enforce recovery point integrity against tampering
- Require multi-person authorization for critical failover decisions to prevent single-point human error

---

### FAQ 38: What is the role of the monitor server during disaster recovery?

- Monitor server maintains independent history records of all log shipping operations separate from primary or secondary
- During primary server failure, monitor server provides audit trail evidence of last successful backup and restore times
- Helps determine whether data loss occurred and exact extent of gap between last backup and failure
- If both primary and secondary fail, monitor server records can guide recovery decisions and root cause analysis
- Monitor server continues recording status even if primary or secondary becomes unavailable or fails
- Post-incident analysis relies heavily on detailed monitor server records to understand events and sequence
- Monitor server should be located on separate physical server from both primary and secondary for high availability

---

### FAQ 39: What are the considerations for geo-redundant Log Shipping across data centers?

- Network latency and bandwidth between geographically separated data centers affects log file copy performance metrics
- Plan for higher backup job intervals if inter-site latency exceeds 100 milliseconds for acceptable response times
- Implement compression on network-transferred log files to reduce bandwidth utilization across WAN connections
- Configure network monitoring to track inter-site throughput and identify congestion patterns during peak hours
- Consider dedicated network connections or WAN optimization appliances for geographically dispersed log shipping
- Separate secondary servers in different data centers increase RPO and RTO due to distance and latency
- Plan for longer failover recovery times because geographic distance increases manual intervention time requirements

---

### FAQ 40: How do you test Log Shipping failover without affecting production?

- Create complete copy of production environment including SQL Server instances and network configuration for testing
- Restore production database backups to test environment to replicate production data state
- Configure test log shipping to replicate production configuration exactly including job schedules and monitoring
- Perform test failover by following documented failover procedure step-by-step in test environment
- Validate all transactions on secondary database are present and uncorrupted after failover completion
- Test application connectivity to failed-over database with actual application instances in test environment
- Measure actual time required to complete all failover steps to validate against RTO requirements

---

## SECTION 6: SECURITY AND BEST PRACTICES

### FAQ 41: How do you secure the network shared folder used for log file transfer?

- Use NTFS permissions to restrict access only to SQL Server service accounts from primary and secondary servers
- Enable share-level permissions to restrict access at network share level to authorized servers and accounts only
- Place shared folder on dedicated file server rather than on production database servers if possible
- Use UNC network path with explicit server name rather than mapped drives which are fragile and problematic
- Implement encryption on network connection if log files contain sensitive data using IPsec or VPN tunnels
- Monitor access to shared folder and log all file operations for compliance auditing and security purposes
- Regularly review permissions and remove access for departed staff or decommissioned servers from environment

---

### FAQ 42: What is the recommended backup strategy for Log Shipping configurations?

- Back up primary database on separate schedule independent of log shipping operations to maintain full backup recovery points
- Maintain full backups at least weekly, with more frequent backups for critical systems requiring higher data protection
- Retain full backups for duration of legal data retention requirements and compliance obligations
- Back up monitor server msdb database regularly to preserve historical records of operations
- Back up secondary database periodically if used for reporting to enable point-in-time recovery for reports
- Store backups in offsite location separate from primary and secondary data centers for disaster protection
- Implement backup encryption and compression to protect sensitive data and reduce storage costs substantially

---

### FAQ 43: How do you maintain SQL Server security across Log Shipping configuration?

- Apply same security policies and patches to both primary and secondary servers consistently and uniformly
- Maintain matching Windows update levels and SQL Server service pack versions between servers for compatibility
- Configure same SQL Server security settings on both servers including authentication mode and encryption requirements
- Synchronize server-level logins and roles from primary to secondary using documented procedures and scripts
- Implement database-level permissions consistently across primary and secondary databases for users and roles
- Monitor both servers for security events and vulnerabilities using centralized security tools and SIEM solutions
- Restrict administrative access to log shipping infrastructure to authorized DBA personnel only with audit logging

---

### FAQ 44: What happens to SQL Server logins and permissions during Log Shipping setup?

- SQL Server logins stored in master database are not automatically shipped to secondary server during setup
- Must manually create matching logins on secondary server with identical passwords for authentication to work
- Database users in secondary database become orphaned if corresponding server logins do not exist on secondary
- Database permissions granted to roles within secondary database are restored when database is restored
- After failover, use sp_change_users_login to map orphaned users to existing logins on secondary server
- Can script logins from primary server using SQL Server Management Studio scripting features and automation
- Automate login synchronization by creating job that periodically compares and synchronizes logins between servers

---

### FAQ 45: How do you protect against accidental deletion of data with Log Shipping?

- Configure restore job with delay period longer than typical user response time to accidental deletions
- For example, 2-hour delay gives users time to discover accidental deletion and restore from secondary
- Secondary database in STANDBY mode remains accessible for queries during delay period for verification
- Users can query secondary database to verify whether critical data exists in delayed state
- After discovering accidental deletion, stop restore job and restore secondary to production temporarily
- Create backup of secondary database at point before accidental deletion was applied
- Restore backup to separate database to recover deleted data and merge back to production

---

### FAQ 46: What are the compliance and audit considerations for Log Shipping?

- Maintain detailed audit logs of all changes to log shipping configuration on all servers for compliance
- Track all administrator actions that modify log shipping through SQL Agent jobs and T-SQL scripts
- Document all failover events including timestamp, reason, and person authorizing the failover for records
- Preserve monitor server history records for duration of compliance retention requirements
- Implement read-only access restrictions to shared log file folder to prevent tampering with backups
- Generate regular compliance reports from monitor server showing backup, copy, and restore activity
- Include log shipping configuration in disaster recovery plan documentation for audit and compliance purposes

---

### FAQ 47: How do you document Log Shipping configuration for knowledge transfer?

- Create detailed runbook documenting each step of log shipping configuration procedure with screenshots
- Include network diagrams showing shared folder locations and server connections and dependencies
- Document backup job schedule, copy job schedule, and restore job schedule with time intervals and durations
- List all SQL Server logins and permissions that must be synchronized after failover to new servers
- Include complete failover procedure documentation with step-by-step instructions for operations teams
- Document location and access procedures for off-site backup storage for disaster recovery scenarios
- Create quick reference card for common troubleshooting issues and resolution steps for daily operations

---

### FAQ 48: How do you handle SQL Server Agent job failures in Log Shipping?

- Verify SQL Server Agent is running on affected server and set to start automatically on server restart
- Check SQL Server Agent startup account has required permissions to access network shared folder
- Review job history for detailed error messages indicating specific cause of failure
- Check Windows Event Viewer and SQL Server error logs for related error messages and warnings
- Verify disk space available on affected server for new log files and backup operations
- Test network connectivity and shared folder access from affected server using net use command
- Stop and restart SQL Server Agent service if configuration changes were made to take effect

---

### FAQ 49: What should you include in a Log Shipping disaster recovery test plan?

- Schedule quarterly or semi-annual failover tests to validate procedures and maintain operator readiness and skills
- Perform tests in non-production environment if possible to avoid impacting production systems and applications
- Include representatives from database, application, and business teams in test planning and execution
- Document test procedures, expected outcomes, and roles for each participant in test plan documentation
- Measure actual failover time from failure detection through secondary database online and operational
- Verify application connectivity and functionality against failed-over secondary database with real applications
- Test login authentication and permissions to identify orphaned user issues before they occur in production

---

### FAQ 50: How do you recover from corruption in the primary database affecting Log Shipping?

- Stop backup job immediately to prevent corrupted transactions from being shipped to secondary database
- Verify secondary database does not contain corruption by checking timestamps and comparing data
- Take clean backup from secondary database if corruption has not yet been replicated to it
- Restore primary database from last known good backup before corruption occurred in backup history
- Perform database consistency check DBCC CHECKDB on restored primary to verify integrity after restoration
- Perform fresh full backup of recovered primary database to establish new baseline for log shipping
- Remove and reconfigure log shipping to establish new baseline from recovered primary database

---

## SECTION 7: PERFORMANCE TUNING AND OPTIMIZATION

### FAQ 51: How do you optimize Log Shipping performance for large databases?

- Schedule backup jobs during off-peak hours when primary server load is lower for less impact
- Reduce backup job interval if resources permit and RPO requirement supports more frequent backups
- Implement backup compression to reduce network bandwidth and copy job duration significantly
- Offload backup storage to dedicated file server or storage appliance with fast network connectivity
- Configure copy job to run more frequently than backup job to prevent log file accumulation in folder
- Allocate additional CPU and memory to secondary servers if restore job duration is excessive
- Implement disk performance optimization on secondary servers using SSD storage for log file staging

---

### FAQ 52: How do you reduce network bandwidth consumption in Log Shipping?

- Enable backup compression on backup job to reduce backup file size by 50-70 percent
- Compress log files using GZIP or similar compression before copying to secondary servers over network
- Adjust backup job interval to reduce number of log files shipped per day across network
- Implement differential backup strategy on primary for smaller change sets and reduced file sizes
- Use bandwidth throttling or network QoS to prioritize log shipping copy operations in network queue
- Schedule log shipping copy jobs during off-peak network usage windows for better performance
- Implement local backup staging on secondary servers to minimize repeated network transfers

---

### FAQ 53: How do you handle slow or high-latency network connections?

- Increase backup job interval to reduce frequency of network transfers across high-latency links
- Implement backup compression to reduce volume of data transferred across constrained network
- Monitor and measure actual network latency between primary and secondary servers using diagnostic tools
- Consider adding intermediate secondary servers closer to primary to reduce hop count and latency
- Implement network optimization appliances for WAN acceleration if bandwidth is constrained
- Configure explicit retry logic in copy jobs to handle transient network failures gracefully
- Implement dedicated network connections or private circuits between primary and secondary servers

---

### FAQ 54: How do you minimize impact of Log Shipping backup jobs on primary database performance?

- Schedule backup jobs during off-peak hours when application query load is lowest on primary
- Set backup job priority to low in Windows Task Scheduler to limit resource consumption by backups
- Configure I/O throttling to limit disk I/O consumed by backup operations during business hours
- Use backup compression to reduce CPU utilization during backup operations on primary server
- Implement incremental or differential backups to reduce backup duration when full backups not required
- Monitor primary server CPU, memory, and disk I/O during backup job execution to identify impact
- Adjust backup job interval to allow adequate time between jobs for recovery from previous backup

---

### FAQ 55: How do you optimize restore job performance on secondary servers?

- Configure restore job to run more frequently to avoid large backup batches requiring long processing times
- Allocate dedicated SSD storage for transaction log staging to improve I/O performance during restores
- Enable backup compression on primary to reduce size of files requiring restoration on secondary
- Monitor secondary server CPU, memory, and disk I/O during restore operations to identify bottlenecks
- Adjust restore job interval to allow adequate time for large transactions to complete successfully
- Use BULK_LOGGED recovery model on secondary if performance is critical for restore operations
- Implement parallel restore operations if secondary server capacity permits multiple concurrent restores

---

### FAQ 56: What should you monitor to identify Log Shipping performance degradation?

- Monitor time delta between last backup and last restore to detect increasing lag patterns
- Track backup job duration to identify whether jobs taking progressively longer over time
- Monitor copy job duration to detect network or file system performance issues emerging
- Monitor restore job duration to identify CPU, memory, or disk bottlenecks on secondary server
- Monitor disk space consumption on shared folder to predict capacity issues before they occur
- Track SQL Server error logs for I/O errors or other performance-related messages
- Monitor network bandwidth consumption to ensure log shipping does not saturate network links

---

### FAQ 57: How do you handle Job timeout errors in Log Shipping?

- Increase SQL Server Agent job step timeout settings for backup, copy, and restore jobs
- Configure longer timeout values for jobs operating over high-latency network connections
- Monitor job execution time under production load to determine appropriate timeout values
- Implement retry logic in job steps to handle transient failures gracefully without immediate failure
- Break large jobs into multiple steps with individual timeout configurations if supported by setup
- Review timeout logs to identify jobs close to timeout and risk failure
- Communicate timeout changes to operations team to ensure understanding of new limits

---

### FAQ 58: What is the impact of database size on Log Shipping backup performance?

- Larger databases take proportionally longer to backup, increasing backup job duration
- Very large databases might exceed configured backup job interval causing overlap of backup operations
- Consider splitting very large databases into smaller logical units if feasible for faster backups
- Implement database partitioning strategies to reduce backup scope per database and improve speed
- Use differential or incremental backups for large databases to reduce backup volume and duration
- Implement backup compression more aggressively on large databases to maximize compression ratios
- Schedule backups during longest available maintenance window for large databases

---

### FAQ 59: How do you configure optimal backup and restore job schedules?

- Start with baseline of every 15 minutes for backup jobs in most environments as standard
- Align copy job schedule to run every 5 to 10 minutes to prevent log file accumulation in folder
- Configure restore job to run at least as frequently as backup jobs for acceptable synchronization
- Stagger job schedules to avoid resource conflicts between backup, copy, and restore on same server
- Plan longer intervals during high-load periods and shorter intervals during low-load periods
- Test schedule configurations under actual production load before deployment to production
- Monitor actual job execution times and adjust schedules if jobs consistently run long

---

### FAQ 60: How do you measure database RPO in practice across Log Shipping configuration?

- Calculate maximum time between last backup and failover by subtracting last backup timestamp from failure
- Measure actual backup job interval by calculating average time between successive backup completions
- Monitor restore job lag by subtracting restore timestamp from backup timestamp to measure actual lag
- Document RPO during normal operations and during high-load periods to understand variations
- Perform calculations monthly to identify trends or degradation in RPO over time
- Create RPO trending reports and alert operators if RPO exceeds defined thresholds
- Test RPO by simulating failures and measuring recovery time from backups

---

## SECTION 8: TROUBLESHOOTING COMMON ISSUES

### FAQ 61: What does "Log shipping database is out of sync" message mean?

- Secondary database is falling further behind primary because restore jobs not keeping up with backup jobs
- Time gap between last backup and last restore exceeding your configured threshold limit
- Either backup jobs running too frequently or restore jobs running too slowly for current workload
- Secondary server resources such as CPU, memory, or disk I/O might be insufficient for restore
- Network latency might be causing delays in copy job execution and file transfers
- Backup job creating larger transactions than restore job can process in available time window
- Must either reduce backup job frequency or increase restore job frequency to resolve issue

---

### FAQ 62: How do you resolve permissions errors when copy jobs fail?

- Verify secondary server SQL Server service account has read permissions on primary shared backup folder
- Verify primary server SQL Server service account has write permissions on shared folder for backups
- Run icacls command on shared folder to display current NTFS permissions for review
- Grant explicit permissions to SQL Server service accounts using GUI or command line tools
- Verify network share-level permissions allow access for affected service accounts and servers
- Check if service account passwords changed recently, causing authentication failures
- Test folder access from secondary server command line using net use with service account credentials

---

### FAQ 63: How do you recover from missing transaction log backup files?

- Identify which log backup files are missing from shared folder by comparing backup history
- Determine time range of missing backups and identify gap in LSN chain for recovery
- Recopy missing log backup files from primary server to secondary if still available
- If missing files deleted from primary server, reconfigure log shipping with fresh full backup
- Stop restore job to prevent it from failing when missing files are encountered
- Restore missing backup files by copying from backup history location if stored elsewhere
- Update log shipping configuration to start from next available log backup file

---

### FAQ 64: How do you troubleshoot if the secondary database restore remains in NORECOVERY state?

- Restore job has not completed successfully enough times to bring database online
- Check restore job history to identify errors preventing successful restore operations
- Verify restore job schedule is configured and enabled for secondary database setup
- Check secondary server SQL Server Agent running and has permissions to execute restore jobs
- Manually execute restore job to test whether it runs successfully on demand for verification
- Check disk space on secondary server to ensure sufficient capacity for restored log files
- Verify secondary database exists and is accessible by SQL Server service account

---

### FAQ 65: How do you handle "Cannot open backup device" errors during restore jobs?

- Verify network path to backup folder is accessible from secondary server correctly
- Check UNC path is correct and matches configuration in log shipping setup procedures
- Verify secondary server SQL Server service account has read permissions on backup folder
- Test network connectivity to primary server using ping and network trace commands
- Verify network shared folder exists and contains expected backup files for restoration
- Check for network drive mapping issues if using mapped drives instead of UNC paths
- Verify firewall rules allow SMB traffic between primary and secondary servers

---

### FAQ 66: How do you resolve "Backup and Restore LSN mismatch" errors?

- Backup LSN does not match restore LSN due to missing or corrupted log files in sequence
- Verify all transaction log backups copied successfully from primary to secondary
- Check for gaps in backup file sequence numbers indicating missing backups
- Compare backup history on primary with files present in secondary folder
- Remove corrupted log backup files preventing restore chain from continuing
- Restore missing log backups from off-site backup storage if available for recovery
- If gap cannot be closed, remove and reconfigure log shipping with fresh full backup

---

### FAQ 67: How do you fix "Insufficient disk space" errors on secondary server?

- Check available disk space on secondary server using disk management utilities
- Identify which drives running low on capacity and storage space
- Delete old or unnecessary files from drives to free up space immediately
- Move secondary database data or log files to different drive with more capacity
- Move log backup staging folder to drive with sufficient space for operations
- Compress old backup files if retained locally on secondary server for storage optimization
- Implement disk cleanup policies to automatically delete old backup files after retention expires

---

### FAQ 68: How do you troubleshoot "SQL Server Agent is not running" errors?

- Verify SQL Server Agent service is enabled and set to start automatically on server restart
- Check Windows Services to confirm SQL Server Agent service is in Running state currently
- Review Windows Event Viewer for service startup failures or dependency issues preventing start
- Check SQL Server Agent startup account credentials are correct and account not locked
- Restart SQL Server Agent service to clear any stalled job processes blocking execution
- Verify SQL Server Agent has required permissions to access shared folders and databases
- Check SQL Server error logs for messages related to Agent startup or job execution

---

### FAQ 69: How do you resolve "Connection timeout" errors between primary and secondary servers?

- Check network connectivity between servers using ping and tracert commands for connectivity
- Verify firewall rules allow port 445 SMB traffic between primary and secondary servers
- Check network device configurations for firewall or routing blocking log shipping traffic
- Monitor network latency to secondary server to identify high-latency connections
- Increase connection timeout settings in log shipping jobs if network latency identified
- Verify DNS resolution working correctly for server names used in network paths
- Check for network interface failures or degradation on primary or secondary servers

---

### FAQ 70: How do you handle monitor server connectivity issues?

- Verify monitor server network connection and status in log shipping configuration
- Test network connectivity from primary and secondary servers to monitor server
- Verify monitor server instance accessible by attempting connection from SQL Server Management Studio
- Check monitor server SQL Server Agent running and alert jobs enabled for operations
- Review monitor server security to ensure primary and secondary servers have required permissions
- Check monitor server disk space and verify msdb database has adequate free space
- Reconfigure remote monitor connectivity settings if monitor server moved to different network

---

## SECTION 9: ADVANCED SCENARIOS AND EDGE CASES

### FAQ 71: Can you use Log Shipping with Always-On Availability Groups as a secondary failover target?

- Log shipping and AlwaysOn are separate DR technologies not designed to work together
- Should choose one technology as primary DR solution rather than combining them together
- Cannot configure log shipping on availability group replica or recipient database
- If need redundancy, implement either log shipping or availability groups, but not both
- Using both introduces unnecessary complexity and configuration challenges for operations
- Organizations sometimes use log shipping for cross-region DR while using availability groups for local HA
- In such scenarios, availability group primary becomes log shipping source database for backup

---

### FAQ 72: How do you use Log Shipping with database snapshots for testing?

- Create database snapshot of primary database to capture point-in-time copy for testing
- Use snapshot for testing or report queries without affecting primary database or transactions
- Database snapshots are read-only and cannot be used as log shipping secondary databases
- Log shipping replicates primary database including snapshots associated with it automatically
- Snapshots consume storage space but do not affect log shipping operations or backup frequency
- Can create snapshots of secondary database for testing purposes and validation
- Snapshots useful for testing data changes or backup recovery without affecting production

---

### FAQ 73: How do you implement Log Shipping across SQL Server versions or editions?

- Log shipping backward compatible with older SQL Server versions on primary server
- For example, SQL Server 2016 primary can ship to SQL Server 2019 secondary successfully
- Newer versions cannot ship to older versions due to compatibility issues with log format
- Must match or upgrade secondary SQL Server version when upgrading primary
- Plan upgrade procedures carefully to avoid breaking log shipping chain during upgrades
- Test version mismatches in non-production before upgrading production systems
- Document version requirements in configuration and disaster recovery procedures

---

### FAQ 74: How do you handle Log Shipping when database names are different on primary and secondary?

- Log shipping allows secondary database to have different name than primary database
- Specify secondary database name during log shipping configuration setup in wizard
- Application connection strings must be updated to use correct secondary database name
- During failover, either rename secondary database to match primary or update connection strings
- Renaming secondary database after failover requires dropping log shipping configuration first
- Different database names complicate operational procedures so minimize this practice if possible
- Document primary and secondary database names and maintain mapping for reference

---

### FAQ 75: How do you implement read-only reporting on the secondary database?

- Configure secondary database in STANDBY mode to allow read-only access between restore operations
- Users can run SELECT queries against secondary database while it is in STANDBY state
- Write operations and DDL statements not allowed on STANDBY secondary databases
- Restore operations briefly bring database offline, interrupting report queries temporarily
- Plan report schedules to avoid periods when restores running on secondary for continuous access
- Use database snapshots of secondary database to provide consistent report access
- Implement snapshot-based reporting to avoid conflicts with restore job timing

---

### FAQ 76: How do you configure Log Shipping for standby mode database mirroring failover?

- Log shipping and database mirroring are separate DR technologies not meant to combine
- Combining technologies introduces complexity and potential conflicts in operations
- Must choose either log shipping or database mirroring as primary DR solution
- Database mirroring uses different failover mechanisms than log shipping
- Attempting use both simultaneously creates operational confusion and complicates failover
- If need both features, evaluate AlwaysOn Availability Groups for comprehensive redundancy
- Document decision to use log shipping or database mirroring in DR plan

---

### FAQ 77: How do you use Log Shipping with third-party backup software?

- Third-party backup software can backup log shipping backup folder contents
- Transaction log backups created by log shipping can be managed by third-party backup tools
- Ensure third-party backup software does not interfere with log shipping operations
- Coordinate scheduling between log shipping and third-party backup software to avoid conflicts
- Verify third-party software supports open file handles for active backup files during operations
- Use native SQL Server backup compression before third-party compression for optimization
- Implement exclusion rules if third-party software backs up shared log backup folder

---

### FAQ 78: What is the process for renaming servers in a Log Shipping configuration?

- Log shipping configuration stores server names and database names in multiple locations
- Renaming server breaks these references and requires reconfiguration of log shipping
- Server names stored in msdb tables and used by SQL Agent jobs for connectivity
- After renaming servers, remove existing log shipping configuration completely first
- Update DNS or hosts file entries if applications use server names for connectivity
- Reconfigure log shipping from scratch using new server names
- Test connectivity from renamed servers to all other servers in configuration

---

### FAQ 79: How do you implement Log Shipping in contained databases?

- Contained databases store security information within database rather than master database
- Log shipping supports contained databases on secondary with password information
- User passwords included in contained database backup and restored to secondary
- Contained database users do not require login creation on secondary server
- This simplifies metadata synchronization requirements compared to regular databases significantly
- Set containment type to PARTIAL on database before enabling log shipping
- Verify contained database settings identical on primary and secondary databases

---

### FAQ 80: How do you use Log Shipping for migration purposes rather than DR?

- Log shipping can migrate databases with minimal downtime by maintaining synchronized copy
- Enable log shipping from source to target server at migration start
- Allow log shipping to run for duration of planned downtime window
- Stop applications from writing to source database before final cutover
- Perform tail-log backup on source database to capture final transactions
- Restore tail-log backup on target database to capture final transactions
- Redirect applications to connect to target database after verification

---

## SECTION 10: DISASTER RECOVERY SCENARIOS

### SCENARIO 1: Primary Server Complete Hardware Failure

- Primary server experiences catastrophic hardware failure including motherboard or all storage failure
- All log backups in transit and pending restores are lost in the failure
- Last backup on secondary is recovery point for data restoration and continuity
- Execute failover immediately using documented procedures with administrator authorization
- Restore tail-log backup from monitor server if available for capturing final transactions
- Bring secondary database online with RECOVERY option for production use
- Verify data integrity by running DBCC CHECKDB on recovered database before releasing
- Redirect applications to secondary server instance for business continuity
- Determine RPO time of last known good backup and document data loss
- Initiate hardware replacement and recovery procedures for failed primary

---

### SCENARIO 2: Primary Database Corruption Discovered

- Data corruption discovered in primary database, possibly already replicated to secondary
- Immediately stop backup job to prevent shipping corrupted data further to secondary
- Verify secondary database corruption timestamp and extent of data affected
- If corruption existed before secondary's last restore, secondary is also affected
- Use secondary database to restore to point before corruption occurred
- Restore primary database from last clean backup before corruption detected
- Perform DBCC CHECKDB to verify database integrity after restoration
- Take new full backup and reconfigure log shipping to resume operations
- Investigate root cause of corruption such as storage issues or application bugs
- Document incident and implement preventive controls for future protection

---

### SCENARIO 3: Accidental Data Deletion on Primary Database

- User accidentally deletes critical customer data from primary database
- Deletion is committed and backed up into log files during normal operations
- Secondary database contains unmodified data if restore job not yet applied deletion
- Stop log shipping restore job to freeze secondary at point before deletion
- Users query secondary database to verify deleted data exists for recovery
- Create backup of secondary database at this point before applying deletion
- Restore secondary backup to separate database for data recovery and verification
- Copy deleted data from recovery database back to primary using insert statements
- Verify data integrity before resuming normal operations and log shipping
- Implement preventive controls such as table-level backups or soft-delete procedures

---

### SCENARIO 4: Network Connectivity Loss Between Primary and Secondary

- Network failure prevents log file copying from primary to secondary server
- Copy jobs fail immediately and alert operators of failure conditions
- Restore jobs on secondary have no new files to process but continue running
- Secondary database falls progressively out of sync with primary
- Monitor time delta and begin manual recovery planning for failover readiness
- Investigate network issues and restore connectivity as quickly as possible
- Once network restored, copy job catches up with backlog of log files
- Allow time for restore job to catch up to current state for synchronization
- Verify secondary database is synchronized before resuming normal operations
- Review network redundancy architecture to prevent single points of failure

---

### SCENARIO 5: Secondary Server Disk Full Error

- Secondary server runs out of disk space during restore job execution
- Restore operation fails due to insufficient space for log file restoration
- Secondary database remains at previous restore point for recovery
- New log files continue being copied but cannot be restored to secondary
- Secondary falls progressively out of sync with primary as backlog accumulates
- Emergency response: stop copy job to stop accumulating files on full disk
- Free up disk space by deleting old log files or other non-essential data
- Increase disk capacity or move database files to different drive with space
- Resume copy and restore jobs after space is available for operations
- Verify secondary synchronization restored to acceptable levels

---

### SCENARIO 6: SQL Server Agent Service Crashes

- SQL Server Agent service stops unexpectedly on primary or secondary server
- Backup and copy jobs not executed while service is stopped
- Restore jobs continue running against stale files if service failure on secondary
- Secondary falls out of sync progressively as lag increases
- Monitor alerts notify operators immediately if alerts are configured
- Restart SQL Server Agent service on affected server for recovery
- Verify jobs resume execution successfully after service restart
- Allow backlog of jobs to catch up for synchronization
- Verify secondary database resynchronization is complete
- Implement service restart automation if possible for faster recovery

---

### SCENARIO 7: Share Permissions Changed Unexpectedly

- Permissions on shared log backup folder changed or removed without notice
- Copy job immediately fails because secondary server SQL Server service account cannot access folder
- Alert operators receive failure notification from SQL Server Agent
- Restore job continues running against existing files
- Secondary database begins falling behind primary as gap increases
- Investigate permission changes and restore correct permissions immediately
- Verify secondary server service account has read access to folder
- Restart copy job to resume file transfer to secondary
- Allow secondary to catch up on restored files for synchronization
- Implement monitoring of folder permission changes for quick detection

---

### SCENARIO 8: Primary and Secondary Databases Become Inconsistent

- Primary and secondary databases contain different data beyond expected replication lag
- Might occur due to direct modifications to secondary database
- Log shipping cannot repair this inconsistency automatically
- Verify extent of inconsistency by comparing row counts and checksums
- Restore secondary database from known good point in time
- Reconfigure log shipping to resume synchronization from restored point
- Investigate how inconsistency occurred to prevent recurrence
- Implement read-only access restrictions on secondary if modifications occurred
- Document incident and implement preventive controls

---

### SCENARIO 9: Monitor Server Becomes Unavailable

- Monitor server fails or loses network connectivity to primary and secondary servers
- Backup, copy, and restore jobs continue executing but history not centralized
- Alert notifications cannot be sent from monitor server
- Local job history continues recorded on primary and secondary servers
- Operators must monitor jobs using local reports and history manually
- Restore monitor server or replace with new monitor server instance
- Reconfigure log shipping to update monitor server reference
- Allow time for history to synchronize to new monitor server
- Resume centralized monitoring and alert notifications
- Test monitor server recovery procedures periodically

---

### SCENARIO 10: All Backups Older Than Retention Period Deleted Prematurely

- Backup job configured to delete backups older than 7 days deletes critical backups
- Secondary server cannot restore files that no longer exist
- Restore job fails on missing log files in sequence
- LSN chain broken and secondary database orphaned for recovery
- Remove and reconfigure log shipping with fresh full backup of primary
- Adjust backup retention policies to longer period to prevent similar issues
- Verify backup retention logic correct before production deployment
- Restore from off-site backup if available for recovery
- Document incident and implement validation procedures for retention policies

---

### SCENARIO 11: Primary Server Network Interface Failure

- Primary server network card fails unexpectedly during normal operations
- Primary database continues running but cannot write log backups to shared folder
- Backup job fails due to network connectivity loss
- Secondary server cannot access shared folder to copy log files
- Applications on primary continue running but log shipping non-functional
- Hardware replacement or network failover brings primary back online
- Reconfigure network settings and verify connectivity to shared folder
- Backup and copy jobs resume after network restored
- Resume normal log shipping operations for database protection
- Implement network redundancy to prevent single point of failure

---

### SCENARIO 12: Application Mistakenly Connects to Secondary Database

- Application connection string contains secondary database server by mistake
- Network routing failure causes connections to route to secondary
- Users might modify data on what they think is primary
- Secondary database read-only in STANDBY mode so modifications rejected
- If secondary in NORECOVERY mode, connections rejected entirely
- Investigate application connection string and routing configuration
- Correct application configuration to point to primary server
- Verify no data modifications occurred on secondary
- Resume normal operations after verification
- Implement connection string validation checks for prevention

---

### SCENARIO 13: Backup File Corruption in Transit

- Backup file corrupted while being copied from primary to secondary server
- File checksum validation detects corruption
- Restore job fails when attempting to restore corrupted file
- Secondary database cannot advance beyond previous restore point
- Identify corrupted file and delete it from destination folder
- Recopy backup file from primary server to secondary
- Verify file integrity after copy completes successfully
- Resume restore job to process valid backup file
- Implement network checksum validation if possible
- Monitor file transfer processes for similar issues

---

### SCENARIO 14: Recovery Model Unexpectedly Changed

- Database recovery model changed from FULL to SIMPLE on primary
- Transaction log backups stop being created with SIMPLE mode truncating logs
- Secondary database receives no new log backups for restoration
- Restore jobs run on stale data and secondary falls behind
- Change recovery model back to FULL immediately
- Start new transaction log backup to resume log shipping
- Resume backup and copy jobs for operations
- Verify secondary synchronization restored successfully
- Implement monitoring to alert if recovery model changes
- Restrict permissions on database properties to prevent changes

---

### SCENARIO 15: Dramatic Increase in Transaction Volume

- Application transaction volume increases dramatically from batch processes
- Log backup file sizes grow significantly requiring more storage
- Copy and restore job duration increase beyond configured intervals
- Secondary falls progressively behind primary
- Monitor log shipping job execution times and adjust schedules
- Increase backup frequency if secondary server resources permit
- Implement compression more aggressively to reduce file sizes
- Analyze transaction volume increase to understand root cause
- Optimize application queries if excessive logging is issue
- Increase secondary server resources if bottleneck is restore

---

### SCENARIO 16: Both Primary and Secondary Server Fail Simultaneously

- Catastrophic failure affects both primary and secondary simultaneously
- Data center outage, power failure, or natural disaster disables both
- Both databases are unavailable for operations and access
- Off-site backups and separate storage only recovery option
- Assess backup availability and recovery point for restoration
- Restore from off-site backups to tertiary server for continuity
- Verify data integrity and completeness of restored database
- Redirect applications to tertiary server after validation
- Document recovery time and data loss for analysis
- Implement recovery procedures for this scenario in DR plan

---

### SCENARIO 17: Restore Job Continuously Exceeds Maximum Duration

- Restore job consistently takes longer than configured maximum time
- Job terminates before completion on secondary server
- Secondary database remains at previous restore point
- Secondary progressively falls behind primary as lag increases
- Investigate secondary server CPU, memory, and disk I/O utilization
- Increase secondary server resources if bottlenecks identified
- Review secondary database workload and reduce reporting load
- Optimize database indexes on secondary if query performance degrading
- Increase restore job timeout to accommodate longer restore times
- Adjust backup frequency if secondary cannot keep pace

---

### SCENARIO 18: Logins Cannot Be Synchronized Across Primary and Secondary

- Service account password changed on primary but not updated on secondary
- Log shipping jobs fail with authentication errors
- Shared folder access and database connections become impossible
- Update service account password on all servers consistently
- Ensure password changes coordinated across all servers in configuration
- Implement service account password management procedures
- Use managed service accounts if available to reduce manual management
- Verify connectivity after password changes
- Restart SQL Server Agent service on all servers after password changes

---

### SCENARIO 19: Monitor Server Alert Thresholds Are Exceeded Continuously

- Alerts firing constantly because job execution times exceed thresholds
- Alert fatigue develops and operators ignore alerts
- Critical failures might be missed among noise
- Recalibrate alert thresholds based on actual baseline execution times
- Set thresholds at 110-120 percent of normal time rather than fixed values
- Implement alert aggregation to reduce noise from non-critical alerts
- Review baseline metrics monthly and adjust thresholds accordingly
- Test alert configuration to ensure only significant issues trigger
- Document threshold rationale for future reference

---

### SCENARIO 20: Failback to Recovered Primary is Complex Due to Configuration Changes

- Primary server recovered and repaired but requires significant reconfiguration
- Database file locations changed or SQL Server service account changed
- Attempting failback using old log shipping configuration causes failures
- Reconfigure log shipping from scratch rather than reusing old configuration
- Test new configuration thoroughly before production failback
- Verify metadata synchronization logins permissions and jobs
- Run validation checks to confirm primary and secondary consistency
- Execute gradual failback by redirecting subset of users first
- Document all configuration changes made during recovery
- Maintain updated documentation for future reference

---

## CONCLUSION

Log shipping remains viable and cost-effective disaster recovery solution for organizations requiring database protection without automatic failover. 
Success depends on proper configuration, regular monitoring, and periodic testing of failover procedures. 
Organizations must balance RPO and RTO requirements against resource costs and operational complexity. 
Regular review of log shipping health metrics and incident response procedures ensures readiness. 
Combining log shipping with comprehensive backup strategies and off-site storage provides robust protection. 
Planning for realistic disaster scenarios and testing recovery procedures builds confidence. 
Proper documentation and operator training enable rapid response when failures occur. 
Periodic evaluation ensures log shipping remains best choice for your organization.

---

END OF FAQ DOCUMENT

Total Questions Covered: 100 FAQs + 20 Disaster Recovery Scenarios

Last Updated: July 2026

Source: Microsoft Official SQL Server Documentation
