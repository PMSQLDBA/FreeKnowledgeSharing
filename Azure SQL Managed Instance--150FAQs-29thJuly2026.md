# Azure SQL Managed Instance: 150 FAQs

## Section 1: Basics (Questions 1-15)

### 1. What is Azure SQL Managed Instance?
- Fully managed relational database service
- Provides SQL Server compatibility on Azure infrastructure
- No need to manage underlying hardware, OS, or database engine patches
- Automatic updates and patching handled by Azure

### 2. How does Managed Instance differ from SQL Database (single database)?
- Managed Instance runs inside a VNet with instance-level scope
- SQL Database uses multitenant, shared infrastructure
- Managed Instance provides more surface area control
- Better compatibility with on-premises SQL Server features
- Managed Instance supports more legacy features

### 3. What SQL Server versions does Managed Instance support?
- Supports SQL Server 2016, 2017, 2019, and 2022 compatibility levels
- Service automatically updates the engine version
- You control the compatibility level of individual databases
- Can run multiple compatibility levels on same instance

### 4. Is Managed Instance a PaaS or IaaS offering?
- Managed Instance is PaaS (Platform as a Service)
- You don't manage virtual machine, storage, or patches
- Focus on databases, logins, and configuration
- Azure handles infrastructure and maintenance

### 5. What are the licensing models for Managed Instance?
- Pay-as-you-go hourly pricing available
- Azure Hybrid Benefit (AHUB) for existing SQL Server licenses
- AHUB provides significant cost savings
- Choose model based on your license position

### 6. Can you run multiple SQL Server instances on one Managed Instance?
- No, one Managed Instance equals one SQL Server instance
- Create separate Managed Instance resources for multiple instances
- Each instance is independent with separate logins and databases
- Cannot consolidate multiple SQL instances into one resource

### 7. What is the maximum number of databases per Managed Instance?
- Up to 100 system and user databases per instance
- Performance depends on your tier and workload
- Exceeding 100 databases requires a separate instance
- Factor in backup and maintenance overhead

### 8. Does Managed Instance support Always On Availability Groups?
- Yes, built-in high availability with Always On
- Failover happens automatically within the instance cluster
- Local redundancy includes multiple replicas
- Geo-redundancy available through Availability Groups

### 9. What backup options are available?
- Automated backups with point-in-time restore (PITR)
- Default retention period of 7 to 35 days
- Long-term retention (LTR) available up to 10 years
- Geo-redundant backup storage by default

### 10. Can you connect to Managed Instance from on-premises?
- Yes, through VNet peering
- Site-to-site VPN connection supported
- ExpressRoute available for dedicated connectivity
- Direct RDP access to the instance is not available

### 11. What is the default authentication method?
- SQL Server logins and users created within instance
- Azure Active Directory (Azure AD) authentication also supported
- Mixed-mode authentication available
- Choose based on your identity management approach

### 12. Is Managed Instance HIPAA and SOC 2 compliant?
- Yes, meets HIPAA compliance standards
- SOC 2 Type II certified
- Microsoft publishes compliance documentation
- Attestations available for audit purposes

### 13. What retention period applies to automated backups?
- Backups retained for 7 to 35 days by default
- PITR retention setting controls this window
- Long-term retention extends to 10 years
- Retention cost increases with longer periods

### 14. Can you migrate directly from on-premises SQL Server to Managed Instance?
- Yes, use Azure Database Migration Service (DMS)
- Supports full migration of schema, data, and logins
- Minimal downtime migration possible
- One-way migration from on-premises to Azure

### 15. What is the SLA uptime guarantee?
- 99.99% uptime SLA for instances with at least one replica
- Default configuration includes multiple replicas
- Built-in high availability ensures this SLA
- Downtime only occurs during planned maintenance

---

## Section 2: Planning & Sizing (Questions 16-30)

### 16. What are the compute tiers available?
- General Purpose tier for most workloads
- Business Critical tier for latency-sensitive applications
- Business Critical requires high availability
- General Purpose is more cost-effective

### 17. What is the difference between General Purpose and Business Critical?
- General Purpose uses remote Azure Storage
- General Purpose handles variable latency
- Business Critical uses local NVMe SSD storage
- Business Critical provides lower latency and faster performance
- Business Critical costs significantly more
- Choose based on application latency requirements

### 18. How do you determine the right vCore count?
- Assess your on-premises CPU usage patterns
- Managed Instance vCores are comparable to SQL Server cores
- Benchmark your workload with a development instance
- Start with estimated vCore count and adjust based on testing
- Monitor CPU usage after deployment

### 19. What does reserved capacity mean?
- Prepay for compute for one or three years
- Receive discount on hourly rates
- Reduces cost if your usage is predictable
- Three-year commitments offer better discounts
- Best for stable, long-term workloads

### 20. Can you scale compute up or down without downtime?
- Yes, scaling causes a brief failover
- Failover typically takes under one minute
- Plan maintenance windows for scaling
- Use read replicas to minimize user impact
- Automatic failover to replica during scaling

### 21. What storage options are available?
- General Purpose uses managed Azure Storage
- Business Critical uses local NVMe SSD storage
- Storage provisioned in increments
- Scales automatically as needed
- Both tiers include automated backups

### 22. How is storage capacity calculated?
- Includes data files, log files, and tempdb
- Charged per GB provisioned
- Over-provisioning cheaper than frequent scaling
- Account for backup storage in long-term retention
- Monitor storage usage monthly

### 23. Can you exceed the maximum vCore limit per region?
- No, regional quotas apply
- Request quota increases through Azure support
- Request before you need additional capacity
- Quota increases take time to process
- Plan vCore requirements in advance

### 24. What is the impact of subnet size on Managed Instance deployment?
- VNet subnet needs sufficient free IP addresses
- Required for instance and Azure infrastructure
- A /25 subnet is minimum recommended
- Larger subnets provide growth flexibility
- Plan IP allocation before deployment

### 25. Can multiple Managed Instances share the same subnet?
- No, each instance requires dedicated subnet
- Instances cannot share the same /25 subnet
- Subnet must be within same VNet
- Each instance isolated in its subnet
- Better for network security and isolation

### 26. How many IP addresses does Managed Instance consume?
- Approximately 5 IPs reserved per instance
- Used for infrastructure and management
- Plan subnet with sufficient free IPs
- Avoid creating subnet with exactly needed IPs
- Leave buffer for Azure management operations

### 27. What happens if your subnet runs out of IP space?
- Can expand subnet if VNet has space
- If not, create new VNet with larger CIDR
- Migrate to new instance if expansion fails
- Plan subnets with growth in mind
- Monitor IP usage proactively

### 28. Can you change the VNet or subnet after deployment?
- No, VNet and subnet set at creation time
- Changing VNet requires migration
- Create new instance in target VNet
- Use backup/restore or SSMS for migration
- Plan network architecture before deployment

### 29. What is a management endpoint and how is it used?
- Azure-only endpoint for management operations
- Separate from application connection endpoint
- Used by Azure Portal and management tools
- Not directly accessible from on-premises
- Supports management plane operations

### 30. Can you move a Managed Instance to a different region?
- No direct move capability
- Create new instance in target region
- Migrate databases using backup/restore
- SSMS migration wizard available
- Consider geo-replica for disaster recovery

---

## Section 3: Deployment & Configuration (Questions 31-45)

### 31. What is the minimum time to deploy a new Managed Instance?
- Initial deployment typically takes 4 to 6 hours
- Plan accordingly for migrations
- First-time deployment in region takes longer
- Subsequent deployments may be faster
- Account for deployment time in migration planning

### 32. Can you automate Managed Instance creation?
- Yes, use Azure Resource Manager (ARM) templates
- Terraform support available
- Azure CLI can automate deployment
- PowerShell scripting for automation
- Infrastructure as Code (IaC) approach recommended

### 33. What is a collation and can you change it?
- Collation determines sort order and comparison rules
- Set at instance creation time
- Changing collation requires recreating the instance
- All databases inherit instance collation
- Collation affects string comparisons and sorting

### 34. What is the default collation for Managed Instance?
- Default is SQL_Latin1_General_CP1_CI_AS
- Choose different collation during creation if needed
- Collation impacts application compatibility
- Verify collation requirements before deployment
- Document collation for compliance purposes

### 35. Can you enable CLR in Managed Instance?
- Yes, CLR is supported and can be enabled
- Review security implications before enabling
- CLR allows running .NET code in SQL
- Disabled by default for security
- Enable only if your applications require it

### 36. Does Managed Instance support full-text search?
- Yes, full-text indexes are fully supported
- Works identically to on-premises SQL Server
- Enables keyword searching in text columns
- Improves search performance for text data
- Configure full-text catalogs as needed

### 37. What about R Services and Python extensions?
- R Services not supported in Managed Instance
- Python extensions not supported
- Use Azure Machine Learning alternatively
- SQL Server on VMs if in-database ML required
- Consider Synapse Analytics for data science workloads

### 38. Can you use FILESTREAM or FileTable?
- Both FILESTREAM and FileTable are supported
- Work identically to on-premises SQL Server
- Enables storing files in SQL database
- Integrates file and data management
- Plan storage requirements for file data

### 39. How do you configure SQL Agent?
- SQL Agent enabled by default
- Works same as on-premises SQL Server
- Create jobs through SSMS or T-SQL
- Schedule jobs for automated tasks
- Monitor job history and failures

### 40. Can you use SQL Server Integration Services (SSIS)?
- SSIS packages run in Azure-SSIS Integration Runtime
- Runtime hosted within Azure Data Factory
- Managed Instance can be SSIS target
- Deploy packages to Azure-SSIS runtime
- Schedule SSIS execution in Data Factory

### 41. What about SQL Server Reporting Services (SSRS)?
- SSRS not hosted in Managed Instance
- Deploy Power BI separately for reporting
- Power BI connects to Managed Instance
- Reporting Services available on VMs
- Consider cloud-native reporting solutions

### 42. Can you run Analysis Services in Managed Instance?
- Analysis Services not supported in Managed Instance
- Use Azure Analysis Services separately
- Consider Power BI for analytics
- Synapse Analytics alternative for large scale
- Deploy Analysis Services on Azure VMs if needed

### 43. How do you set up notifications and alerts?
- Use Azure Monitor for metrics
- Action Groups deliver alerts
- Send alerts based on diagnostic logs
- Email notifications available
- Integration with incident management tools

### 44. Can you configure email notifications from SQL Agent?
- Yes, but requires SMTP relay configuration
- Direct internet SMTP not available
- Configure relay through Azure VMs
- External services available for SMTP
- SendGrid or similar services supported

### 45. What is the tempdb configuration?
- tempdb automatically configured and managed
- Cannot customize its location directly
- Cannot adjust size directly
- Azure manages tempdb resources
- Increase tempdb files for concurrency improvements

---

## Section 4: Networking & Security (Questions 46-60)

### 46. Is Managed Instance publicly accessible from the internet?
- No, not by default
- Only accessible within VNet by default
- Must configure public endpoint for external access
- Private endpoint recommended for security
- Network controls restrict access

### 47. What is a public endpoint?
- Allows connections from public internet
- Controlled through network rules
- Firewall rules restrict IP access
- Adds internet exposure risk
- Use only when necessary

### 48. What is a private endpoint and how does it help?
- Connects applications over Azure backbone network
- Bypasses public internet routing
- More secure than public endpoints
- Better for internal applications
- Reduces attack surface

### 49. Can you have both public and private endpoints simultaneously?
- Yes, both can be enabled together
- Control traffic separately through firewall rules
- Use public for external apps
- Use private for internal apps
- Manage both independently

### 50. How do you configure firewall rules?
- Use Azure Portal to add rules
- T-SQL configuration available
- IP-based firewall rules control access
- Specify allowed IP ranges
- Rules apply to public endpoint

### 51. Does Managed Instance support Network Security Groups (NSGs)?
- NSGs don't apply directly to Managed Instance
- Configure firewall rules within Managed Instance
- Firewall rules provide network control
- Use Managed Instance firewall instead of NSGs
- NSGs applied at subnet level don't restrict access

### 52. What ports does Managed Instance use?
- Port 1433 is standard SQL port
- Port 1434 used for named pipes
- Only public endpoint exposes to internet
- Private endpoint uses private routing
- Configure firewall for these ports

### 53. Can you change the default port?
- No, port 1433 is fixed
- Cannot customize connection port
- All instances use port 1433
- Applications must connect on default port
- No alternative port available

### 54. Does Managed Instance support Transparent Data Encryption (TDE)?
- Yes, TDE is supported
- Encrypts database file at rest
- Enabled by default
- Uses Azure Key Vault for key storage
- Automatic key rotation available

### 55. Can you bring your own encryption key (BYOK)?
- Yes, customer-managed keys supported
- Configure keys in Azure Key Vault
- Enable TDE with your key
- Maintain key ownership and control
- Rotate keys independently

### 56. What is Always Encrypted and is it supported?
- Always Encrypted encrypts columns client-side
- Fully supported in Managed Instance
- Data encrypted before sending to database
- Decryption happens in application
- Best for highly sensitive data

### 57. Can you use Azure AD for authentication?
- Yes, create Azure AD logins
- Azure AD users supported
- Managed identities for application auth
- MFA compatible with Azure AD
- Eliminate SQL password management

### 58. What about Azure AD multi-factor authentication?
- MFA supported for Azure AD authentication
- Interactive SSMS connections work with MFA
- Applications with MFA libraries supported
- Enhanced security for privileged access
- Conditional Access policies can apply

### 59. Can you audit database access?
- Yes, enable SQL Audit
- Logs connection attempts
- Captures T-SQL statements
- Logs user activities
- Audit trails stored in storage or Event Hubs

### 60. Does Managed Instance support row-level security (RLS)?
- Yes, fully supported
- Works identically to SQL Server
- Filters rows based on user or role
- Transparent to applications
- Enhances data privacy

---

## Section 5: High Availability (Questions 61-75)

### 61. What high availability features are built into Managed Instance?
- Local redundancy with multiple replicas included by default
- Always On Availability Groups for distributed HA
- Automatic failover within instance cluster
- Geo-redundancy for disaster recovery
- No additional configuration required for local HA

### 62. How many replicas does Managed Instance maintain?
- General Purpose tier typically has 4 replicas
- Business Critical tier typically has 3 replicas
- Microsoft manages replica count automatically
- Failover handled by Azure
- Replicas transparent to applications

### 63. How long does a failover take?
- Automatic failover takes 30 seconds to few minutes
- Time depends on failure type
- Local failover faster than geo-failover
- Plan for brief connectivity loss
- Applications need reconnection handling

### 64. Can you control which replica becomes primary during failover?
- No, Azure manages replica selection automatically
- Cannot specify preferred replica
- Failover happens transparently
- No administrator intervention needed
- System chooses optimal replica

### 65. What is an Availability Group in Managed Instance?
- Allows distributed replicas across multiple instances
- Supports disaster recovery across regions
- Create secondary replicas for failover
- Asynchronous replication available
- Read-only secondary replicas supported

### 66. Can you have multiple Availability Groups on one instance?
- Yes, multiple AGs on same instance
- Each AG manages its databases independently
- Configure separate secondary replicas per AG
- Distribute workloads across multiple AGs
- Each AG has separate failover group

### 67. What is the difference between local redundancy and geo-redundancy?
- Local redundancy keeps replicas in same region
- Geo-redundancy maintains replica in different region
- Local redundancy for high availability
- Geo-redundancy for disaster recovery
- Both can be implemented together

### 68. How do you set up a geo-replica for disaster recovery?
- Create Availability Group on primary instance
- Add secondary replica in another region
- Configure asynchronous replication
- Secondary replica is read-only
- Manual failover to secondary if needed

### 69. What is the RPO (Recovery Point Objective) for geo-replicas?
- RPO depends on replication lag
- Typically few seconds to minutes
- Network latency affects RPO
- Asynchronous replication may lose recent transactions
- Monitor transaction log queue length

### 70. Can applications automatically failover to a geo-replica?
- No, applications must detect failover
- Implement application-level retry logic
- Use Availability Group listener DNS names
- Connection strings should support multiple endpoints
- Monitor connection state in applications

### 71. What is the Recovery Time Objective (RTO) for failover?
- RTO within region typically 1 to 2 minutes
- RTO for geo-failover depends on manual promotion
- Automatic failover faster than manual
- Network latency affects geo-failover speed
- Plan for worst-case RTO scenarios

### 72. Can you have read-only replicas for reporting?
- Yes, secondary replicas are read-only
- Direct reporting queries to secondary
- Reduces load on primary instance
- Query performance depends on replication lag
- Monitor secondary replica for bottlenecks

### 73. Do read replicas incur separate costs?
- Yes, each replica charged separately
- Each replica is full Managed Instance
- Budget for multiple instances
- Read replicas not free feature
- Costs multiply with each additional replica

### 74. Can you add or remove replicas without downtime?
- Modifying Availability Group causes brief interruption
- Not true zero-downtime operation
- Connectivity loss measured in seconds
- Data not lost during modification
- Plan for maintenance windows

### 75. What is the maximum number of replicas in an Availability Group?
- Up to 9 total replicas across instances
- Includes primary plus 8 secondaries
- Distributed across multiple Managed Instances
- Each replica is separate instance
- Plan costs for multiple instances

---

## Section 6: Backup & Disaster Recovery (Questions 76-85)

### 76. Are backups automatic?
- Yes, automated backups run continuously
- Backup files stored in Azure managed storage
- No manual backup configuration needed
- Backups happen transparently
- Storage automatically managed by Azure

### 77. Where are backup files stored?
- Stored in geo-redundant Azure Storage (GRS) by default
- Copies exist in paired region
- Accessible for disaster recovery
- Separate from primary database storage
- Long-term retention uses separate storage account

### 78. Can you disable automatic backups?
- No, backups are mandatory
- Cannot be disabled or bypassed
- Required for data protection
- No option to reduce backup frequency
- Included in service offering

### 79. How long can you restore to a specific point in time?
- Default retention 7 to 35 days
- PITR retention setting controls window
- Extend long-term retention to 10 years
- Longer retention increases cost
- Retention applies to all databases

### 80. What is long-term backup retention (LTR)?
- Allows keeping weekly backups for 10 years
- Monthly backups available
- Yearly backups for long-term archival
- Useful for compliance requirements
- Separate storage charges apply

### 81. How do you perform a point-in-time restore?
- Use Azure Portal for GUI restore
- PowerShell scripting available
- SSMS supports restore operations
- Specify exact restore timestamp
- Choose target database name

### 82. Can you restore to a different Managed Instance?
- Yes, restore to any instance in subscription
- Cross-region restore supported
- Useful for disaster recovery
- Target instance must have space
- Restore creates new database on target

### 83. What is the restore time for a large database?
- Depends on database size
- Recovery model impacts restore time
- Multi-TB databases take hours
- Network bandwidth affects restore speed
- Plan downtime accordingly

### 84. Do restored databases consume additional storage?
- Yes, each restored database consumes storage
- Counts toward instance storage quota
- Plan storage capacity before restore
- Restored databases are independent copies
- Delete unwanted restores to free space

### 85. Can you restore individual tables or objects?
- No, restore is database-level only
- Cannot selectively restore tables
- Entire database restored together
- Use table-level tools for selective recovery
- Backup and restore cannot target objects

---

## Section 7: Performance Tuning (Questions 86-95)

### 86. How do you identify slow queries?
- Use built-in Query Store
- Extended Events available
- SQL Profiler still supported
- Query Store requires minimal configuration
- Provides execution metrics automatically

### 87. What is Query Store and how does it help?
- Captures query execution plans
- Records runtime statistics
- Identifies performance regressions
- Tracks query behavior over time
- Recommends query performance improvements

### 88. Can you use SQL Server Management Studio (SSMS) to monitor performance?
- Yes, SSMS provides execution plans
- Activity Monitor shows active queries
- Stored procedures for performance checks
- Remote monitoring fully supported
- Connect via public or private endpoint

### 89. What metrics should you monitor?
- Monitor CPU percentage usage
- Track memory usage trends
- Storage IOPS measurements
- Network throughput monitoring
- Use Azure Monitor and sys.dm_db_resource_stats

### 90. How do you index a table for performance?
- Analyze query patterns first
- Create clustered indexes on primary keys
- Add non-clustered indexes on frequent columns
- Use sys.dm_db_missing_index_details
- Consolidate overlapping indexes

### 91. Can you rebuild indexes without downtime?
- Yes, online index rebuild supported
- Allows concurrent queries during rebuild
- No blocking during maintenance
- Slightly higher CPU during rebuild
- Test rebuild windows on dev environment

### 92. What is tempdb contention and how do you reduce it?
- Tempdb contention occurs with high concurrency
- Increase number of tempdb files
- Reduces contention on GAM pages
- File count should match vCore count
- Configure during instance creation

### 93. How do you optimize T-SQL queries?
- Examine execution plans
- Identify expensive operations
- Eliminate table scans where possible
- Use appropriate join methods
- Parameterize queries to improve caching

### 94. Can you use query hints to improve performance?
- Yes, hints like NOLOCK available
- INDEX hints for specific index usage
- RECOMPILE forces plan recompilation
- Test hints thoroughly
- Hints can negatively impact other queries

### 95. What is the impact of increasing vCores on query performance?
- More vCores provide additional CPU capacity
- Increases available memory
- Supports more concurrent query execution
- Improves parallel processing
- Always measure workload before and after scaling

---

## Section 8: Real-time Scenarios (Questions 96-150)

### 96. Your application suddenly experiences slow response times. What's the first step?
- Check sys.dm_exec_requests for active queries
- Query Store shows current execution plans
- Compare current vs historical baselines
- Identify recently changed queries
- Monitor CPU and memory usage

### 97. How do you diagnose high CPU usage on Managed Instance?
- Query sys.dm_exec_query_stats for CPU stats
- Use sys.dm_db_resource_stats for trends
- Identify CPU-intensive queries
- Check for missing indexes
- Review poorly written queries

### 98. Your instance ran out of storage. What do you do?
- Managed Instance automatically scales storage
- Scaling up to tier maximum happens automatically
- Contact support if near absolute limit
- Scale to larger instance type if needed
- Monitor storage usage proactively

### 99. A table became corrupted. How do you recover?
- Use point-in-time restore to recover
- Restore to moment before corruption
- DBCC REPAIR_ALLOW_DATA_LOSS not available
- Recover using backup restoration
- Schedule regular consistency checks

### 100. Your failover is taking longer than expected. Why?
- Large transactions slow failover
- Long-running queries prevent fast failover
- Uncommitted changes block failover
- Commit transactions before maintenance
- Avoid long transactions during failover windows

### 101. Can you migrate from SQL Server 2008 or earlier?
- Yes, migration possible
- SQL Server 2008 is unsupported on Azure
- Upgrade on-premises to 2016 or later first
- Then migrate to Managed Instance
- Plan upgrade timeline before migration

### 102. How do you handle legacy code that uses deprecated features?
- Use Upgrade Advisor to identify features
- Refactor code before migration
- Test thoroughly on dev instance
- Rewrite deprecated code
- Plan code refactoring timeline

### 103. A user accidentally deleted critical data. How do you recover?
- Use point-in-time restore for recovery
- Restore to moment before deletion
- Extract deleted data from restored database
- Copy data back to production database
- Implement access controls to prevent deletion

### 104. Your instance is approaching the maximum number of databases. What happens?
- Maximum of 100 databases per instance
- Exceeding 100 requires new instance
- Create second Managed Instance
- Plan database consolidation strategy
- Monitor database count regularly

### 105. Can you run two different versions of SQL Server side by side in Managed Instance?
- No, one instance has one version
- All databases use same instance version
- Different compatibility levels supported
- Set compatibility level per database
- Upgrade database compatibility separately

### 106. Your migration is failing midway. What should you check?
- Verify source and destination compatibility
- Check network connectivity
- Confirm sufficient disk space
- Review Database Migration Service logs
- Check for unsupported features

### 107. After migration, your application connects but runs slowly. Why?
- Statistics may be outdated
- Run UPDATE STATISTICS on all tables
- Check for parameter sniffing
- Identify missing indexes
- Compare execution plans with source

### 108. Can you batch-load millions of records efficiently?
- Use bulk insert for high performance
- SSIS available for data loading
- INSERT...SELECT from source table
- Disable triggers temporarily
- Use BATCHSIZE and TABLOCK options

### 109. Your scheduled backup failed. What's the cause?
- Automated backups rarely fail
- Check if instance is running
- Monitor for resource constraints
- Review backup logs in Azure Portal
- Verify storage account permissions

### 110. Can you export Managed Instance data to Azure Blob Storage?
- Yes, use BCP for export
- SSIS supports data export
- CREATE EXTERNAL TABLE option available
- Configure storage account permissions
- Encrypt data in transit and at rest

### 111. You need to share a database snapshot with another team. How?
- Database snapshots not supported in Managed Instance
- Restore database copy to another instance
- Create read-only replica for sharing
- Use Availability Group secondary replica
- Alternative is geo-replica for isolation

### 112. Your Availability Group replica is lagging. What causes this?
- High transaction volume causes lag
- Network latency delays replication
- Monitor redo queue length metrics
- Check network bandwidth availability
- Check disk I/O on secondary replica

### 113. Can you run multiple concurrent workloads on one instance?
- Yes, multiple workloads supported
- Workloads compete for resources
- Resource Governor not available
- Use separate instances for isolation
- Separate instances provide guaranteed resources

### 114. How do you prevent concurrent data modifications to the same table?
- Use pessimistic locking WITH NOLOCK UPDLOCK
- Optimistic concurrency with row versions
- Design applications for concurrency
- Reduce lock contention through design
- Monitor blocking and deadlocks

### 115. Your application uses local accounts and logins. How do you migrate them?
- Extract logins using TSQL scripts
- Recreate logins on Managed Instance
- Transfer password hashes securely
- Reset passwords post-migration
- Test login functionality after migration

### 116. Can you modify system databases like master or msdb?
- Yes, system database modification supported
- Be cautious with changes
- Affects instance behavior
- Test changes in dev environment
- Document all modifications

### 117. What happens if you change the recovery model on a user database?
- Changes take effect immediately
- Full recovery enables transaction log backups
- Simple recovery truncates logs automatically
- Cannot restore to point in time in Simple mode
- Full recovery requires regular log backups

### 118. How do you compress tables to save storage?
- Use data compression on tables
- Compress indexes separately
- Reduces storage footprint
- Improves query performance
- Increases CPU usage during compression

### 119. Your transaction log is growing rapidly. How do you manage it?
- In Full recovery, take log backups frequently
- In Simple recovery, logs truncate automatically
- Commit transactions to truncate
- Monitor log file size
- Plan for log growth

### 120. Can you use replication between Managed Instance and SQL Server on-premises?
- Transactional replication available
- Managed Instance as publisher or subscriber
- Network latency affects replication
- Test replication thoroughly
- Monitor latency and bandwidth

### 121. How do you handle identity column conflicts during merge replication?
- Use identity ranges to avoid conflicts
- GUID columns alternative to identity
- Assign ranges to each instance
- Test conflict resolution
- Monitor for identity exhaustion

### 122. Your development environment needs to be refreshed with production data. How?
- Restore production backup to dev instance
- Ensure data anonymization for GDPR
- Remove sensitive data from dev
- Create separate dev instance
- Schedule regular refresh cycles

### 123. Can you upgrade a database from an older compatibility level?
- Yes, use ALTER DATABASE command
- Test applications after upgrade
- Verify query performance after upgrade
- Monitor for regressions
- Plan upgrade timeline

### 124. How do you validate data integrity after migration?
- Run SELECT COUNT(*) comparisons
- Verify constraints validation
- Check referential integrity
- Use RedGate SQL Compare tool
- Document validation results

### 125. Your application uses Windows authentication. Does Managed Instance support it?
- Windows auth not supported over public endpoints
- Use Azure AD authentication instead
- SQL logins available for applications
- Managed identities for Azure services
- Mixed-mode auth not supported for cloud

### 126. Can you implement time-based access restrictions?
- Use application-level logic
- Azure AD Conditional Access policies
- T-SQL triggers log unauthorized attempts
- Implement via firewall rules
- Document access restriction rules

### 127. How do you handle schema versioning across deployments?
- Track migration scripts in version control
- Deploy scripts in sequence
- Use Azure Pipelines for automation
- Manual runbooks alternative
- Test scripts in dev first

### 128. Your backup restore failed due to database compatibility mismatch. How do you fix?
- Restore to same or newer SQL Server version
- Use RESTORE with MOVE option
- Change file paths if needed
- Verify backup file integrity
- Check target instance compatibility

### 129. Can you monitor backup completion in real time?
- Query msdb.dbo.backupset for completion
- Azure Monitor tracks operations
- Automation alerts notify failures
- Check backup history
- Monitor backup duration trends

### 130. How do you securely share database access with external vendors?
- Create limited-access logins
- Specific database permissions only
- Restrict to object level
- Use IP-based firewall rules
- Private endpoints for external vendors

### 131. Your instance connectivity is intermittent. What should you investigate?
- Check network connectivity status
- Verify VPN status if using VPN
- Review firewall rules
- Verify public endpoint enabled
- Check private endpoint configuration

### 132. Can you upgrade compute during peak hours?
- Yes, you can upgrade anytime
- Upgrade triggers brief failover
- Schedule during maintenance windows
- Minimize impact on users
- Failover typically takes under 1 minute

### 133. How do you forecast storage growth?
- Monitor monthly storage increase
- Project growth forward quarterly
- Factor in backup growth
- Account for log retention
- Plan storage scaling timeline

### 134. Your backup retention policy changed. How do you adjust?
- Update settings in Azure Portal
- Changes apply to new backups
- Existing backups follow old rules
- Long-term retention policy separate
- Document retention policy changes

### 135. Can you migrate a sharded database to Managed Instance?
- Yes, migration possible
- Migrate each shard separately
- Use federation features if needed
- Create separate database per shard
- Plan cross-shard query consolidation

### 136. How do you handle application connection pooling?
- Configure pool size by workload
- Monitor connection counts
- Use sys.dm_exec_sessions
- Tune pool parameters
- Monitor connection wait times

### 137. Your application crashes when the database fails over. Why?
- Application not handling timeouts
- Implement retry logic
- Use exponential backoff
- Increase connection timeout
- Test failover scenarios

### 138. Can you run DBCC commands on Managed Instance?
- Yes, most DBCC commands supported
- Some administrative commands restricted
- CHECKDB fully supported
- REPAIR commands available
- Test DBCC on non-production first

### 139. How do you check database consistency?
- Run DBCC CHECKDB regularly
- Schedule during low-traffic periods
- DBCC consumes resources
- Log consistency check results
- Investigate any errors immediately

### 140. Your data contains PII. How do you ensure compliance?
- Use Always Encrypted
- Enable Transparent Data Encryption
- Implement column-level security
- Audit all data access
- Implement row-level security

### 141. Can you disable certain advanced features to save cost?
- Pricing based on compute and storage
- Features included in pricing
- Disabling features doesn't reduce cost
- All features always available
- Budget based on compute and storage only

### 142. How do you migrate SQL Server Agent jobs?
- Export jobs using TSQL scripts
- SQL Server Management Studio export
- Recreate jobs on Managed Instance
- Test job execution after migration
- Adjust job schedules if needed

### 143. Your nightly job failed. How do you diagnose?
- Check SQL Agent job history in SSMS
- Review job step output
- Check error messages
- Enable job notifications
- Use SQL Agent mail for alerts

### 144. Can you automate daily maintenance tasks?
- Yes, use SQL Agent jobs
- Schedule index maintenance
- Schedule statistics updates
- Schedule consistency checks
- Schedule backup verification

### 145. How do you validate network connectivity to Managed Instance?
- Use telnet to test connectivity
- SSMS connection test
- Verify firewall rules
- Check endpoint configuration
- Verify VNet routing

### 146. Your application pool size is undersized. What happens?
- Connections queue and wait
- Requests experience latency
- Increase pool size
- Monitor connection wait times
- Use dm views for monitoring

### 147. Can you use Managed Instance for development, test, and production simultaneously?
- Yes, create separate instances
- Use separate databases per tier
- Isolate workloads
- Different firewall rules per tier
- Cost scales with instance count

### 148. How do you estimate Managed Instance cost?
- Use Azure Pricing Calculator
- Factor compute vCores
- Account for storage
- Include backup retention
- Consider reserved instances discount

### 149. Your team needs read-only access to production data. How?
- Create read-only database user roles
- Direct traffic to geo-replicas
- Use Always On secondary replicas
- Configure view-level access
- Monitor read replica performance

### 150. What's the best strategy for disaster recovery planning?
- Implement automated backups
- Enable long-term retention
- Maintain geo-replica Availability Groups
- Test restore procedures quarterly
- Document failover runbooks
- Test failover annually
- Keep runbooks updated

---

## Quick Reference

Use these FAQs to plan Managed Instance deployments, troubleshoot issues, and optimize performance. For specific scenarios, consult Microsoft documentation or Azure support.

Key Takeaways:

- Managed Instance is fully managed PaaS offering
- High availability included by default
- Plan VNet sizing before deployment
- Allocate sufficient subnet IP space
- Choose General Purpose or Business Critical tier
- Monitor performance using Query Store
- Use Azure Monitor for metrics
- Implement backup strategies from day one
- Enable long-term retention for compliance
- Use private endpoints for security
- Test migrations thoroughly
- Document all procedures
- Leverage Availability Groups for DR
- Maintain geo-replicas for resilience
- Schedule regular failover testing
- Keep runbooks updated
- Audit all access to sensitive data
- Use encryption for data protection
- Implement retry logic in applications
- Scale compute during maintenance windows
