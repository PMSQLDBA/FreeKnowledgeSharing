SQL SERVER DATABASE MIRRORING REAL-TIME SCENARIO 100 FAQS 

================================================================================

IMPORTANT NOTES:
- All FAQs based on Microsoft Official Documentation and verified sources
- Database Mirroring marked deprecated but fully supported in SQL Server 2019-2022
- Microsoft recommends Always On Availability Groups for new implementations
- Information accurate as of SQL Server 2022 and SQL Server 2025
- Always test procedures in non-production environment before production use
- For latest updates, refer to https://learn.microsoft.com/en-us/sql/database-engine/database-mirroring/

================================================================================

FAQ 1: WHAT IS DATABASE MIRRORING IN SQL SERVER?

Q: What is SQL Server Database Mirroring?

A:
1. Database mirroring is a high availability solution that maintains a mirror or standby copy of a SQL Server database on a separate server instance
2. Ensures two identical copies of data exist at all times across principal and mirror servers
3. Provides complete data redundancy and protects against hardware, network, or SQL Server failures
4. Automatically replicates every insert, update, and delete operation from principal to mirror database
5. Operates on a per-database basis, not at the SQL Server instance level
6. Requires separate server instances hosted on separate physical systems as per Microsoft recommendations

================================================================================

FAQ 2: WHAT ARE THE OPERATING MODES IN DATABASE MIRRORING?

Q: What are the two operating modes available in Database Mirroring?

A:
1. High-Safety Mode: Operates synchronously ensuring transactions complete on both principal and mirror before client acknowledgment; requires full recovery model
2. High-Performance Mode: Operates asynchronously allowing principal to send log records without waiting for mirror confirmation; trades availability for performance
3. High-Safety mode supports both automatic and manual failover when witness is present
4. High-Performance mode supports only forced service failover with possible data loss; witness configuration not recommended
5. Choice depends on your recovery objectives and tolerance for potential data loss during failover
6. High-Safety mode provides zero data loss in automatic failover scenarios

================================================================================

FAQ 3: WHAT IS THE ROLE OF A WITNESS SERVER?

Q: What is the purpose and role of a witness server in mirroring?

A:
1. Witness is an optional third SQL Server instance that enables automatic failover capability
2. Only serves the purpose of supporting automatic failover, does not host the database
3. Enables mirror server to recognize when principal fails and initiate automatic failover without manual intervention
4. Must be separate from both principal and mirror servers on different physical hosts
5. Quorum requirement states at least two of three instances must be connected for database availability
6. Should never be configured with witness in High-Performance mode as it can adversely impact availability

================================================================================

FAQ 4: WHAT ARE THE PREREQUISITES FOR DATABASE MIRRORING?

Q: What are the basic prerequisites for setting up Database Mirroring?

A:
1. Both principal and mirror databases must use Full Recovery Model for mirroring support
2. Database names must be identical on both principal and mirror servers
3. Both partners must run same SQL Server version but witness can run any supported version
4. Both partners must be running same SQL Server edition (Standard or Enterprise)
5. Mirror server and principal server should be on separate physical systems in different locations
6. All server instances should ideally run under same domain account for simplified configuration

================================================================================

FAQ 5: WHAT ARE THE RESTRICTIONS ON MIRRORED DATABASES?

Q: What operations are not allowed on mirrored databases?

A:
1. Backup and restore operations on mirror database are prohibited; only principal backup is allowed
2. BACKUP LOG WITH NORECOVERY cannot be executed on principal database
3. Cannot perform ALTER DATABASE operations while mirroring is active
4. Truncate log operations are restricted on mirrored databases
5. Cannot attach or detach mirrored database while mirroring session is active
6. File growth and autogrowth settings must be configured before initiating mirroring

================================================================================

FAQ 6: HOW TO PREPARE A MIRROR DATABASE FOR MIRRORING?

Q: What steps are required to prepare mirror database before mirroring?

A:
1. Take full backup of principal database with BACKUP DATABASE command
2. Take log backup of principal database to capture recent transactions
3. Restore full backup on mirror server using RESTORE DATABASE with NORECOVERY option
4. Restore all log backups taken after full backup using RESTORE LOG with NORECOVERY
5. Mirror database must be in RESTORING state for mirroring to work correctly
6. Ensure database names match exactly on both servers before establishing mirroring session

================================================================================

FAQ 7: WHAT IS TRANSACTION SAFETY LEVEL IN MIRRORING?

Q: What is transaction safety level and how does it affect mirroring?

A:
1. Transaction safety is a database property determining synchronous or asynchronous operation
2. FULL safety level enables synchronous operation and high data consistency
3. OFF safety level enables asynchronous operation for performance but risks data loss
4. FULL safety level is required for high-safety mode operation
5. OFF safety level must be used with high-performance mode
6. Changing safety level affects failover capabilities and recovery time objectives

================================================================================

FAQ 8: HOW TO ESTABLISH DATABASE MIRRORING SESSION?

Q: What are the steps to establish database mirroring session?

A:
1. Create mirroring endpoints on principal, mirror, and witness servers with specific TCP ports
2. Grant CONNECT permission on endpoints to service accounts of remote instances
3. Restore backups on mirror database in RESTORING state
4. Use Configure Database Mirroring Security Wizard in SSMS or ALTER DATABASE T-SQL
5. Set operating mode to high-safety or high-performance based on requirements
6. Monitor mirroring status in Mirroring page of Database Properties to confirm session is active

================================================================================

FAQ 9: WHAT TYPES OF FAILOVER ARE SUPPORTED?

Q: What are the different types of failover available in database mirroring?

A:
1. Automatic Failover: Occurs automatically when principal fails in high-safety mode with witness
2. Manual Failover: Administrator initiates failover when principal is still running
3. Forced Service Failover: Used in high-performance mode or during principal unavailability
4. Forced service failover may result in data loss and requires explicit command execution
5. Manual failover requires high-safety mode with FULL transaction safety
6. Automatic failover occurs within seconds without requiring administrator intervention

================================================================================

FAQ 10: WHAT HAPPENS DURING AUTOMATIC FAILOVER?

Q: What occurs when automatic failover happens in database mirroring?

A:
1. Mirror server detects principal server failure through connection loss
2. Witness confirms principal failure before mirror initiates automatic failover
3. Mirror database transitions from RESTORING state to ONLINE state
4. Mirror becomes new principal server and accepts user connections
5. Failover completes within seconds, typically between 3-5 seconds
6. Old principal becomes new mirror when it comes back online

================================================================================

FAQ 11: HOW TO MANUALLY FAILOVER DATABASE?

Q: How do you perform manual failover in database mirroring?

A:
1. Connect to principal server instance using SQL Server Management Studio
2. Navigate to Database Properties and select Mirroring tab
3. Click Failover button in Mirroring dialog
4. Confirm failover when prompted
5. Principal server attempts connection to mirror using Windows Authentication
6. If successful, roles swap with mirror becoming principal and principal becoming mirror

================================================================================

FAQ 12: WHAT IS FORCED SERVICE AND WHEN IS IT USED?

Q: What is forced service failover and in what scenarios is it used?

A:
1. Forced service is manual operation that brings mirror database online when principal is unavailable
2. Can only be used in high-performance mode or when quorum is lost
3. Executed using command: ALTER DATABASE database_name SET PARTNER FORCE_SERVICE_ALLOW_DATA_LOSS
4. Results in mirror becoming principal and transitioning to ONLINE state
5. May cause data loss if transactions were sent to principal but not yet replicated
6. Should only be used as last resort when principal server cannot be recovered

================================================================================

FAQ 13: WHAT IS QUORUM IN DATABASE MIRRORING?

Q: What is quorum and how does it impact database availability?

A:
1. Quorum is requirement that at least two of three instances be connected
2. Ensures database can only be served by one partner at a time
3. Prevents split-brain scenarios where both copies think they are principal
4. Without quorum, database becomes unavailable and forcing service is impossible
5. Quorum requirement applies primarily to high-safety mode with witness
6. In high-performance mode without witness, quorum does not apply

================================================================================

FAQ 14: WHAT ARE HARD ERRORS IN MIRRORING?

Q: What types of errors are classified as hard errors in mirroring?

A:
1. Hard errors are failures reported directly to SQL Server instance
2. Include network failures that break connection between principal and mirror
3. SQL Server process crashes or sudden termination
4. Disk storage failures and I/O subsystem errors
5. Hard errors are detected and reported immediately
6. Hard errors trigger faster recovery and failover initiation than soft errors

================================================================================

FAQ 15: WHAT ARE SOFT ERRORS IN MIRRORING?

Q: What types of errors are classified as soft errors in mirroring?

A:
1. Soft errors are failures not immediately reported by components
2. Detected through database mirroring timeout mechanism
3. Default timeout period is 10 seconds (minimum recommended value)
4. Examples include slow network performance and resource contention
5. Detection delay depends on configured timeout period
6. Increasing timeout reduces false positives but delays failure detection

================================================================================

FAQ 16: WHAT IS AUTOMATIC PAGE REPAIR IN MIRRORING?

Q: What is automatic page repair feature in database mirroring?

A:
1. Feature available on SQL Server 2008 Enterprise or later versions
2. Automatically attempts to repair unreadable data pages
3. Partner server that cannot read page requests fresh copy from other partner
4. If copy succeeds, unreadable page is replaced by good copy
5. Resolves read errors without administrator intervention
6. Operates transparently during normal database operations

================================================================================

FAQ 17: HOW TO CHECK MIRRORING STATUS?

Q: How can you monitor and check database mirroring status?

A:
1. Query sys.database_mirroring catalog view to check mirroring state
2. Mirroring role desc shows whether instance is principal or mirror
3. Mirroring state desc indicates SYNCHRONIZED, SYNCHRONIZING, or DISCONNECTED
4. Mirroring safety level desc shows FULL or OFF
5. View Database Properties Mirroring tab in SQL Server Management Studio
6. Monitor mirroring performance using DMVs and Extended Events

================================================================================

FAQ 18: WHAT IS MIRRORING STATE SYNCHRONIZED?

Q: What does SYNCHRONIZED mirroring state mean?

A:
1. SYNCHRONIZED state indicates mirror database is fully synchronized with principal
2. All committed transactions on principal are replicated to mirror
3. Mirror is ready for automatic failover without data loss
4. Zero lag between principal and mirror databases
5. Transactions can complete on principal without waiting for mirror acknowledgment in async mode
6. SYNCHRONIZED state is stable state for normal mirroring operations

================================================================================

FAQ 19: WHAT IS MIRRORING STATE SYNCHRONIZING?

Q: What does SYNCHRONIZING mirroring state mean?

A:
1. SYNCHRONIZING state indicates mirror is catching up with principal
2. Transactions are being replicated from principal to mirror but not yet synchronized
3. Automatic failover should not be attempted during SYNCHRONIZING state
4. State typically occurs when mirror server was stopped or restarted
5. Performance impact on principal occurs while mirroring is synchronizing
6. Monitor progress using sys.database_mirroring DMV

================================================================================

FAQ 20: WHAT IS MIRRORING STATE DISCONNECTED?

Q: What does DISCONNECTED mirroring state mean?

A:
1. DISCONNECTED state indicates connection loss between principal and mirror
2. Mirror server may be offline, unreachable, or experiencing network issues
3. Principal continues to accept transactions but mirror falls behind
4. In high-safety mode, principal may suspend to prevent data divergence
5. Must restore connection or resolve issues to resume mirroring
6. Check network connectivity and SQL Server error logs when disconnected

================================================================================

FAQ 21: HOW MIRRORING IMPACTS PRINCIPAL SERVER PERFORMANCE?

Q: What performance impact does database mirroring have on principal server?

A:
1. Asynchronous mirroring (high-performance mode) has minimal principal performance impact
2. Synchronous mirroring (high-safety mode) adds latency due to waiting for mirror acknowledgment
3. Network bandwidth utilization increases with transaction volume
4. CPU usage increases slightly due to log stream processing
5. I/O throughput increases on principal server for log writing
6. Performance impact depends on transaction volume and network latency

================================================================================

FAQ 22: WHAT IS ROLLING UPGRADE IN MIRRORING?

Q: How can you upgrade SQL Server instances in mirroring configuration?

A:
1. Rolling upgrade allows sequential upgrade of principal and mirror instances
2. Minimize downtime during SQL Server version upgrades
3. Upgrade mirror server first, then failover to make mirror new principal
4. Upgrade original principal while it is mirror server
5. Initiate manual failover to return to original configuration
6. Total downtime is only duration of manual failover

================================================================================

FAQ 23: WHEN IS FULL RECOVERY MODEL REQUIRED?

Q: Why is Full Recovery Model required for database mirroring?

A:
1. Full Recovery Model enables capturing all transaction log records
2. Necessary for replicating all database changes to mirror
3. Simple or Bulk-Logged recovery models do not support mirroring
4. Full Recovery Model enables transaction log backups required for mirroring
5. Database must use Full Recovery Model before initiating mirroring
6. Changing recovery model requires mirroring session to be suspended

================================================================================

FAQ 24: WHAT CROSS-DATABASE TRANSACTIONS ISSUE OCCURS IN MIRRORING?

Q: What issue can occur with cross-database transactions during automatic failover?

A:
1. Automatic failover can cause logical inconsistencies in cross-database transactions
2. Transactions updating multiple databases in same instance at failover time
3. Transactions using Microsoft Distributed Transaction Coordinator (MSDTC)
4. If failover occurs during transaction commit, databases may have conflicting state
5. Resolution requires manual data reconciliation after failover
6. Design applications to minimize cross-database transactions in mirrored environments

================================================================================

FAQ 25: CAN REPLICATION WORK WITH MIRRORED DATABASE?

Q: How does database mirroring work with SQL Server replication?

A:
1. Mirroring can be configured with replication for combined HA and data distribution
2. Configure distribution for mirror server separately from principal
3. Specify mirror name as Publisher in distribution configuration
4. Use same Distributor and snapshot folder as principal server
5. Replication publications appear only on active server
6. Replication agents require failover partner configuration to identify mirror after failover

================================================================================

FAQ 26: WHAT METADATA CHANGES WHEN FAILOVER OCCURS?

Q: What metadata changes are needed after database mirroring failover?

A:
1. Logins and users may need recreation if different between servers
2. SQL Server Agent jobs may need reconfiguration on new principal
3. Linked server references might need updating
4. Database permissions and roles should be verified
5. Jobs that depended on principal database need reassignment
6. Execute sp_help_revlogin to script logins from old principal

================================================================================

FAQ 27: WHAT IS MIRRORING ENDPOINT?

Q: What is a mirroring endpoint and why is it required?

A:
1. Mirroring endpoint is network interface for mirroring communication
2. Endpoints must be created on principal, mirror, and witness servers
3. Each endpoint requires specific TCP port for communication
4. Endpoints authenticate using Windows Authentication or certificate-based authentication
5. CONNECT permission on endpoints must be granted to service accounts
6. Endpoint configuration enables secure mirroring session establishment

================================================================================

FAQ 28: HOW TO CREATE MIRRORING ENDPOINT?

Q: How do you create mirroring endpoint on SQL Server?

A:
1. Use CREATE ENDPOINT T-SQL command on each server instance
2. Specify unique endpoint name and FOR DATABASE_MIRRORING clause
3. Configure TCP port (typically 5022 or custom port)
4. Set ROLE to PARTNER for principal and mirror servers
5. Grant CONNECT permission to service account of remote instances
6. Verify endpoint is created using sys.database_mirroring_endpoints DMV

================================================================================

FAQ 29: WHAT NETWORK REQUIREMENTS EXIST FOR MIRRORING?

Q: What network configuration is required for database mirroring?

A:
1. Network must allow TCP communication between principal and mirror servers
2. Firewall rules must allow traffic on mirroring endpoint ports
3. Network latency should be minimized to reduce transaction latency
4. Witness server must have network connectivity to both principal and mirror
5. DNS or host file entries must resolve server names correctly
6. Validate network connectivity before establishing mirroring session

================================================================================

FAQ 30: HOW TO TROUBLESHOOT DISCONNECTED STATE?

Q: How do you troubleshoot DISCONNECTED state in database mirroring?

A:
1. Check network connectivity between principal and mirror servers
2. Verify SQL Server services are running on both instances
3. Check SQL Server error logs for error messages related to mirroring
4. Verify mirroring endpoint is running and accessible
5. Check firewall rules allow traffic on mirroring endpoint port
6. Test connection using ping and telnet to verify network path

================================================================================

FAQ 31: WHAT ERROR 1475 MEANS IN MIRRORING?

Q: What does Error 1475 indicate in database mirroring?

A:
1. Error 1475 indicates database has bulk logged changes not backed up
2. Last log backup on principal must be restored on mirror
3. Occurs when database enters bulk-logged mode during mirroring setup
4. Solution is to take full backup and log backup on principal
5. Restore full backup and log backup on mirror with NORECOVERY
6. Restart mirroring after backups are restored

================================================================================

FAQ 32: HOW TO SUSPEND MIRRORING?

Q: How can you suspend a database mirroring session?

A:
1. Use ALTER DATABASE command: ALTER DATABASE database_name SET PARTNER SUSPEND
2. Suspending stops replication of changes to mirror
3. Principal continues accepting transactions
4. Mirror database stops receiving updates
5. Mirroring can be resumed without full reinitialization
6. Useful for maintenance or troubleshooting activities

================================================================================

FAQ 33: HOW TO RESUME MIRRORING?

Q: How can you resume a suspended database mirroring session?

A:
1. Use ALTER DATABASE command: ALTER DATABASE database_name SET PARTNER RESUME
2. Resuming restarts replication from principal to mirror
3. Mirror catches up with transactions that occurred during suspension
4. If lag is too large, may need to resynchronize with backups
5. Monitor progress during resume using sys.database_mirroring
6. Verify SYNCHRONIZED state before relying on automatic failover

================================================================================

FAQ 34: HOW TO REMOVE MIRRORING?

Q: How do you remove database mirroring from a database?

A:
1. Use ALTER DATABASE command: ALTER DATABASE database_name SET PARTNER OFF
2. Removing mirroring terminates active mirroring session
3. Mirror database transitions to OFFLINE state
4. Principal database becomes standalone
5. Endpoints remain on servers but are no longer used
6. Cannot be undone so plan removal carefully

================================================================================

FAQ 35: WHAT IS RECOVERY DURING FAILOVER?

Q: What happens during recovery when failover occurs?

A:
1. Mirror database rolls forward all committed transactions from log
2. Any uncommitted transactions are rolled back
3. Database transitions from RESTORING to ONLINE state
4. New principal becomes available for client connections
5. Recovery time depends on amount of undo required
6. Applications connecting with failover partner automatically reconnect

================================================================================

FAQ 36: HOW CLIENT CONNECTIONS HANDLE FAILOVER?

Q: How do client applications handle failover in database mirroring?

A:
1. SQLClient Provider provides implicit support for mirroring failover
2. Connection strings include Principal and Failover Partner parameters
3. If principal fails, client automatically redirects to mirror
4. Developer does not need to write special failover handling code
5. Connection retry logic must be implemented in application
6. Witness server automatically redirects connections after automatic failover

================================================================================

FAQ 37: WHAT IS FORCED SERVICE COMMAND SYNTAX?

Q: What is the exact syntax for forced service failover command?

A:
1. Command: ALTER DATABASE database_name SET PARTNER FORCE_SERVICE_ALLOW_DATA_LOSS
2. ALLOW_DATA_LOSS keyword explicitly acknowledges potential data loss
3. Must be executed on mirror server to bring it online as new principal
4. Command immediately transitions mirror to ONLINE state
5. No quorum check is performed during forced service
6. Should be used only when principal cannot be recovered

================================================================================

FAQ 38: WHAT MONITORING SHOULD BE DONE FOR MIRRORING?

Q: What monitoring and checks should be performed for mirrored databases?

A:
1. Monitor mirroring state using sys.database_mirroring DMV regularly
2. Alert on DISCONNECTED state to detect connection issues early
3. Monitor transaction log size on mirror to detect replication lag
4. Track failover time to ensure it meets RTO objectives
5. Monitor principal server performance during mirroring
6. Use Extended Events to capture detailed mirroring activity

================================================================================

FAQ 39: WHAT IS MIRRORING SAFETY LEVEL?

Q: What is mirroring safety level and how is it configured?

A:
1. Safety level property controls synchronous or asynchronous operation
2. FULL safety level enables synchronous operation
3. OFF safety level enables asynchronous operation
4. Set using ALTER DATABASE command with SET SAFETY
5. Must be FULL for high-safety mode and automatic failover
6. Changing safety level affects failover capabilities

================================================================================

FAQ 40: HOW LONG DOES AUTOMATIC FAILOVER TAKE?

Q: How much time does automatic failover take in mirroring?

A:
1. Automatic failover typically completes within 3-5 seconds
2. Detection time depends on network and configuration
3. Witness confirms principal failure before initiating failover
4. Recovery time depends on uncommitted transactions to rollback
5. Total time is detection plus recovery plus client reconnection
6. Test failover time in your environment before production use

================================================================================

FAQ 41: CAN MIRRORING BE USED WITH SQL ALWAYS ON?

Q: Is database mirroring compatible with SQL Server Always On?

A:
1. Database mirroring and Always On Availability Groups serve similar purposes
2. Microsoft recommends using Always On for new deployments
3. Mirroring is deprecated but still supported in SQL Server 2019 and 2022
4. Both cannot be used simultaneously on same database
5. Always On provides more features and flexibility than mirroring
6. Consider migration to Always On if currently using mirroring

================================================================================

FAQ 42: WHAT IS COMPATIBILITY BETWEEN MIRRORING AND FAILOVER CLUSTERS?

Q: How does database mirroring work with SQL Server failover clusters?

A:
1. Mirroring works between failover cluster instances and non-clustered servers
2. Cluster node failure triggers cluster failover before mirroring failover
3. Mirroring can provide additional protection for databases in cluster
4. Two-cluster configuration recommended for high-safety mode
5. Witness can reside on third cluster or non-clustered server
6. Test failover scenarios with cluster to understand interaction

================================================================================

FAQ 43: WHAT BACKUP REQUIREMENTS EXIST FOR MIRRORED DATABASES?

Q: What are backup requirements for mirrored databases?

A:
1. Full backups can be taken on principal database only
2. Log backups must be taken on principal database
3. Mirror database cannot be backed up while mirroring is active
4. Backups on principal must be restored to mirror with NORECOVERY
5. Backup chain must be maintained for recovery purposes
6. Test restore procedures regularly for mirrored database backups

================================================================================

FAQ 44: WHAT IS ROLE SWITCHING IN MIRRORING?

Q: What is role switching and how does it occur in mirroring?

A:
1. Role switching is process where principal and mirror exchange roles
2. Automatic role switching occurs during automatic failover
3. Manual role switching occurs during manual failover or role switching
4. Principal becomes mirror and mirror becomes principal
5. Other mirroring sessions are not affected by role switching
6. Connection strings with failover partner handle role switching automatically

================================================================================

FAQ 45: WHEN TO USE HIGH-SAFETY MODE?

Q: In what scenarios should high-safety mode be used?

A:
1. Use high-safety mode when data loss cannot be tolerated
2. Required for automatic failover capability
3. Best for mission-critical databases requiring near-zero RTO
4. Acceptable transaction latency increase in exchange for data protection
5. Requires witness server for automatic failover capability
6. Suitable for online transaction processing (OLTP) workloads

================================================================================

FAQ 46: WHEN TO USE HIGH-PERFORMANCE MODE?

Q: In what scenarios should high-performance mode be used?

A:
1. Use high-performance mode when availability is priority over data loss
2. For geographically remote mirror servers with high latency
3. When principal server performance cannot afford synchronous waits
4. Best for read-intensive or data warehouse workloads
5. Acceptable minor data loss during failover
6. Forced service is only failover option available

================================================================================

FAQ 47: WHAT IS TRANSACTION LOG HARDENING?

Q: What is transaction log hardening in database mirroring?

A:
1. Hardening is writing and hardening log records to stable storage
2. In high-safety mode, transaction not complete until log hardened on mirror
3. Ensures consistency between principal and mirror databases
4. Provides zero-data-loss guarantee in automatic failover
5. May introduce latency due to waiting for mirror acknowledgment
6. Critical for maintaining ACID properties across failover

================================================================================

FAQ 48: WHAT MONITORING QUERIES SHOULD BE USED?

Q: What T-SQL queries should be used to monitor mirroring?

A:
1. Query sys.database_mirroring to check session state and safety level
2. Query sys.database_mirroring_endpoints to verify endpoints are created
3. Check sys.dm_database_mirroring_connections for connection status
4. Monitor witness connections with sys.database_mirroring
5. Use sp_dbmmonitoraddmonitoredalert for automated alerts
6. Create custom monitoring using Extended Events for detailed analysis

================================================================================

FAQ 49: HOW TO CHANGE TRANSACTION SAFETY LEVEL?

Q: How do you change transaction safety level in mirroring session?

A:
1. Use ALTER DATABASE command: ALTER DATABASE database_name SET SAFETY FULL or OFF
2. Changing from OFF to FULL requires synchronizing mirror first
3. Changing from FULL to OFF can be immediate
4. Cannot change safety level while principal is unavailable
5. Changing safety affects failover capabilities
6. Verify synchronization before relying on automatic failover

================================================================================

FAQ 50: WHAT IS MIRRORING TIMEOUT?

Q: What is the mirroring timeout and how does it affect failover?

A:
1. Timeout is period waiting for response before declaring soft error
2. Default timeout is 10 seconds (minimum recommended)
3. Timeout applies to both principal and mirror servers
4. Shorter timeout detects failures faster but increases false positives
5. Longer timeout reduces false positives but delays failure detection
6. Configure using ALTER DATABASE SET PARTNER TIMEOUT command

================================================================================

FAQ 51: HOW TO CONFIGURE MIRRORING TIMEOUT?

Q: How do you set mirroring timeout value?

A:
1. Use ALTER DATABASE command: ALTER DATABASE database_name SET PARTNER TIMEOUT timeout_value
2. Timeout value is in seconds with minimum of 10
3. Default value is 10 seconds
4. Maximum practical value is around 30-60 seconds for most environments
5. Configure timeout after partners are connected
6. Higher values reduce false positives in high-latency networks

================================================================================

FAQ 52: WHAT IS AUTHENTICATION IN MIRRORING?

Q: What authentication methods are supported for mirroring endpoints?

A:
1. Windows Authentication is primary and recommended method
2. Certificate-based authentication supported for non-domain environments
3. All three instances must use same authentication method
4. Endpoints must have CONNECT permission for remote service accounts
5. Kerberos authentication available with Windows Authentication
6. Encryption can be enabled for additional security

================================================================================

FAQ 53: HOW TO ENCRYPT MIRRORING CONNECTIONS?

Q: How can you encrypt database mirroring communications?

A:
1. Create endpoints with ENCRYPTION option in CREATE ENDPOINT
2. Encryption algorithms supported include AES, RC4, and NONE
3. Set ENCRYPTION REQUIRED to enforce encryption
4. Encryption adds CPU overhead but protects data in transit
5. Both endpoints must support matching encryption algorithm
6. Verify encryption is active using sys.database_mirroring_endpoints

================================================================================

FAQ 54: WHAT IS PRINCIPAL SERVER ROLE?

Q: What is the role and responsibilities of principal server?

A:
1. Principal server hosts the active database accepting client connections
2. Generates transaction log records from all database modifications
3. Sends log stream to mirror server for replication
4. Continues operation even if mirror becomes unavailable in high-performance mode
5. Must remain available for database operations
6. Initiates failover process in case of planned maintenance

================================================================================

FAQ 55: WHAT IS MIRROR SERVER ROLE?

Q: What is the role and responsibilities of mirror server?

A:
1. Mirror server maintains standby copy of database in RESTORING state
2. Receives and applies log records from principal server
3. Becomes online when failover occurs
4. Does not accept client connections during normal operation
5. Cannot be used for read-only operations or backups
6. Continuously synchronizes with principal in high-safety mode

================================================================================

FAQ 56: CAN MIRROR DATABASE BE ACCESSED FOR REPORTING?

Q: Can mirror database be accessed for read-only queries?

A:
1. Mirror database cannot be accessed while in RESTORING state
2. Cannot be used for reporting or analytics during mirroring
3. If reporting required, use high-performance mode or snapshot isolation
4. Database snapshots can be created on mirror for reporting
5. Alternative is to use Always On readable secondaries for this purpose
6. Plan reporting strategy before implementing mirroring

================================================================================

FAQ 57: WHAT IS DATABASE MIRRORING DEPRECATION STATUS?

Q: Is database mirroring deprecated in SQL Server?

A:
1. Mirroring was marked deprecated in SQL Server 2012
2. Remains fully supported in SQL Server 2019 and 2022
3. Deprecation means no new features are being added
4. Does not mean feature is disappearing soon
5. Microsoft recommends Always On for new implementations
6. Existing mirroring deployments should be maintained and patched

================================================================================

FAQ 58: WHAT SHOULD BE USED INSTEAD OF MIRRORING?

Q: What is Microsoft's recommended alternative to database mirroring?

A:
1. Always On Availability Groups is primary recommended replacement
2. Provides database-level replication with readable secondaries
3. Supports multiple replicas for added redundancy
4. Enables geographic distribution of replicas
5. Provides more flexible high availability options
6. Recommended for all new high availability implementations

================================================================================

FAQ 59: WHAT IS LOG SHIPPING COMPARED TO MIRRORING?

Q: How does log shipping differ from database mirroring?

A:
1. Log shipping is asynchronous replication at log level
2. Simpler to implement than mirroring with fewer prerequisites
3. Does not provide automatic failover
4. Provides warm standby for disaster recovery
5. Suitable where automatic failover is not required
6. Lower cost alternative to mirroring for some scenarios

================================================================================

FAQ 60: WHEN SHOULD MIRRORING BE CONSIDERED DEPRECATED?

Q: When should you consider migrating away from mirroring?

A:
1. Consider migration when implementing new high availability solutions
2. Migrate when readable secondaries are required for reporting
3. Migrate when multiple replicas would provide better redundancy
4. Migrate when more flexible failover policies are needed
5. Migrate when maintenance becomes burden on infrastructure
6. Keep existing mirroring stable until new solution is proven

================================================================================

FAQ 61: WHAT IS PLANNED FAILOVER PROCEDURE?

Q: What steps should be followed for planned failover?

A:
1. Notify application teams of planned failover window
2. Pause or suspend non-critical batch jobs
3. Connect to principal using SSMS
4. Click Failover button on Database Properties Mirroring tab
5. Monitor failover completion and application reconnection
6. Verify new principal is functioning correctly before resuming operations

================================================================================

FAQ 62: WHAT IS UNPLANNED FAILOVER PROCEDURE?

Q: What steps should be followed during unplanned failover?

A:
1. Determine if principal server failure is temporary or permanent
2. Check SQL Server error logs on principal and mirror
3. If principal cannot be recovered quickly, initiate automatic failover
4. If automatic failover does not occur, connect to mirror and use forced service
5. Verify new principal is accepting connections correctly
6. Investigate root cause of failure after stability restored

================================================================================

FAQ 63: HOW TO HANDLE SPLIT-BRAIN SCENARIO?

Q: What is split-brain scenario and how is it prevented?

A:
1. Split-brain occurs when both principal and mirror think they are active
2. Results in divergent databases with conflicting changes
3. Prevented by quorum requirement with witness server
4. Prevents both sides of network partition from acting as principal
5. If split-brain occurs, manual resolution required
6. Use database snapshots to preserve state for forensic analysis

================================================================================

FAQ 64: WHAT HAPPENS IF WITNESS FAILS?

Q: What occurs when witness server fails?

A:
1. In high-safety mode with witness, failover cannot occur without witness
2. Automatic failover is disabled until witness is restored
3. Manual failover remains available to administrator
4. Database continues to operate but automatic protection is lost
5. Recreate witness on different server as soon as possible
6. Monitor for witness unavailability continuously

================================================================================

FAQ 65: WHAT HAPPENS IF PRINCIPAL FAILS BRIEFLY?

Q: What occurs when principal server fails temporarily then recovers?

A:
1. Mirror detects principal failure through connection timeout
2. Automatic failover occurs if witness quorum is met
3. Mirror becomes new principal and transitions to ONLINE
4. When original principal recovers, it becomes mirror
5. Mirror automatically resynchronizes with new principal
6. Verify all systems recovered correctly before resuming normal operations

================================================================================

FAQ 66: HOW TO HANDLE PRINCIPAL SYNCHRONIZATION ISSUES?

Q: What should be done if principal and mirror are out of sync?

A:
1. Check mirroring state using sys.database_mirroring
2. If SYNCHRONIZING state, allow time for synchronization
3. Monitor transaction log size to determine lag
4. If sync cannot be achieved, suspend and remove mirroring
5. Restore backups from principal to mirror with NORECOVERY
6. Re-establish mirroring connection

================================================================================

FAQ 67: WHAT IS MIRROR SERVER FAILOVER TIMING?

Q: How long does it take for mirror to become principal?

A:
1. Automatic failover detection takes seconds to minutes based on timeout
2. Recovery time depends on amount of uncommitted transactions to rollback
3. Rollback can take minutes for large transactions
4. Total failover time includes detection and recovery
5. Test failover timing in your specific environment
6. Document expected RTO for capacity planning

================================================================================

FAQ 68: HOW TO MONITOR REPLICATION LAG?

Q: How can you measure and monitor replication lag between principal and mirror?

A:
1. Query sys.database_mirroring to check synchronization status
2. Monitor transaction log free space on mirror
3. Use Extended Events to capture log send and redo rates
4. Large lag indicates mirror falling behind principal
5. Network latency and mirror server performance impact lag
6. Alert when lag exceeds predetermined threshold

================================================================================

FAQ 69: WHAT CAUSES CONTINUOUS SYNCHRONIZING STATE?

Q: Why might mirror remain in SYNCHRONIZING state?

A:
1. Mirror server performance issues prevent keeping up with principal
2. Network latency or bandwidth limitations slow log transmission
3. Mirror server disk I/O bottleneck slowing log replay
4. Uncommitted transactions on principal blocking synchronization
5. High transaction rate on principal exceeding mirror capacity
6. Resolve by improving mirror performance or reducing principal load

================================================================================

FAQ 70: HOW TO CHECK ENDPOINT STATUS?

Q: How do you verify mirroring endpoints are functioning correctly?

A:
1. Query sys.database_mirroring_endpoints to check endpoint state
2. Endpoint state should be STARTED for active mirroring
3. Verify endpoint is listening on specified TCP port
4. Check endpoint permissions with CONNECT privilege granted
5. Use Extended Events to monitor endpoint activity
6. Test connection using test login from remote server

================================================================================

FAQ 71: WHAT HAPPENS DURING PRINCIPAL SERVER RESTART?

Q: What occurs when principal server is restarted?

A:
1. Mirroring session is interrupted when principal is restarted
2. Mirror transitions to DISCONNECTED state
3. Automatic failover occurs if witness quorum is met
4. When principal restarts, it rejoins mirroring as mirror
5. Synchronization resumes automatically
6. Verify synchronization before relying on failover

================================================================================

FAQ 72: WHAT HAPPENS DURING MIRROR SERVER RESTART?

Q: What occurs when mirror server is restarted?

A:
1. Mirroring session transitions to DISCONNECTED state
2. Principal continues to accept transactions
3. Log build-up occurs on principal until mirror returns
4. Automatic failover cannot occur while mirror is offline
5. When mirror returns, synchronization begins
6. Monitor transaction log growth during mirror downtime

================================================================================

FAQ 73: CAN MIRRORING WORK ACROSS DOMAINS?

Q: Is database mirroring supported across different Active Directory domains?

A:
1. Mirroring can work across domains with certificate authentication
2. Windows Authentication requires domain trust relationship
3. Service accounts must have appropriate cross-domain permissions
4. Certificate-based authentication simpler for cross-domain scenarios
5. Network connectivity must be maintained across domain boundaries
6. Test cross-domain failover thoroughly before production use

================================================================================

FAQ 74: WHAT IS MIRROR DATABASE SNAPSHOT?

Q: Can you create snapshot on mirror database?

A:
1. Database snapshots can be created on mirror for reporting
2. Snapshot provides read-only view of database at snapshot creation time
3. Snapshot captures state before recent log records are applied
4. Useful for offloading reporting workload from principal
5. Snapshot must be refreshed periodically by dropping and recreating
6. Snapshots consume additional disk space on mirror

================================================================================

FAQ 75: HOW TO HANDLE NETWORK PARTITION?

Q: What happens in network partition scenario with mirroring?

A:
1. Network partition separates principal from mirror and witness
2. If witness remains with mirror, mirror cannot become principal
3. Quorum prevents both sides from accepting writes
4. If witness remains with principal, principal continues accepting writes
5. When partition heals, synchronization resumes
6. Quorum requirement ensures only one principal at a time

================================================================================

FAQ 76: WHAT IMPACT DOES DATABASE SIZE HAVE?

Q: How does database size impact mirroring setup and performance?

A:
1. Larger databases require more time to prepare backups
2. Mirror database size must match principal database size
3. Replication overhead increases with database size
4. Storage requirements doubled for mirrored configuration
5. Backup and restore process takes longer for large databases
6. Network bandwidth must accommodate log replication

================================================================================

FAQ 77: WHAT TRANSACTION VOLUME IMPACT?

Q: How does transaction volume impact mirroring performance?

A:
1. Higher transaction volume increases log generation rate
2. Mirror must keep pace with transaction volume for synchronization
3. Replication latency increases with transaction volume
4. High transaction rates may require higher network bandwidth
5. Principal performance impact increases with transaction volume
6. High-performance mode reduces principal latency for high volumes

================================================================================

FAQ 78: HOW TO CONFIGURE AUTOMATIC ALERTS?

Q: How can you configure automatic alerts for mirroring issues?

A:
1. Use sp_dbmmonitoraddmonitoredalert to configure alerts
2. Configure alert thresholds for replication lag and disconnection
3. Alerts sent when thresholds exceeded
4. SQL Server Agent must be running for alerts to function
5. Configure email notification for critical alerts
6. Review alert logs regularly to identify patterns

================================================================================

FAQ 79: WHAT IS DATABASE MIRRORING STATUS REPORT?

Q: How can you generate status report for mirrored database?

A:
1. Execute sp_dbmmonitorresults to get monitoring data
2. Report shows principal and mirror roles
3. Includes synchronization status and safety level
4. Shows mirror commit overhead and replication time
5. Export results to database for trending analysis
6. Schedule status report generation for scheduled monitoring

================================================================================

FAQ 80: HOW TO PREVENT DATABASE CORRUPTION REPLICATION?

Q: How is database corruption handled in mirrored environment?

A:
1. Database mirroring replicates all changes including corrupted data
2. If principal corrupted, mirror becomes corrupted through replication
3. Automatic page repair can fix some corruption scenarios
4. Corruption prevention requires maintaining separate backups
5. Regular DBCC CHECKDB scans on both servers
6. Isolation mechanisms needed for data quality protection

================================================================================

FAQ 81: WHAT IS MAXIMUM NETWORK LATENCY?

Q: What is acceptable network latency for database mirroring?

A:
1. High-safety mode recommended for latency under 100ms round-trip
2. High-performance mode suitable for latency over 100ms
3. Higher latency increases transaction commit time
4. Very high latency may make high-safety mode impractical
5. Test with actual network conditions before production deployment
6. Use performance baselines to determine acceptable thresholds

================================================================================

FAQ 82: HOW TO HANDLE LARGE TRANSACTION ROLLBACK?

Q: What happens when large transaction must be rolled back during failover?

A:
1. Large uncommitted transactions must rollback when mirror becomes principal
2. Rollback time depends on transaction size and changes
3. Database unavailability extends for duration of rollback
4. Recovery time increases with number of uncommitted transactions
5. Monitor uncommitted transaction size and count
6. Design applications to use appropriate transaction sizes

================================================================================

FAQ 83: CAN MIRRORING WORK IN VIRTUALIZED ENVIRONMENT?

Q: Is database mirroring supported in virtualized environments?

A:
1. Mirroring fully supported on virtual machines
2. Principal and mirror VMs must be on different physical hosts
3. Witness can reside on separate physical host
4. Network latency from virtualization must be acceptable
5. VM resource contention can impact mirroring performance
6. Test failover in virtualized environment before production use

================================================================================

FAQ 84: WHAT IS STORAGE CONSIDERATION FOR MIRRORING?

Q: What storage considerations apply to mirrored databases?

A:
1. Mirror server requires storage capacity equal to principal database
2. Transaction log storage needed on principal for log backups
3. Snapshot space required if creating reporting snapshots
4. Archival storage for full and log backups
5. Growth projections must account for dual storage
6. SSD storage recommended for transaction log to minimize latency

================================================================================

FAQ 85: HOW TO HANDLE PRINCIPAL NETWORK ISOLATION?

Q: What happens when principal is isolated from network?

A:
1. Isolated principal loses connection to mirror and witness
2. Without quorum, principal may suspend in high-safety mode
3. Mirror and witness maintain quorum, initiating automatic failover
4. Mirror becomes new principal
5. Isolated principal cannot serve database
6. Heal network partition to restore original configuration

================================================================================

FAQ 86: HOW TO VALIDATE MIRRORING CONFIGURATION?

Q: What steps should you follow to validate mirroring is configured correctly?

A:
1. Verify endpoints are created on all three servers
2. Test connectivity between servers using telnet on endpoint ports
3. Check permissions on endpoints using CONNECT privilege
4. Verify databases have same name on principal and mirror
5. Confirm mirroring state is SYNCHRONIZED
6. Test failover to verify configuration is functional

================================================================================

FAQ 87: WHAT IS SECURITY CONSIDERATION FOR MIRRORING?

Q: What security considerations apply to mirroring?

A:
1. Service accounts must have sufficient permissions
2. Network encryption should be enabled for data in transit
3. Authentication method must be secure and verified
4. Endpoint security permissions must be properly configured
5. Firewall rules should restrict mirroring port access
6. Regular security audits should include mirroring configuration

================================================================================

FAQ 88: HOW TO DOCUMENT MIRRORING CONFIGURATION?

Q: What should be documented for database mirroring?

A:
1. Document principal, mirror, and witness server names and IP addresses
2. Document TCP ports used for mirroring endpoints
3. Record authentication method and service accounts
4. Document operating mode and safety level settings
5. Maintain documented recovery procedures and RTO/RPO
6. Keep runbooks for common operational scenarios

================================================================================

FAQ 89: WHAT IS DISASTER RECOVERY PLAN FOR MIRRORING?

Q: What should disaster recovery plan include for mirroring?

A:
1. Plan for principal server complete failure
2. Plan for network partition between servers
3. Plan for concurrent failures of multiple components
4. Document manual failover procedures
5. Include recovery procedures for split-brain scenarios
6. Test disaster recovery procedures quarterly

================================================================================

FAQ 90: HOW TO CAPACITY PLAN FOR MIRRORING?

Q: What capacity planning considerations apply to mirroring?

A:
1. Network bandwidth must accommodate log replication
2. Storage capacity required for principal, mirror, and backups
3. CPU and memory on mirror must handle log replay
4. Disk I/O capacity on mirror must keep pace with principal
5. Calculate total cost of ownership including doubled storage
6. Review resource utilization trends over time

================================================================================

FAQ 91: HOW TO PERFORM FAILOVER TESTING?

Q: What steps should be followed for regular failover testing?

A:
1. Schedule failover testing during maintenance windows
2. Initiate manual failover from principal to mirror
3. Verify mirror transitions to ONLINE state
4. Test application connectivity to new principal
5. Measure and document actual failover time
6. Monitor system performance after failover
7. Fail back to original principal after successful testing

================================================================================

FAQ 92: WHAT IS MAINTENANCE WINDOW PLANNING?

Q: How should maintenance windows be planned with mirroring?

A:
1. Schedule maintenance during low-traffic periods
2. Coordinate maintenance between principal, mirror, and witness
3. Suspend mirroring before maintenance activities
4. Resume mirroring after all servers are updated
5. Verify synchronization before resuming normal operations
6. Document maintenance activities in system logs

================================================================================

FAQ 93: HOW TO HANDLE MIRROR RESEEDING?

Q: When is mirror reseeding required and how is it performed?

A:
1. Reseeding required when mirror database becomes inaccessible
2. Full backup from principal must be taken
3. Restore backup on mirror using NORECOVERY
4. Restore all log backups since full backup
5. Reestablish mirroring connection with ALTER DATABASE SET PARTNER
6. Monitor synchronization after reseeding

================================================================================

FAQ 94: WHAT COMPLIANCE CONSIDERATIONS EXIST?

Q: What compliance considerations apply to database mirroring?

A:
1. Ensure mirroring meets data protection requirements
2. Comply with data residency regulations
3. Document recovery time objectives for compliance
4. Maintain audit logs of failover events
5. Verify encryption meets security standards
6. Regular compliance testing and validation

================================================================================

FAQ 95: HOW TO MIGRATE FROM MIRRORING TO ALWAYS ON?

Q: What steps are involved in migrating from mirroring to Always On?

A:
1. Plan migration during maintenance window
2. Create Always On Availability Group on new servers
3. Add mirrored database as initial replica
4. Verify all replicas are SYNCHRONIZED
5. Redirect applications to Always On listener
6. Decommission old mirroring configuration

================================================================================

FAQ 96: WHAT PERFORMANCE BASELINE SHOULD BE ESTABLISHED?

Q: What performance metrics should be baselined for mirroring?

A:
1. Establish baseline for principal transaction commit time
2. Measure replication lag in steady state
3. Document network latency between servers
4. Record CPU and disk utilization on all servers
5. Baseline mirror server redo rate capacity
6. Compare baselines regularly to identify changes

================================================================================

FAQ 97: HOW TO HANDLE STANDBY SERVER FAILURES?

Q: What actions should be taken when mirror server fails?

A:
1. Assess whether mirror will be recovered or rebuilt
2. If recoverable, restart SQL Server service
3. Allow mirroring to reconnect and synchronize
4. If not recoverable, rebuild mirror from backups
5. Reestablish mirroring connection after rebuild
6. Monitor synchronization status closely after failure

================================================================================

FAQ 98: WHAT IS WARM STANDBY CAPABILITY?

Q: What is warm standby and how does mirroring provide it?

A:
1. Warm standby provides standby server ready to take over
2. Mirror database remains in RESTORING state but ready for failover
3. Provides faster recovery than cold standby
4. Automatic failover brings mirror online within seconds
5. Alternative to Always On when readable secondaries not needed
6. Ensures business continuity with minimal data loss

================================================================================

FAQ 99: HOW TO ESTABLISH BASELINE RECOVERY TIME?

Q: How should you establish baseline RTO for mirrored database?

A:
1. Test automatic failover multiple times and measure time
2. Time should include detection, failover, and recovery
3. Add margin for network variability and load variations
4. Document RTO for capacity planning and SLA purposes
5. Test manual failover time separately
6. Test forced service failover time for worst-case scenarios

================================================================================

FAQ 100: WHAT IS CONTINUOUS IMPROVEMENT FOR MIRRORING?

Q: What continuous improvement practices should be applied?

A:
1. Review monthly logs for mirroring issues or disconnections
2. Analyze failover test results to identify improvements
3. Optimize performance baseline compared to actual performance
4. Update runbooks based on operational experience
5. Conduct quarterly failover drills with application teams
6. Implement lessons learned from each operational event

================================================================================

END OF FAQ DOCUMENT

Total FAQs: 100
Last Updated: July 30, 2026 - Thursday
Source: Microsoft Official SQL Server Documentation (Learn.microsoft.com)

IMPORTANT NOTES:
- All FAQs based on Microsoft Official Documentation and verified sources
- Database Mirroring marked deprecated but fully supported in SQL Server 2019-2022
- Microsoft recommends Always On Availability Groups for new implementations
- Information accurate as of SQL Server 2022 and SQL Server 2025
- Always test procedures in non-production environment before production use
- For latest updates, refer to https://learn.microsoft.com/en-us/sql/database-engine/database-mirroring/

================================================================================
