# SQL SERVER ALWAYSON AVAILABILITY GROUPS: 100 SCENARIO-BASED FAQs

## Table of Contents
1. [Configuration and Setup FAQs (1-20)](#configuration-and-setup)
2. [Failover and Availability Modes FAQs (21-40)](#failover-and-availability-modes)
3. [Maintenance and Troubleshooting FAQs (41-60)](#maintenance-and-troubleshooting)
4. [Performance and Optimization FAQs (61-80)](#performance-and-optimization)
5. [Disaster Recovery Scenarios FAQs (81-100)](#disaster-recovery-scenarios)

---

## CONFIGURATION AND SETUP

### 1. What prerequisites must be met before enabling AlwaysOn Availability Groups?

Your environment needs these minimum requirements:

- Windows Server Failover Cluster (WSFC) infrastructure with all nodes in the same Active Directory domain
- SQL Server instances capable of supporting AlwaysOn (Enterprise or Standard edition for basic AGs)
- Database Mirroring endpoint on each participating SQL Server instance
- Administrator group membership on local computers and full WSFC cluster control
- All databases to participate must use FULL recovery model
- Service account running SQL Server needs permission to create network names

### 2. How do I enable AlwaysOn Availability Groups on SQL Server instances?

Follow this process carefully:

- Open SQL Server Configuration Manager on each instance
- Right-click the SQL Server service and select Properties
- Click the AlwaysOn High Availability tab
- Check the Enable AlwaysOn Availability Groups checkbox
- Click Apply and wait for the SQL Server service to restart
- Repeat on all instances that will host availability replicas
- Verify enablement using SQL Server Management Studio or PowerShell

### 3. What is the difference between Basic and Advanced Availability Groups?

These two types serve different needs:

- Basic AGs support only 2 replicas (primary and 1 secondary) with single database failover, limited to Standard edition SQL Server 2016 or later
- Advanced AGs support up to 8 secondary replicas, multiple databases in one group, and require Enterprise edition
- Basic AGs do not support readable secondaries, backup on secondary replicas, or integrity checks on secondaries
- Advanced AGs provide full feature set including active secondaries and complex multi-site configurations
- Basic AGs are simpler to manage but cannot be upgraded to Advanced AGs; they must be dropped and recreated

### 4. What is a Database Mirroring Endpoint and why is it required?

This endpoint handles replication traffic:

- A Database Mirroring endpoint is a TCP listener that facilitates encrypted communication between SQL Server instances
- AlwaysOn uses these endpoints to send transaction log records between primary and secondary replicas
- Each SQL Server instance must have exactly one Database Mirroring endpoint configured
- Endpoints operate on a specific TCP port (default 5022 but customizable)
- The endpoint must have appropriate firewall rules allowing traffic on its assigned port
- You can verify endpoint status using the sys.database_mirroring_endpoints catalog view

### 5. How do I create a new Availability Group?

Use the New Availability Group Wizard:

- Connect to the SQL Server instance that will host the primary replica
- In SSMS, expand AlwaysOn High Availability, right-click Availability Groups, and select New Availability Group Wizard
- Provide an AG name and select databases to include (all must meet prerequisites)
- Add all secondary replica instances and configure their properties
- Configure availability mode (synchronous-commit or asynchronous-commit) for each replica
- Set up the listener (DNS name that clients use for connections)
- Review and complete the wizard to create the AG

### 6. Can I add databases to an Availability Group after creation?

Yes, database additions are straightforward:

- In SSMS, right-click the Availability Group and select Add Database
- The database must exist on the primary replica only initially
- The database must use FULL recovery model
- No active transactions should be running on the target database
- Full backup and transaction log backups will be restored on secondary replicas
- Secondary replicas will begin syncing their copies automatically

### 7. What storage configuration is required for AlwaysOn?

Storage setup differs significantly from other HA solutions:

- AlwaysOn does not require shared storage between replicas (non-shared storage solution)
- Each SQL Server instance maintains its own independent copy of data files
- Storage can be direct-attached (DAS), SAN, or cloud-based per instance
- Each replica must have sufficient local storage for full database copies
- Transaction log files must be on fast, reliable storage to minimize replication latency
- Backup storage must be accessible for backup and restore operations across instances

### 8. How many replicas can an Availability Group contain?

Replica limits depend on your SQL Server edition:

- Enterprise edition supports up to 8 secondary replicas (primary plus 8 secondaries equals 9 total)
- Standard edition with Basic AG supports maximum 2 replicas (primary plus 1 secondary)
- Enterprise can have multiple synchronous and asynchronous commit replicas
- Standard can have only synchronous-commit replicas in Basic AG configuration
- Linux-based SQL Server with Basic AGs can have an additional configuration-only replica
- Each replica contributes to overall cluster quorum and failover decisions

### 9. What is the Availability Group Listener and how is it created?

The listener is a critical client-facing component:

- An AG Listener is a DNS name and virtual IP address that clients connect to instead of server names
- Clients specify the listener name in their connection strings (format: AGListenerName,1433)
- When failover occurs, the listener name remains the same, applications reconnect without connection string changes
- Each Availability Group can have up to one listener (Enterprise) or one listener (Standard Basic AG)
- The listener requires Windows Server Failover Cluster distributed network name and IP address resource
- DNS must be updated to point to the listener virtual IP address

### 10. How do I configure synchronous-commit availability mode?

Synchronous mode ensures no data loss:

- Right-click the Availability Group in SSMS and select Properties
- In the Availability Replicas tab, select each secondary replica
- Set Availability Mode to Synchronous-Commit
- Set Failover Mode to Automatic (requires at least 2 synchronous-commit replicas)
- The primary will not commit transactions until secondaries confirm receipt
- This mode provides zero data loss (RPO=0) but may impact transaction latency
- Best used for replicas within the same data center or low-latency network segments

### 11. How do I configure asynchronous-commit availability mode?

Asynchronous mode prioritizes performance:

- In Availability Group Properties, select the secondary replica
- Set Availability Mode to Asynchronous-Commit
- Failover Mode automatically becomes Manual for asynchronous replicas
- The primary commits transactions without waiting for secondary confirmation
- Secondaries apply log records on their own schedule, creating potential lag
- This mode allows higher transaction throughput but risks data loss during disasters
- Ideal for geographically distant replicas where network latency is high

### 12. What is the Windows Server Failover Cluster role in AlwaysOn?

WSFC is the foundation of AlwaysOn functionality:

- WSFC monitors the health of all cluster nodes and Availability Group resources
- WSFC detects SQL Server failures and initiates automatic failover when configured
- All SQL Server instances hosting AG replicas must be nodes in the same WSFC cluster
- WSFC maintains quorum (majority of voting nodes) to protect cluster integrity
- Cluster resources include the AG, listener network name, and listener IP addresses
- WSFC can span multiple subnets and geographic locations for advanced HA scenarios

### 13. How do I join a SQL Server instance to a Windows failover cluster?

Instance clustering involves multiple steps:

- Create the WSFC cluster first with at least 2-3 nodes running Windows Server
- On each SQL Server instance node, WSFC must recognize the instance as a cluster node
- In SQL Server Configuration Manager, enable AlwaysOn Availability Groups on each instance
- WSFC will automatically detect and add SQL Server cluster resources
- Verify cluster node status using Failover Cluster Manager on any cluster node
- Ensure network connectivity and firewall rules allow cluster heartbeat traffic

### 14. What is the role of quorum in an Availability Group failover?

Quorum protects against split-brain scenarios:

- Quorum is a majority vote of cluster nodes to determine cluster validity
- Node votes are assigned based on quorum model (Node Majority, Node and Disk Majority, Node and File Share Majority)
- A minimum of 3-5 odd-numbered cluster nodes is recommended for proper quorum operation
- Without quorum, the cluster stops to prevent data inconsistency
- Quorum witnesses (disk or file share) supplement node votes in smaller clusters
- For multi-site AGs, quorum configuration must account for site outages

### 15. Can AlwaysOn work without Windows Server Failover Cluster?

No, clustering is mandatory for traditional HA AGs:

- Automatic failover functionality requires WSFC cluster infrastructure
- Read-scale Availability Groups do not require WSFC (available in SQL Server 2017+)
- Read-scale AGs provide read-only workload distribution without automatic failover
- Linux-based Pacemaker can serve as cluster manager instead of WSFC
- On-premises and cloud deployments must still maintain cluster nodes
- If cluster-free deployment is required, asynchronous replication or other solutions should be considered

### 16. How do I configure backup preferences for an Availability Group?

Backup settings control where backups run:

- Right-click the Availability Group and select Properties
- Click the Backup Preferences tab to configure backup location
- Select prefer secondary replicas to reduce primary workload
- Specify secondary replica priority for backup (1=highest to 5=lowest)
- Primary can also be configured as backup location if secondaries are busy
- Configure backup jobs to run against the listener using REPLICA_ROLE() condition
- Only replicas in healthy state and configured for backups will accept backup operations

### 17. What is a configuration-only replica and when would I use one?

Configuration-only replicas serve specialized purposes:

- Configuration-only replicas hold AG metadata but not user database copies
- They count toward quorum without consuming storage or database resources
- Useful in multi-site scenarios where you need quorum without full database replicas
- Supported on SQL Server 2017+ (Linux basic AGs) and SQL Server 2019+
- These replicas appear in sys.availability_replicas and participate in cluster voting
- Reduces infrastructure cost while improving quorum for geographically distributed clusters

### 18. How do I remove a replica from an Availability Group?

Replica removal is reversible:

- Right-click the Availability Group and select Properties
- In Availability Replicas tab, select the replica to remove
- Click the Remove button
- Confirm the removal operation
- The replica's database copy will stop receiving log records
- The replica can be re-added later if needed (will require re-seeding)
- Monitor secondary replica for orphaned database copies after removal

### 19. What happens to replica databases if the primary replica goes offline?

Secondary replicas behave based on availability mode:

- Synchronous-commit secondaries will stop accepting new transactions if primary fails
- Asynchronous-commit secondaries will continue applying log records already received
- If automatic failover is configured, a healthy synchronous-commit secondary promotes to primary
- If manual failover is required, you must explicitly fail over using SSMS or T-SQL
- Replica databases remain in SYNCHRONIZED or UNSYNCHRONIZED state depending on sync status
- Transactions committed on the original primary may be lost if the replica was asynchronous

### 20. How do I verify the health status of an Availability Group?

Multiple tools provide health visibility:

- Open SSMS and expand AlwaysOn High Availability to view AG status visually
- Run sp_server_diagnostics to get detailed information about replica health
- Query sys.dm_hadr_availability_group_states for AG state information
- Use sys.dm_hadr_database_replica_states to check database synchronization status
- Review Windows Event Viewer for cluster and availability group events
- Enable SQL Server Always On health monitoring through Extended Events for detailed diagnostics

---

## FAILOVER AND AVAILABILITY MODES

### 21. What is an automatic failover and how is it triggered?

Automatic failover protects against downtime:

- Automatic failover transitions the primary role to a secondary replica without manual intervention
- Triggered when the primary replica fails and WSFC cluster detects the failure
- Requires at least 2 synchronous-commit availability replicas with automatic failover enabled
- WSFC must have a healthy quorum for automatic failover to proceed
- Failover time typically ranges from 1-3 minutes depending on cluster detection speed
- Clients connected to the listener reconnect automatically after failover completes

### 22. What is a manual failover and when would I use it?

Manual failover provides controlled transitions:

- Manual failover allows administrator to move the primary role to a secondary replica
- Initiated when performing planned maintenance on the primary replica
- Used to test failover procedures or load balance across replicas
- Manually failing over to a synchronous-commit secondary guarantees no data loss
- Manual failover to asynchronous-commit secondary may lose committed transactions
- Connection interruption is typically less than 1 minute for manual failover

### 23. What is forced failover and what are the risks?

Forced failover is a disaster recovery last resort:

- Forced failover allows you to transition to a secondary when the primary is completely unavailable
- Used when the cluster has lost quorum or the primary is irrevocably failed
- Forced failover to an unsynchronized secondary may result in data loss
- Transactions committed on the original primary will be lost if the replica was asynchronous
- After forced failover, you should immediately restore the original primary as a secondary
- Use forced failover only when normal failover is impossible and business impact justifies data loss risk

### 24. How do I perform a planned automatic failover?

Planned failover is low-impact:

- In SSMS, right-click the Availability Group and select Failover
- In the Failover Availability Group Wizard, select the target secondary replica
- Verify the target replica is in SYNCHRONIZED state
- Click Finish to start the failover operation
- Monitor the progress in the Output window
- After completion, verify the new primary's health using sp_server_diagnostics
- Re-configure the original primary as a secondary for high availability restoration

### 25. How do I perform a forced failover?

Forced failover requires caution:

- Connect to a surviving secondary replica (preferably synchronous-commit)
- Use ALTER AVAILABILITY GROUP command with FAILOVER_ALLOW_DATA_LOSS option
- Example: ALTER AVAILABILITY GROUP [AG_Name] FORCE_FAILOVER_ALLOW_DATA_LOSS;
- Confirm that you understand the data loss implications
- After forced failover, the cluster must have quorum
- Contact Microsoft Support if the original primary becomes available to plan recovery

### 26. What is the difference between planned and unplanned failovers?

These failovers have different characteristics:

- Planned failover occurs during maintenance windows and transitions synchronous-commit replicas
- Planned failover guarantees no data loss when failing to synchronized replicas
- Unplanned failover occurs due to infrastructure failure and may promote asynchronous replicas
- Unplanned failover may result in data loss if promoted replica was asynchronous-commit
- Planned failover allows you to perform health checks before and after transition
- Unplanned failover prioritizes restoring service availability over data preservation

### 27. What is a failover cluster instance and how does it differ from Availability Groups?

FCIs and AGs serve different HA purposes:

- A Failover Cluster Instance (FCI) is an instance-level failover using shared storage
- FCIs provide automatic failover but only one active instance at a time
- AlwaysOn Availability Groups provide database-level failover with multiple active replicas
- FCIs require shared storage (SAN, NAS) while AGs use non-shared storage
- FCIs are suitable for local HA within a single data center
- AGs are better for multi-site disaster recovery with active secondaries
- You can combine FCI and AG technologies for advanced HA solutions

### 28. How do I configure automatic failover between specific replicas?

Configure automatic failover carefully:

- Automatic failover works only with synchronous-commit availability replicas
- You need at least 2 replicas in synchronous-commit mode for automatic failover
- In Availability Group Properties, set Automatic Failover Mode for synchronous-commit replicas
- Set a failover priority (1 is highest priority) to control which replica becomes primary
- Only one synchronous-commit replica can be designated for automatic failover from the current primary
- Monitor the failover priority configuration to ensure the desired replica has highest priority

### 29. What happens during a failover event?

Failover transitions follow this sequence:

- WSFC detects the primary replica failure or administrator initiates manual failover
- WSFC verifies quorum is maintained and selects the target secondary replica
- The target secondary replica transitions to the primary role
- Secondary replicas sync with the new primary and transition to secondary role
- Listener automatically updates to point to the new primary replica
- Client connections are dropped and applications reconnect via listener name
- The former primary becomes a secondary when it comes back online

### 30. How do I verify replica synchronization status?

Check sync status regularly:

- In SSMS, expand Availability Group and check replica icons (green=synced, yellow=partial, red=failed)
- Query sys.dm_hadr_database_replica_states to view synchronization details
- Run sp_server_diagnostics to see replica health information
- Check sys.dm_hadr_availability_group_states for AG-level status
- Review Availability Group dashboard in SSMS for visual status summary
- Enable monitoring via SQL Server 2016+ built-in monitoring features

### 31. What causes a replica to become unsynchronized?

Multiple conditions cause desynchronization:

- Network latency or packet loss between primary and secondary prevents log delivery
- Secondary replica experiencing high CPU or disk I/O cannot keep up with log application
- Primary replica redo queue exceeds configured thresholds
- Database maintenance operations block log application on the secondary
- Primary replica failure leaves secondary unable to receive new log records
- Asynchronous-commit replicas naturally lag if primary transaction rate is high

### 32. How do I troubleshoot a replica that is stuck in resolving state?

Resolving state indicates a transient issue:

- Resolving state occurs when replica role is transitioning between primary and secondary
- Check the SQL Server error log on the affected replica for specific error messages
- Verify network connectivity between the affected replica and other cluster nodes
- Check WSFC cluster status using Failover Cluster Manager
- If stuck for more than a few minutes, restart the SQL Server service on the replica
- Review sys.dm_hadr_availability_group_states to see when the state change occurred

### 33. What is redo and how does it relate to failover?

Redo is part of replica synchronization:

- Redo is the process of applying transaction log records on secondary replicas
- Redo happens continuously on asynchronous replicas to stay as current as possible
- During failover, the new primary begins redo on former primary (now secondary)
- Redo rate is limited by secondary replica's storage performance
- High redo queue indicates secondary cannot keep pace with primary transaction rate
- Monitoring redo queue length helps identify performance bottlenecks

### 34. How do I handle a replica database that is in NOT RECOVERING state?

NOT RECOVERING state indicates a critical problem:

- NOT RECOVERING state means the replica database cannot open or apply log records
- Check SQL Server error log for the cause (usually corruption or permission issues)
- Verify database file integrity using DBCC CHECKDB
- Confirm replica's SQL Server service account has proper file permissions
- If corruption is found, rebuild the database copy on the replica
- Remove the database from the AG, fix the issue, then re-add the database

### 35. What are recovery point objective (RPO) and recovery time objective (RTO)?

These metrics define disaster recovery capabilities:

- RPO (Recovery Point Objective) is the maximum acceptable data loss measured in time
- RTO (Recovery Time Objective) is the maximum acceptable downtime before service restoration
- Synchronous-commit AGs achieve RPO=0 (zero data loss) when failing to synchronized replicas
- Asynchronous-commit AGs have RPO measured in seconds or minutes depending on lag
- RTO for automatic failover is typically 1-3 minutes for detection and transition
- RTO for manual failover is typically 5-10 minutes including manual intervention time

### 36. How do I configure priority-based automatic failover?

Priority settings control failover target:

- Failover priority is a number (1-5) assigned to each synchronous-commit secondary
- Lower numbers have higher priority for becoming the new primary
- Set failover priority in Availability Group Properties under replica settings
- Only one synchronous-commit replica can have automatic failover enabled
- Priority affects which replica becomes primary if multiple are eligible
- Change priority if you want to shift automatic failover target to a different replica

### 37. What happens to the former primary replica after failover?

The former primary becomes a secondary:

- Former primary automatically transitions to secondary role when it re-joins the cluster
- Former primary begins receiving and applying log records from the new primary
- If former primary was synchronous-commit, it resynchronizes as synchronous-commit
- Data on former primary is restored to match new primary through redo/undo process
- Redo-undo process may take several minutes depending on log backlog
- Monitor former primary to ensure it successfully re-synchronizes

### 38. Can I have multiple availability groups in the same cluster?

Yes, multiple AGs share cluster infrastructure:

- One cluster can host multiple Availability Groups
- Each AG maintains separate failover and synchronization behavior
- Cluster resources are shared, but AGs operate independently
- Monitor cluster load when hosting many AGs on limited nodes
- Each AG needs its own listener for client connections
- Failover of one AG does not affect other AGs in the cluster

### 39. How do I move an Availability Group listener to a different IP address?

Listener IP changes require caution:

- In Failover Cluster Manager, locate the listener network name resource
- Right-click and select Properties
- Modify the IP address resource assignment
- Update DNS records to reflect the new IP address
- Test client connections to the new listener IP
- Update any firewall rules for the new IP address
- Inform applications about potential brief connection interruptions

### 40. What is automatic page repair in the context of AGs?

Automatic page repair prevents corruption spread:

- Automatic page repair detects corrupted pages on secondary replicas
- Secondary automatically requests the page from the primary replica
- Primary sends a fresh copy of the page to the secondary
- This prevents corruption from being promoted during failover
- Corrupted pages on primary must be fixed manually through backup/restore
- Enable tracing to monitor automatic page repair activity in your AGs

---

## MAINTENANCE AND TROUBLESHOOTING

### 41. How do I perform rolling updates on an Availability Group?

Rolling updates minimize downtime:

- Ensure you have at least 2 synchronous-commit secondary replicas
- Update the lowest priority secondary replica first (not automatic failover target)
- Stop SQL Server service, apply updates, and start the service
- Wait for the replica to resynchronize before proceeding to the next replica
- Perform a manual failover to one of the updated secondaries
- Update the original primary replica
- Fail back to the original primary after verification

### 42. How do I monitor Availability Group health using Extended Events?

Extended Events provide detailed diagnostics:

- Create an Extended Events session targeting AlwaysOn events
- Key events include availability_replica_state_change, availability_group_state_change
- Use the system_health session which includes relevant AlwaysOn events
- Query sys.dm_xe_session_events for active event collection
- Extended Events capture more detail than sp_server_diagnostics
- Analyze events to identify patterns in replica failures or network issues

### 43. What backup strategy is recommended for Availability Groups?

Backup planning is essential:

- Configure backup to run on secondary replicas to minimize primary workload
- Use REPLICA_ROLE() function in backup jobs to target the appropriate replica
- Take full backups from the primary or configured secondary
- Take transaction log backups every 15-30 minutes from the same replica
- Backup history is separate on each replica, coordinate backups centrally
- Backups from secondary replicas cannot be used for point-in-time recovery across replicas
- Test restore procedures regularly to ensure backup integrity

### 44. How do I troubleshoot high network latency between replicas?

Network issues impact AG performance:

- Check network connectivity between replica nodes using ping and tracert
- Use network monitoring tools to measure latency and packet loss
- Configure replicas in the same geographic region to reduce latency
- For multi-site AGs, use dedicated network links between sites
- Check switch port mirroring status and network load
- Use sys.dm_hadr_database_replica_states to monitor log send queue length
- High send queue indicates network or replica performance bottleneck

### 45. What is the maximum supported database size in an Availability Group?

Database size limits are generous:

- AlwaysOn supports databases of any size supported by SQL Server (up to 524,272 TB)
- Large database initial seeding takes longer, plan backup/restore windows accordingly
- Very large databases may cause longer synchronization times
- Monitor disk space on all replicas to ensure sufficient capacity
- Use backup compression when adding very large databases to AGs
- Consider initial full seeding on separate network connections for large databases

### 46. How do I handle database corruption on a secondary replica?

Corruption requires careful remediation:

- If secondary replica has corruption, run DBCC CHECKDB with REPAIR_REBUILD
- Set the database to SINGLE_USER mode before running DBCC CHECKDB
- Primary replica continues normal operations while secondary is being repaired
- After repair, the secondary may need to be re-seeded to ensure consistency
- Monitor the repair operation closely and review detailed error output
- Consider rebuilding the secondary from a recent backup if corruption is extensive

### 47. What causes the redo queue to grow unexpectedly?

Redo queue growth indicates performance issues:

- Primary replica is generating transactions faster than secondary can apply them
- Secondary replica's disk I/O subsystem is a performance bottleneck
- Network bandwidth between primary and secondary is insufficient
- Secondary replica's CPU is saturated processing other workloads
- Missing indexes or inefficient queries slow down log application
- Increase secondary replica resources (CPU, disk, network) to improve redo performance

### 48. How do I remove a database from an Availability Group?

Database removal is straightforward:

- Right-click the database within the AG and select Remove Database
- The database becomes a standalone database on each replica
- Replicas' copies of the database remain intact but stop receiving log records
- You can configure the removed database differently on each replica
- Monitor to ensure log truncation resumes on all replicas after removal
- If you need to re-add the database, it must be re-seeded

### 49. How do I change the availability mode of a replica?

Availability mode changes require failover planning:

- Right-click the Availability Group and select Properties
- Select the replica to modify in the Availability Replicas tab
- Change the Availability Mode dropdown (Synchronous-Commit or Asynchronous-Commit)
- If changing from synchronous to asynchronous, wait for replica to synchronize first
- If changing to synchronous-commit, the replica must be healthy and synchronized
- After mode change, verify synchronization status with sys.dm_hadr_database_replica_states

### 50. What is the impact of losing quorum on an Availability Group?

Quorum loss is catastrophic:

- When cluster loses quorum, all replicas take the AG offline to prevent data corruption
- Without quorum, no automatic or manual failover can occur
- You must force quorum and potentially force failover to restore service
- Forcing quorum carries risk of data inconsistency across replicas
- Quorum loss usually indicates infrastructure problem (network partition, node failure)
- Prevent quorum loss by proper cluster design with odd-numbered nodes or witnesses

### 51. How do I monitor the secondary replica's apply queue?

Apply queue indicates secondary lag:

- Query sys.dm_hadr_database_replica_states and examine redo_queue_size
- If apply queue is 0, the secondary is caught up with the primary
- Non-zero apply queue indicates transactions waiting to be applied on secondary
- Growing apply queue suggests secondary cannot keep pace with primary workload
- Check secondary replica's CPU and disk utilization when apply queue grows
- Redo_queue_size measured in KB, growing queue requires performance investigation

### 52. What happens to tempdb during Availability Group failover?

Tempdb is not replicated:

- Tempdb is not part of any Availability Group (system database)
- During failover, tempdb is recreated fresh on the new primary
- Applications must reconnect and rebuild temporary objects after failover
- Tempdb on secondary replicas is not synchronized with primary
- Each replica maintains independent tempdb for its local operations
- Design applications to tolerate tempdb recreation during failover events

### 53. How do I verify AG replica endpoint connectivity?

Endpoint connectivity is essential:

- Query sys.database_mirroring_endpoints to view endpoint configuration
- Verify the endpoint is in STARTED state
- Test connectivity using Telnet or PortQry from each replica to others
- Firewall rules must allow traffic on the endpoint port (default 5022)
- Use SQL Server Error Log to identify endpoint authentication issues
- Network configuration must support endpoint-to-endpoint communication

### 54. What should I do if a replica is in a SUSPENDED state?

Suspended state indicates replication stopped:

- Suspended state means log flow from primary to secondary has halted
- Automatic suspension occurs when secondary cannot apply log records
- Check SQL Server error log for specific suspension cause
- Manually resume using sp_server_diagnostics or ALTER AVAILABILITY GROUP RESUME
- After resuming, the secondary will begin redo from the suspension point
- If redo fails again, investigate and fix the underlying cause before resuming

### 55. How do I export Availability Group configuration for backup?

Configuration export aids recovery:

- Right-click the Availability Group and select Script Availability Group
- Choose "Create to New Query Window" or "Create to File"
- The script includes AG creation, replica configuration, and listener setup
- Store the script in source control or secure backup location
- Use the script to quickly recreate AGs if cluster is destroyed
- Remember to also backup database backup files for full recovery

### 56. How do I monitor replica CPU and memory usage?

Resource monitoring aids performance:

- Use Performance Monitor on each replica node to track CPU and memory
- Monitor SQL Server:Availability Replica performance counters
- Track "Log Bytes Received/sec" and "Log Bytes Redone/sec"
- Use sys.dm_exec_requests to identify long-running queries on secondary
- Secondary resource constraints slow down log application and redo
- Ensure replicas have sufficient resources (CPU, memory, disk I/O) for workload

### 57. What is the maximum number of Availability Groups per cluster?

AG limits are operational:

- There is no hard limit on the number of AGs per cluster
- Practical limits depend on cluster node count and resources
- Each AG requires a listener network name and IP address resource
- Monitor cluster resource usage when hosting many AGs
- Each AG independently maintains quorum participation
- Scale testing recommended to determine practical AG limits for your environment

### 58. How do I handle a network partition in the cluster?

Network partitions are dangerous:

- A network partition splits the cluster into disconnected segments
- Each segment tries to maintain quorum independently
- Without proper quorum configuration, both segments may try to run simultaneously
- Configure quorum witnesses (disk or file share) on a neutral network segment
- File share witness must be accessible to all cluster nodes
- Quorum witness prevents both partitions from running simultaneously

### 59. What happens to the listener during failover?

Listener provides transparent failover:

- Listener network name automatically updates to point to the new primary
- DNS update takes 1-2 minutes depending on client DNS cache
- Clients using the listener name reconnect to the new primary automatically
- Applications using IP addresses directly will not benefit from listener protection
- Listener provides automatic connection retry logic
- Configure listener login timeouts appropriately for your RTO requirements

### 60. How do I troubleshoot listener connectivity issues?

Listener issues prevent application access:

- Verify listener exists in cluster using Failover Cluster Manager
- Test connectivity using SQL Server Management Studio with listener name
- Use nslookup to verify listener DNS resolution
- Check listener network name resource status in cluster
- Verify listener IP address is accessible from application servers
- If listener name won't resolve, check DNS server configuration and zones

---

## PERFORMANCE AND OPTIMIZATION

### 61. How can I reduce log send lag between replicas?

Minimizing lag improves failover readiness:

- Increase network bandwidth between primary and secondary replicas
- Use dedicated network adapters for replication traffic
- Tune SQL Server memory and CPU allocation for log reader
- Optimize primary workload to reduce transaction rate when possible
- Enable compression for network traffic in high-latency environments
- Place replicas on faster storage (SSDs) to reduce log write latency

### 62. What impact does asynchronous-commit mode have on performance?

Asynchronous mode improves primary throughput:

- Asynchronous-commit mode allows primary to commit transactions without waiting for secondary
- Reduces write latency on primary by eliminating secondary acknowledgment wait
- Improves OLTP application response times especially for remote secondaries
- Trade-off is accepting potential data loss if primary fails unexpectedly
- Use asynchronous-commit for non-critical databases or geographically distant replicas
- Monitor asynchronous replica lag to maintain acceptable RPO

### 63. How do I optimize storage I/O for Availability Groups?

Storage optimization is critical:

- Use SSDs for transaction log files to minimize write latency
- Store data files and log files on separate physical disks
- Configure storage RAID levels appropriately (RAID-10 for OLTP, RAID-5 for read-heavy)
- Disable unnecessary indexing on secondary replicas to save I/O
- Monitor disk queue length on all replicas
- Use storage compression carefully as it increases CPU on secondary

### 64. What is the impact of network bandwidth on AG replication?

Network is a critical bottleneck:

- Network bandwidth must support peak transaction rate of primary workload
- Insufficient bandwidth causes log send queue to accumulate
- Very high send queue indicates network saturation
- Multi-site AGs should use dedicated network links between sites
- WAN optimization techniques can help with geographically distant replicas
- Monitor network utilization and upgrade links if approaching capacity

### 65. How do I tune secondary replica redo performance?

Redo tuning improves lag:

- Allocate more CPU cores to secondary for better redo parallelism
- Ensure secondary has sufficient memory to cache working set
- Disable unnecessary index maintenance on secondary
- Use lock escalation settings optimized for high throughput
- Monitor redo queue length to identify performance issues
- Query sys.dm_hadr_database_replica_states regularly for redo_rate

### 66. What is the impact of Transaction Log file size on replication?

Log file sizing affects replication flow:

- Larger transaction logs on primary reduce frequency of log file growth
- Small log files cause frequent growth events which slow replication
- Log file auto-growth can introduce temporary lag during growth operations
- Pre-size transaction logs to anticipated peak workload
- Monitor log file growth frequency and adjust sizing if necessary
- Both primary and secondary should have similarly sized log files

### 67. How does backup impact Availability Group performance?

Backups affect replication:

- Backups on secondary replicas consume I/O that could be used for redo
- Large full backups may temporarily increase replica lag
- Schedule backups during low-activity periods to minimize impact
- Use backup compression to reduce I/O impact
- Transaction log backups on secondary have minimal impact compared to full backups
- Monitor replica lag during backup windows

### 68. Should I use query hints on secondary replicas for reading?

Query hints help read workloads:

- Secondary replicas allow read-only connections when configured
- Apply NOLOCK or READ_UNCOMMITTED hints for non-critical read workloads
- Avoid blocking locks on secondary to allow redo to progress
- Use isolation levels wisely to balance consistency with redo concurrency
- Very restrictive isolation levels (SERIALIZABLE) can block redo operations
- Monitor lock conflicts between redo and user queries on secondary

### 69. How can I monitor replication lag in real-time?

Real-time lag monitoring aids troubleshooting:

- Query sys.dm_hadr_database_replica_states to get current redo_queue_size
- Calculate lag by monitoring redo_queue_size trends over time
- Extended Events can track log send and redo completion in detail
- Configure alerts when redo queue exceeds configured threshold
- Use Performance Monitor counters for availability replicas
- Create dashboard to visualize lag across all replicas

### 70. What is the optimal cluster node configuration?

Cluster design impacts performance:

- Use odd number of cluster nodes (3 or 5) for proper quorum
- Ensure all nodes have similar computing resources
- Use dedicated network adapters for cluster heartbeat traffic
- Separate replication traffic on dedicated network links if possible
- Configure cluster networks with proper health detection sensitivity
- Use lower-latency network hardware for cluster communication

### 71. How do I identify blocking issues on secondary replicas?

Blocking hurts replication:

- Queries on secondary replicas can block redo operations
- Query sys.dm_exec_requests to find long-running queries blocking redo
- Use sp_who2 to identify blocking processes on secondary
- Schedule user queries during maintenance windows if blocking occurs frequently
- Use read_only connections with appropriate isolation levels
- Consider adding more secondary replicas to distribute read workload

### 72. What is the impact of index maintenance on replication?

Index maintenance consumes resources:

- Index maintenance (DBCC DBREINDEX, ALTER INDEX REBUILD) generates many log records
- High maintenance activity on primary increases replication lag to secondaries
- Schedule intensive maintenance during low-activity periods
- On secondary replicas, avoid index maintenance that increases I/O
- Use index statistics maintenance (UPDATE STATISTICS) instead of rebuilds when possible
- Monitor log generation rate during maintenance windows

### 73. How do I optimize for multi-site Availability Groups?

Multi-site AGs require special tuning:

- Use asynchronous-commit mode for remote sites to avoid latency impact
- Implement WAN optimization techniques for long-distance replication
- Monitor network latency and bandwidth utilization continuously
- Design application retry logic to handle increased failover time
- Use dedicated network circuits for replication traffic
- Consider hybrid cloud deployment with on-premises primary

### 74. What CPU usage is normal for Availability Group operations?

CPU usage varies by workload:

- Light workloads may use minimal CPU for AG operations
- Redo operations on secondary can consume significant CPU during high load
- Synchronization processes consume CPU during replica re-seeding
- Network compression for replication adds CPU overhead
- Monitor CPU utilization to identify bottlenecks
- High CPU on secondary during redo indicates need for performance investigation

### 75. How does connection pooling affect Availability Group failover?

Connection pooling impacts failover experience:

- Connection pooling applications may hold connections across failover
- Pooled connections must detect failover and reconnect automatically
- Configure connection pooling parameters to detect failed connections quickly
- Set ConnectRetryCount and ConnectRetryInterval appropriately
- Stale pool connections will fail and force application reconnection
- Monitor application logs for connection errors during failover events

### 76. What is the memory footprint of maintaining an Availability Group?

Memory usage for AG operations:

- AG implementation uses memory for log buffers and redo operations
- On idle instance, AG uses minimal memory (0 threads noted in Microsoft documentation)
- Under high load, AG operations can consume significant memory
- Configure SQL Server max memory setting leaving room for AG operations
- Monitor memory pressure using Performance Monitor
- Out-of-memory conditions can cause AG synchronization failures

### 77. How do I optimize for databases with high transaction rate?

High-transaction databases need special tuning:

- Ensure network bandwidth supports peak transaction rate
- Use asynchronous-commit mode for remote replicas
- Allocate more resources to log reader process
- Monitor log send and redo rates to identify bottlenecks
- Consider compression for replication traffic if network-bound
- May need to limit read workload on secondary if it interferes with redo

### 78. What is the impact of distributed transactions on Availability Groups?

Distributed transactions affect replication:

- Distributed transactions across AG replicas are not supported
- Only transactions on primary replica participate in AG replication
- Distributed transactions to other databases/servers must complete on primary
- Secondary replicas cannot be targets for distributed transactions
- Design applications to avoid distributed transactions across AG boundaries
- Use local transactions on primary for best performance and reliability

### 79. How should I size the transaction log for optimal AG performance?

Log sizing is critical:

- Size transaction log to support average hourly workload
- Pre-size to avoid frequent auto-growth events
- Initial size should accommodate growth until next maintenance window
- Monitor log space used percentage to detect undersizing
- Leave sufficient free space for automatic growth if needed
- Coordinate log sizing across all replicas for consistency

### 80. How do I monitor and alert on AG performance metrics?

Monitoring ensures health:

- Set up SQL Agent jobs to monitor AG health regularly
- Use sp_server_diagnostics to get comprehensive health status
- Query sys.dm_hadr_database_replica_states for detailed metrics
- Configure alerts for redo queue size exceeding thresholds
- Monitor synchronization state changes in AG
- Use third-party monitoring tools for integration with enterprise monitoring

---

## DISASTER RECOVERY SCENARIOS

### 81. What is the proper disaster recovery topology for a multi-site AG?

Multi-site design principles:

- Primary site contains 2-3 synchronous-commit replicas in local data center
- Remote DR site contains 1-2 asynchronous-commit replicas
- Synchronous replicas provide local HA with automatic failover capability
- Asynchronous replicas in remote site provide disaster recovery with RTO and RPO
- Quorum configuration should prevent DR site from causing cluster shutdown if disconnected
- File share witness should reside in a third location if possible

### 82. How do I handle a complete primary data center failure?

Primary data center outage procedure:

- If primary site loses connectivity, cluster loses quorum
- Promote the best available secondary in remote site to primary role
- Use forced failover if necessary, understanding data loss may occur
- Update listeners to point to new primary in remote site
- Restore failed primary site as secondary when recovered
- Communicate with all applications about new connection endpoints

### 83. What backup strategy should I implement for disaster recovery?

Comprehensive backup strategy:

- Take full database backups weekly from primary or designated secondary
- Take transaction log backups every 15 minutes from same replica
- Store backups in geographically separate location from databases
- Test restore procedure monthly to ensure recovery capability
- Document backup location and access procedures
- Maintain multiple backup copies at different retention levels

### 84. How do I recover from corruption found on the primary replica?

Primary corruption recovery:

- Do not fail over to corrupt secondary as it may have same corruption
- Restore primary database from clean backup before corruption occurred
- Restore transaction log backups to point of failure if possible
- Re-seed affected secondaries after primary restoration
- Investigate root cause of corruption (software bug, hardware failure)
- Run DBCC CHECKDB on all replicas to verify no spread of corruption

### 85. What is the procedure for planned disaster recovery testing?

Testing validates readiness:

- Schedule DR test during maintenance window with minimal production impact
- Document current primary role and listener configuration
- Simulate primary data center failure by stopping network access
- Promote remote secondary to primary using forced failover
- Verify applications can connect to new primary via listener
- Run critical business transactions to confirm functionality
- Fail back to original primary when testing complete

### 86. How do I manage network partition between primary and remote sites?

Network partition handling:

- If primary site loses connectivity to remote site, secondary goes DISCONNECTED
- Quorum may fail if DR site has voting nodes and primary loses majority
- File share witness helps prevent total quorum loss
- If quorum is lost on primary, force quorum to restore primary
- Once network is restored, manually re-establish replication to catch up remote site
- Monitor network connectivity continuously to detect partition early

### 87. What data recovery options exist after unplanned failover?

Post-failover recovery options:

- Original primary may have commits not yet on secondary
- If original primary is still running, it will be designated "orphaned primary"
- Orphaned primary must be demoted and re-joined as secondary
- Connect orphaned primary to new primary to determine divergence point
- Use SQL Server recovery tools to extract data from old primary if needed
- Accept data loss or restore lost transactions from backup

### 88. How do I establish a site recovery plan (SRP) for Availability Groups?

SRP documentation:

- Document all AG configuration including replicas, listeners, endpoints
- Document backup location and recovery procedures
- Create disaster declaration process and escalation paths
- Define roles and responsibilities during disaster event
- Document cut-over steps to activate remote site as primary
- Schedule quarterly review and testing of SRP
- Maintain offline copy of SRP in secure location

### 89. What is the failback procedure after disaster recovery activation?

Failback restores original configuration:

- Once original primary site is recovered, plan failback to restore high availability
- Verify original primary is healthy before attempting failback
- Run DBCC CHECKDB on original primary to ensure no corruption
- Re-join original primary as secondary to new primary (remote site)
- Allow original primary to catch up fully before failover
- Perform planned failover to move primary role back to original site
- Verify all replicas resynchronize and return to healthy state

### 90. How do I handle RPO requirements in multi-site design?

RPO affects replica configuration:

- Synchronous-commit replicas achieve RPO=0 but suffer latency
- Asynchronous replicas have RPO measured in seconds/minutes
- For RPO of less than 1 minute, use synchronous-commit for remote replicas
- For RPO of 5+ minutes, asynchronous-commit provides better performance
- Monitor actual lag (redo queue) to verify RPO is being met
- Adjust replica configuration if RPO requirements change

### 91. What is the RTO for different failover scenarios?

RTO varies by scenario:

- Planned failover to synchronous replica: typically 1-2 minutes
- Automatic failover: typically 1-3 minutes including detection time
- Manual failover to asynchronous replica: typically 5-10 minutes
- Forced failover: variable, depends on recovery from data loss
- Failback to original primary: 10-20 minutes for catch-up and failover
- Document actual RTO for each scenario through testing

### 92. How do I handle multiple database failover with dependencies?

Dependent database failover:

- If databases have foreign key relationships, all must fail over together
- AlwaysOn AGs naturally group databases for coordinated failover
- Add all dependent databases to the same AG for atomicity
- Document database dependencies in disaster recovery plan
- Test failover of multiple dependent databases to verify consistency
- Avoid splitting dependent databases across multiple AGs

### 93. What is the procedure for declared disaster activation?

Disaster activation steps:

- Declare disaster after exhausting recovery options for primary site
- Activate remote site as primary through forced failover if necessary
- Update DNS/listener to point to new primary
- Notify all application teams of new connection endpoints
- Execute application failover procedures (if any)
- Monitor new primary for stability and performance
- Begin communication with stakeholders about recovery timeline

### 94. How should I handle application connection strings for disaster recovery?

Connection string strategy:

- Use listener name in connection strings instead of server names
- Listener handles transparent failover without connection string changes
- Avoid using IP addresses directly in connection strings
- Configure connection retry logic to handle transient failover disconnections
- Use connection pooling with appropriate timeout settings
- Test application behavior during failover to verify reconnection works

### 95. What monitoring should be in place to detect disaster early?

Proactive disaster detection:

- Monitor primary replica health continuously using sp_server_diagnostics
- Alert on replica becoming unsynchronized
- Monitor network connectivity between sites
- Alert on cluster node failures or quorum changes
- Track redo queue size and alert on unexpected growth
- Monitor backup completion and alert on backup failures
- Set up automated testing of failover capabilities monthly

### 96. How do I document AG configuration for disaster recovery?

Documentation essentials:

- Document all replica server names and IP addresses
- Record listener name and IP address
- List all databases in the AG
- Document availability modes and failover settings
- Record quorum configuration and cluster resources
- List backup locations and retention policies
- Document recovery procedures step-by-step with decision points

### 97. What is the procedure for split-brain prevention in AG?

Preventing split-brain scenarios:

- Ensure cluster has proper quorum configuration with witnesses
- Configure cluster network to detect and handle network partitions
- Use file share witness in neutral location for multi-site clusters
- Implement cluster network health monitoring
- Set cluster isolation policies to shut down rather than allow split brain
- Test network partition scenarios to verify isolation works properly

### 98. How do I coordinate disaster recovery across multiple AGs?

Multi-AG coordination:

- Document failover sequence if multiple AGs must fail over
- Ensure all AGs share cluster infrastructure for coordinated failover
- Use same remote site for all AGs to maintain consistency
- Test failover sequence to verify coordination works
- Consider staggered failover of AGs if resources on DR site are limited
- Maintain centralized disaster recovery plan covering all AGs

### 99. What are the common mistakes in AG disaster recovery planning?

Avoid these pitfalls:

- Assuming automatic failover will always work without testing
- Not documenting disaster recovery procedures
- Storing disaster recovery documentation only on primary site
- Not testing disaster recovery plan regularly
- Using RPO/RTO targets without verifying achievability
- Deploying disaster recovery solution without monitoring
- Not coordinating application-level disaster recovery with database failover
- Failing to update documentation after configuration changes

### 100. How do I implement continuous disaster recovery validation?

Continuous validation ensures readiness:

- Schedule monthly DR testing to validate failover procedures
- Rotate testing between different secondary replicas
- Include application teams in failover testing
- Measure actual RTO and RPO during testing
- Document results and any deviations from plan
- Update procedures based on testing outcomes
- Use test failures as learning opportunities to improve plan
- Maintain test results for compliance and audit purposes

---

## IMPORTANT NOTES FROM MICROSOFT OFFICIAL DOCUMENTATION

Based on Microsoft Learn and SQL Server official resources:

- AlwaysOn Availability Groups require Windows Server Failover Cluster infrastructure for automatic failover capabilities
- Basic Availability Groups in Standard Edition support maximum 2 replicas with limitations on secondary replica usage
- Enterprise Edition supports up to 8 secondary replicas with full feature set
- Synchronous-commit mode achieves RPO=0 but introduces latency on primary transactions
- Asynchronous-commit mode optimizes for throughput at cost of potential data loss
- Forced failover should only be used in disaster scenarios as it risks data loss
- Database Mirroring endpoints are mandatory for all AG replicas
- All databases in AG must use FULL recovery model
- Non-shared storage architecture means each replica maintains independent copy
- Listener provides transparent connection redirection during failover
- File share or disk witness helps maintain quorum in multi-site scenarios
- Read-scale availability groups (SQL Server 2017+) do not require WSFC
- SQL Server on Linux supports Pacemaker as cluster manager alternative
- Backup and restore operations must be coordinated across all replicas
- Network bandwidth and latency are critical factors in multi-site deployments
- Redo/undo operations consume secondary replica resources
- Connection retry logic in applications helps with failover resilience
- Regular testing of disaster recovery procedures is essential
- Quorum configuration must account for geographic distribution of replicas

---

## REFERENCE SOURCES

- Microsoft Learn: Always On Availability Groups Documentation
- Microsoft SQL Server Blog: AlwaysOn Team Blog
- Microsoft Learn: Failover and Failover Modes
- Microsoft Learn: Prerequisites, Restrictions, and Recommendations for AlwaysOn
- Microsoft Learn: Enable or Disable Availability Groups
- SQL Server 2012 AlwaysOn High Availability and Disaster Recovery Design Patterns
- Microsoft Learn: Perform a Forced Manual Failover of an Availability Group
- Microsoft Press: Manage High Availability and Disaster Recovery
