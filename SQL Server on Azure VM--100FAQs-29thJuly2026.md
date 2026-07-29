# SQL Server on Azure VM: Comprehensive FAQ Guide

Complete reference for deploying, managing, and optimizing SQL Server on Azure Virtual Machines.

---

## SECTION 1: BASICS (Q1-15)

### Understanding SQL Server on Azure VM

**Q1: What is SQL Server on Azure VM?**
- Fully managed database service running SQL Server on Azure Virtual Machines
- You control OS, SQL Server version, and configuration
- Azure handles underlying infrastructure
- Sits between self-managed on-premises and fully managed Azure SQL Database
- Provides balance between control and managed services

**Q2: How does SQL Server on Azure VM differ from Azure SQL Database?**
- Azure VM gives full SQL Server features and complete control
- Azure SQL Database is Platform-as-a-Service with less control
- Azure SQL Database provides automatic patching, scaling, and backup
- Choose Azure VM for specific features, legacy versions, or full configuration control
- Azure SQL Database better for new applications without special requirements

**Q3: What are the key benefits of SQL Server on Azure VM?**
- Cost efficiency through pay-as-you-go or hybrid licensing options
- Lift-and-shift migrations from on-premises require minimal code changes
- Full SQL Server compatibility with enterprise features like Always On
- Supports Failover Clustering and other advanced features
- Integration with Azure services for monitoring, backup, and disaster recovery
- Windows Server included with SQL Server licensing

**Q4: Which SQL Server versions run on Azure VMs?**
- SQL Server 2022 is the latest version
- SQL Server 2019, 2016, and 2012 are commonly used
- Older versions like 2008 R2 have limited support
- Check Microsoft support lifecycle for end-of-support dates
- Plan upgrades before reaching end-of-support

**Q5: What licensing requirements exist for SQL Server on Azure VMs?**
- SQL Server requires CAL (Client Access License) for Standard and Enterprise editions
- Marketplace pre-installed images include licensing in hourly VM cost
- Bring your own license option if you have active Software Assurance agreements
- Express edition is free with 10GB database limit
- License compliance is critical for cost management

### Licensing Models

**Q6: What is Pay-as-you-go licensing?**
- You pay hourly rate including both Windows Server and SQL Server licensing
- No upfront costs or long-term commitments required
- Costs are higher per unit but offer flexibility
- Best for variable or unpredictable workloads
- Allows month-to-month adjustments without penalty

**Q7: How does Azure Hybrid Benefit work?**
- Use existing on-premises SQL Server licenses with active Software Assurance
- Pay only base Azure VM infrastructure cost without SQL Server licensing premium
- Potential savings of 40-50% on VM costs
- Requires proper license tracking and compliance audits
- Licensing agreements must be current and valid

**Q8: What are Reserved Instances?**
- Commit to 1 or 3 year terms for significant discounts
- Up to 72% discount off on-demand pricing with 3-year commitment
- Pay upfront or monthly for the commitment
- Best for stable, predictable workloads
- Reduces overall cost of ownership compared to pay-as-you-go

**Q9: How do you calculate total licensing cost?**
- Add VM infrastructure cost plus SQL Server licensing cost
- Pay-as-you-go includes both costs in hourly rate
- Hybrid Benefit only charges VM infrastructure costs
- Reserved Instances divide committed costs by month count
- Factor in storage, backup, and data transfer costs
- Use Azure Cost Calculator for accurate estimates

**Q10: Can you switch between licensing models?**
- You can upgrade from pay-as-you-go to Reserved Instances
- Switching from Hybrid Benefit to pay-as-you-go requires VM redeployment
- Plan licensing model during initial deployment
- Changing models after deployment causes downtime
- Lock in optimal model before production deployment

### Support and Cost Comparison

**Q11: What support options are available?**
- Basic support is free but limited to account and billing issues
- Developer support for non-production environments
- Standard support includes 8/5 business hours response
- Professional Direct support includes 24/7 technical support
- Premier support offers strategic advisory and 15-minute response time
- Choose based on workload criticality and business requirements

**Q12: How does cost compare with on-premises SQL Server?**
- On-premises requires capital expenditure for hardware, OS, licenses
- Azure VMs eliminate hardware costs but require compute and storage fees
- Break-even typically occurs at 3-4 years of use
- Azure scales costs linearly with usage
- On-premises has fixed costs regardless of utilization
- Small workloads favor Azure; large stable workloads may favor on-premises

**Q13: What is the lifecycle and support timeline for SQL Server versions?**
- SQL Server 2012 and earlier are out of mainstream support
- SQL Server 2016 ends mainstream support in 2021, extended in 2026
- SQL Server 2019 has extended support until 2030
- SQL Server 2022 has extended support until 2033
- Plan upgrades well before end-of-support dates
- Unsupported versions receive no security patches

**Q14: Are there hidden costs when running SQL Server on Azure?**
- Outbound data transfer charges apply to external traffic
- Backup storage adds cost based on retention period
- Premium storage costs more than standard disks
- Monitoring and diagnostics have minimal fees
- Unexpected scaling or high transaction volume increases costs
- Set budget alerts to detect spending anomalies

**Q15: What free resources help with SQL Server on Azure VMs?**
- Azure documentation is free and comprehensive
- SQL Server best practices guides available online
- Microsoft Learn provides free training modules
- Azure Portal offers cost analysis tools
- Community forums and Stack Overflow provide peer support
- Microsoft documentation covers common scenarios

---

## SECTION 2: PLANNING & SIZING (Q16-30)

### Assessing Workload Requirements

**Q16: How do you assess SQL Server workload requirements?**
- Review current on-premises performance metrics including CPU, memory, disk
- Analyze query patterns and peak load times
- Examine database sizes and growth trends
- Identify business SLA requirements for uptime and recovery time
- Document current backup and maintenance windows
- Establish baseline performance metrics for comparison

**Q17: What metrics should you gather from existing SQL Server installations?**
- Average and peak CPU utilization
- Memory consumption during peak hours
- Disk read/write IOPS and throughput
- Active connection count and transaction log growth rate
- Query execution times for critical queries
- Wait statistics and blocking patterns
- Backup and restore duration
- Network bandwidth usage

**Q18: How do you estimate storage requirements?**
- Add current database size plus 12-24 months of projected growth
- Reserve 20-30% headroom for index fragmentation
- Account for temporary objects and staging tables
- Calculate transaction log size based on workload
- Include backup storage calculations
- Monitor historical growth trends
- Plan for worst-case scenarios

**Q19: What is rightsizing and why is it important?**
- Rightsizing matches VM and storage resources to actual workload needs
- Oversizing increases costs unnecessarily without performance benefit
- Undersizing causes performance issues and poor user experience
- Review actual usage after 2-4 weeks of production
- Adjust resources based on observed performance
- Prevents waste while maintaining performance

**Q20: How do you handle variable or unpredictable workloads?**
- SQL Server on VM scaling is manual, not automatic
- Consider reserved capacity for baseline needs
- Use pay-as-you-go for spike capacity
- Implement query optimization to reduce resource consumption
- Use read replicas for reporting to isolate workloads
- Plan for peak demand rather than average demand

### Storage Selection and Sizing

**Q21: What are the differences between Standard and Premium storage?**
- Standard storage uses HDDs with lower cost
- Standard storage has higher latency and variable IOPS
- Premium storage uses SSDs with guaranteed IOPS
- Premium storage has lower and predictable latency
- Premium is required for production workloads
- Standard works for development and testing environments
- Premium costs 5-10x more than Standard

**Q22: What is Ultra Disk storage and when should you use it?**
- Ultra Disk provides highest performance tier
- Guaranteed sub-millisecond latency and up to 160K IOPS
- Use for mission-critical workloads requiring extreme performance
- Cost is higher than Premium storage
- Requires specific VM SKU support for provisioning
- Best for high-frequency trading, real-time analytics

**Q23: How do you calculate storage IOPS requirements?**
- Determine peak concurrent transactions from workload analysis
- Each transaction typically generates 2-5 IOPS
- Peak IOPS equals peak transactions multiplied by average IOPS per transaction
- Add 20-30% buffer for traffic spikes
- Match disk type to calculated requirements
- Test actual IOPS with production workload simulation

**Q24: What is disk caching and how does it affect performance?**
- Premium and Ultra Disks support read and write caching
- Read caching stores frequently accessed data in cache
- Write caching buffers writes before persisting to disk
- ReadOnly caching works well for data files
- WriteThrough or NoCache is safer for transaction logs to ensure durability
- Improves performance but adds risk if not configured correctly

**Q25: How many data disks should you attach?**
- One disk minimum for OS and SQL Server binaries
- Separate data, log, and tempdb onto different disks
- Add backup disk if storing backups locally
- High-traffic databases benefit from multiple disks striped across RAID 10
- Generally 3-5 total disks for most workloads
- More disks allow better isolation and performance

### Network and Regional Planning

**Q26: What bandwidth do you need for database replication?**
- Availability Group replication bandwidth proportional to transaction volume
- Plan 1-10 Mbps minimum for typical workloads
- High-throughput transactional systems may need 100+ Mbps
- Replication is asynchronous by default
- Temporary network delays are tolerable in async mode
- Monitor actual bandwidth usage and provision accordingly

**Q27: How does region selection affect performance and cost?**
- Regions close to users reduce network latency
- Different regions have different pricing structures
- Some regions have limited VM SKU availability
- Plan for disaster recovery when selecting regions
- Use cross-region availability for critical workloads
- Document region selection rationale for future reference

**Q28: What network redundancy options exist?**
- Azure provides built-in network redundancy within regions
- Multiple datacenters protect against single-datacenter failure
- For cross-region redundancy, deploy to secondary regions
- Use Azure Traffic Manager for multi-region failover
- Implement Network Security Groups for redundant filtering
- Network redundancy is automatic within Azure regions

**Q29: How do you handle multi-region deployments?**
- Replicate databases using Always On across regions
- Use Azure Data Sync for eventual consistency
- Implement read replicas in secondary regions for reporting
- Design for asymmetric traffic with writes in primary region
- Plan for network costs of cross-region traffic
- Test failover across regions before production

**Q30: What are network latency requirements for Always On and clustering?**
- Availability Groups tolerate up to 5 seconds latency
- Failover Cluster Instances require sub-100ms latency
- Same-region deployment is recommended for clustering
- Cross-region Always On adds recovery time
- Asynchronous replicas acceptable across regions
- Same-region synchronous replicas require low latency

---

## SECTION 3: DEPLOYMENT & CONFIGURATION (Q31-45)

### Creating and Initializing SQL Server VMs

**Q31: What are the steps to create a SQL Server VM from Azure Marketplace?**
- Select SQL Server image from Azure Marketplace with desired version
- Choose appropriate VM SKU based on sizing assessment
- Configure networking including vnet and subnet
- Set administrator credentials securely
- Select storage type (Premium or Standard)
- Review SQL Server configurations
- Deploy and wait 5-15 minutes for completion
- Verify deployment before proceeding

**Q32: What initial setup is required after VM deployment?**
- Update Windows Server and SQL Server patches immediately
- Configure Windows Firewall rules for SQL Server port 1433
- Join domain if using Windows Authentication
- Create SQL Server logins and database roles
- Configure backup storage location
- Set up SQL Server Agent for maintenance jobs
- Test connectivity from client machines

**Q33: Should you install SQL Server from scratch or use prebuilt images?**
- Prebuilt images are faster to deploy with SQL Server pre-configured
- Use prebuilt images for standard configurations
- Install from scratch for specific SQL Server features
- Install from scratch for non-standard configurations
- Prebuilt images reduce human error in setup
- Prebuilt images save 1-2 hours of installation time

**Q34: What is the difference between SQL Server Express, Standard, and Enterprise editions?**
- Express is free with 10GB database limit, 1GB memory, 1 CPU
- Standard supports 128GB memory, 24 CPU cores, includes most features
- Enterprise has no limits and includes all advanced features
- Express lacks Always On and clustering capabilities
- Choose based on workload complexity and business requirements
- Standard adequate for most mid-market organizations

**Q35: How do you configure SQL Server during initial deployment?**
- Enable SQL Server Agent for automated maintenance
- Configure tempdb for optimal performance
- Set Max Server Memory to leave 1-2GB for OS
- Enable instant file initialization for faster file growth
- Configure backup paths and retention policies
- Set up login security and database roles
- Configure SQL Server startup parameters

### Configuration Best Practices

**Q36: What is the best practice for tempdb placement?**
- Place tempdb on fast local SSD or Premium Disk
- Separate tempdb from data and log files
- Size tempdb to accommodate peak concurrent sessions
- Tempdb contention causes performance bottlenecks if misconfigured
- Monitor tempdb size and clear during maintenance windows
- For multiple cores, create multiple tempdb data files
- One log file for tempdb is sufficient

**Q37: How should you organize data, log, and backup files?**
- Place data files on one Premium Disk
- Place transaction logs on separate Premium Disk with WriteThrough caching
- Store backups on another disk or Azure Storage
- Separate disks reduce I/O contention
- Separate disks improve performance and recoverability
- Follow consistent naming conventions for clarity
- Document file locations for team reference

**Q38: What disk caching configuration is optimal for SQL Server?**
- Use ReadOnly for data file disks to cache frequently accessed data
- Use NoCache or WriteThrough for transaction log disks
- WriteThrough or NoCache ensures durability for transaction logs
- Use NoCache for backup disks
- Avoid ReadWrite caching for production environments
- ReadWrite caching risks data loss during failure
- Caching configuration prevents corruption and data loss

**Q39: What are optimal SQL Server memory settings?**
- Set Max Server Memory to 75-80% of total RAM
- Leave 20-25% for OS and other services
- For 16GB RAM, set to 12-13GB
- For 32GB VM, set to 24-25GB
- Min Server Memory should match expected baseline usage
- Dynamic memory allocation handles seasonal fluctuations
- Monitor memory usage and adjust accordingly

**Q40: How do you optimize startup parameters and trace flags?**
- Enable Trace Flag 1118 to eliminate allocation contention
- Trace Flag 1118 enabled by default in SQL Server 2016+
- Enable Trace Flag 3226 to suppress backup logging
- Set startup parameter -T to enable trace flags at startup
- Use Trace Flag 1117 for consistent file growth
- Document all trace flags for team knowledge
- Review trace flags quarterly for ongoing applicability

### Advanced Configuration

**Q41: What configuration changes improve SQL Server performance on Azure VMs?**
- Disable unnecessary services like SQL Browser if unused
- Disable Full-Text Search if not required
- Set Network Name as server alias for connection resilience
- Enable backup compression to reduce storage and time
- Enable instant file initialization to speed up file growth
- Configure max degree of parallelism based on core count
- Disable unused Reporting Services and Analysis Services

**Q42: How do you configure Max Degree of Parallelism?**
- Default auto-configuration works for most workloads
- Set to physical core count for OLTP workloads
- Reduce to 4-8 for mixed workloads to prevent resource exhaustion
- MAXDOP should not exceed available physical cores
- Monitor query plans to verify parallelism settings
- Test MAXDOP settings under production load
- Adjust dynamically based on performance monitoring

**Q43: Should you enable SQL Server compression?**
- Enable backup compression to reduce storage costs
- Backup compression reduces backup time significantly
- Enable page-level compression for large tables with repetitive data
- Enable row-level compression for slower growth but faster queries
- Compression trades CPU for storage and network efficiency
- Monitor CPU impact before enabling compression

**Q44: How do you configure SQL Server collation?**
- Choose collation during installation
- Case-insensitive collations are common for business applications
- Case-sensitive collations required for some applications
- Accent-sensitive collation affects special characters
- Changing collation after installation is complex
- Plan collation carefully before deployment
- Document collation choice for consistency across environments

**Q45: What monitoring and diagnostic settings should you enable?**
- Enable SQL Server error logging to capture issues
- Enable Windows Event Log for OS-level events
- Enable Azure Monitor for cloud-native monitoring
- Enable SQL Server Extended Events for performance diagnostics
- Configure alerts for critical errors
- Configure alerts for high resource usage
- Set up automated reporting on key metrics

---

## SECTION 4: NETWORKING & SECURITY (Q46-60)

### Virtual Networks and Connectivity

**Q46: How do you configure virtual networks for SQL Server VMs?**
- Create or use existing Azure Virtual Network
- Select appropriate subnet with sufficient IP addresses
- Assign static private IP to SQL Server VM for consistency
- Configure Azure DNS or your own DNS servers
- Set up routing for on-premises connectivity
- Use Site-to-Site VPN or ExpressRoute for hybrid connectivity
- Document network architecture for reference

**Q47: What is a Private Endpoint and why use one?**
- Private Endpoint creates private network connection to Azure services
- No internet exposure required with Private Endpoint
- SQL Server VM doesn't need public IP address
- Reduces attack surface by eliminating internet-facing connections
- Provides consistent IP addressing for firewall rules
- Enhances security posture significantly
- Limits network path exposure

**Q48: How do you implement Network Security Groups for SQL Server?**
- Create NSG rules allowing inbound SQL port 1433 only
- Restrict port 1433 to authorized subnets only
- Allow RDP port 3389 only from admin networks
- Deny all other inbound traffic by default
- Allow outbound HTTPS for Windows updates and telemetry
- Apply NSG to VM network interface or subnet
- Test rules to ensure correct filtering

**Q49: What is the difference between NSG and Azure Firewall?**
- Network Security Groups filter traffic at subnet level
- NSG provides simple allow/deny rules
- Azure Firewall provides centralized stateful firewall service
- Azure Firewall offers advanced threat protection
- Use NSG for basic filtering
- Use Azure Firewall for enterprise deployments with complex rules
- Can combine both for layered security

**Q50: How do you set up hybrid connectivity with on-premises networks?**
- Use Site-to-Site VPN for encrypted internet connection
- Use Azure ExpressRoute for dedicated private circuit
- ExpressRoute provides higher bandwidth and lower latency
- Use Virtual Network Peering to connect to other Azure VNets
- Configure routing to direct on-premises traffic correctly
- Test connectivity before production deployment
- Monitor hybrid connection health regularly

### Firewall and Encryption

**Q51: What firewall configurations are required for SQL Server?**
- Enable Windows Firewall with exception for SQL Server port 1433
- Allow SQL Browser port 1434 if using named instances
- Restrict ports to specific source IPs or subnets
- Allow RDP only from admin networks
- Block all unnecessary ports by default
- Test connectivity before moving to production
- Document firewall rules for compliance

**Q52: How do you enable SSL/TLS encryption for SQL connections?**
- Install certificate on SQL Server
- Configure SQL Server to use certificate
- Enable Force Encryption in SQL Server Configuration Manager
- Update client connection strings to require encryption
- Test encrypted connections before production deployment
- Monitor for certificate expiration
- Use encrypted connections for all remote access

**Q53: What certificate options exist for SQL Server encryption?**
- Use self-signed certificates for test environments
- Use certificates from internal PKI for enterprise deployments
- Use Azure Key Vault to store and manage certificates
- Certificates must match SQL Server FQDN
- Renew certificates before expiration
- Wildcard certificates simplify management
- Document certificate location and renewal schedule

**Q54: How do you implement Transparent Data Encryption (TDE)?**
- Enable TDE through SQL Server management tools
- Create Database Master Key
- Create certificate protected by master key
- Create Database Encryption Key using certificate
- Backup encryption keys to secure location immediately
- Monitor TDE performance impact
- Test restore procedures with encrypted databases

**Q55: What encryption is required for data at rest and in transit?**
- Disk encryption protects data at rest using Azure Disk Encryption
- Storage Service Encryption provides additional disk encryption
- TDE encrypts database files at SQL Server level
- SSL/TLS encrypts network traffic
- Back up encryption keys to prevent data loss
- Store backup encryption keys separately from databases
- Multiple layers of encryption provide defense in depth

### Authentication and Access Control

**Q56: What are the authentication options for SQL Server?**
- Windows Authentication uses AD credentials for access
- SQL Server Authentication uses database logins and passwords
- Windows Authentication is more secure and easier to manage
- Windows Authentication integrates with AD groups
- SQL Server Authentication required for non-domain-connected clients
- Most enterprises use Windows Authentication
- Prefer Windows Authentication for security

**Q57: How do you configure Azure AD integration with SQL Server?**
- Azure AD integration requires SQL Server 2019 or later
- Configure Azure AD authentication in SQL Server
- Create Azure AD logins in database
- Users authenticate using Azure AD credentials
- Requires Azure AD Connect for on-premises integration
- Simplifies credential management for cloud-native organizations
- Enables single sign-on capabilities

**Q58: How do you implement role-based access control?**
- Create SQL Server roles aligned with job functions
- Assign minimum required permissions to each role
- Use database roles for access within specific databases
- Use server roles for administrative functions
- Document role definitions and assignments clearly
- Audit role membership regularly for compliance
- Remove unused roles and permissions

**Q59: What are SQL Server login and user permissions best practices?**
- Use principle of least privilege for all accounts
- Disable sa account or rename it
- Create separate accounts for different applications
- Remove default users and roles if not needed
- Audit login attempts regularly for security
- Enforce strong password policies
- Restrict access to sensitive tables and procedures

**Q60: How do you secure SQL Server service accounts?**
- Use dedicated service accounts with minimum required permissions
- Use managed service accounts to eliminate password management
- Configure service account to run with no autologin
- Never use administrative accounts as service accounts
- Change service account passwords during maintenance
- Monitor service account usage for suspicious activity
- Document service account purpose and permissions

---

## SECTION 5: HIGH AVAILABILITY (Q61-75)

### Always On Availability Groups

**Q61: What is Always On Availability Group?**
- High-availability solution that replicates databases to secondary replicas
- Provides automatic failover to reduce downtime
- Supports read-only access on secondary replicas for reporting
- Can span multiple regions for disaster recovery
- Requires Enterprise or Standard editions
- Provides near-zero data loss with synchronous commit
- Achieves RTO under 60 seconds typically

**Q62: How do you set up Always On Availability Group?**
- Enable Always On feature in SQL Server Configuration Manager
- Create Windows Failover Cluster infrastructure
- Configure cluster network resources for heartbeat
- Create Availability Group and specify databases to replicate
- Add secondary replicas specifying synchronization mode
- Configure listener for client connections
- Test failover before production deployment

**Q63: What is the difference between synchronous and asynchronous commit?**
- Synchronous commit waits for secondary replica acknowledgment
- Synchronous provides highest data safety with minimal data loss
- Asynchronous commit returns immediately without waiting for secondary
- Asynchronous allows higher performance
- Asynchronous risks data loss if primary fails suddenly
- Synchronous adds latency to write operations
- Choose based on RTO and RPO requirements

**Q64: How do you configure readable secondaries?**
- Enable Read-Only Routing on secondary replicas
- Create read-only availability group endpoint
- Configure client routing rules to direct read queries
- Applications must explicitly connect to secondary
- Useful for offloading reporting workloads
- Reduces load on primary replica
- Improves query performance for reporting

**Q65: What is Availability Group listener and why is it needed?**
- Listener provides single connection point for client applications
- Automatically routes connections to primary replica
- Handles failover transparently without application changes
- Multiple listeners support read-write and read-only connections
- DNS-registered for easy client connection
- Clients connect to listener name, not instance name
- Simplifies connection management during failover

### Failover Clustering

**Q66: What is Failover Cluster Instance?**
- Provides high availability by sharing storage across multiple nodes
- All nodes access same database files on shared storage
- Automatic failover to healthy node on primary failure
- Appears as single instance to clients
- Requires shared storage like Storage Spaces Direct or SAN
- Faster failover than Always On in some scenarios
- Simpler to understand than Always On architecture

**Q67: How do you set up Windows Failover Clustering?**
- Install Windows Server on multiple VMs
- Configure shared storage accessible from all nodes
- Run Failover Cluster Manager to create cluster
- Validate cluster with built-in validation tests
- Create cluster resources for SQL Server
- Test failover scenarios thoroughly before production
- Document cluster configuration for reference

**Q68: What storage options support Failover Clustering?**
- Storage Spaces Direct uses local disks on each node
- Azure Shared Disk supports Premium SSD shared storage
- SAN storage accessed via iSCSI or Fibre Channel
- Storage Spaces Direct is cost-effective option
- Storage Spaces Direct doesn't require dedicated SAN
- Premium Shared Disk simplifies Azure deployments
- Shared storage is mandatory for FCI

**Q69: What is quorum configuration in clustering?**
- Quorum is majority vote among cluster nodes
- Prevents split-brain scenarios with multiple failures
- Node Majority requires more than half nodes healthy
- Disk Witness uses shared disk as tiebreaker
- File Share Witness uses network file share
- Configure for odd number of nodes when possible
- Quorum configuration prevents cluster split

**Q70: How does FCI differ from Always On Availability Groups?**
- FCI shares storage and provides single instance view
- Always On replicates data across separate storage
- FCI has faster failover but lower performance per node
- Always On provides read replicas and better scale-out
- Use FCI for legacy systems and SAP
- Use Always On for modern deployments
- FCI requires shared storage; Always On doesn't

### Load Balancing and Failover

**Q71: How do you implement load balancing for SQL Server?**
- Use Availability Group listener for read-write connections
- Direct read-only queries to readable secondaries
- Use Azure Load Balancer for distributing connections
- Implement connection pooling on clients
- Minimize connection overhead with pooling
- Monitor connection distribution to detect imbalances
- Balance CPU load across replicas

**Q72: What is connection routing and how does it work?**
- Availability Group listener routes write connections to primary
- Configure read-only routing to direct read queries to secondaries
- Connection routing is transparent to applications
- Clients connect to listener name instead of instance names
- Routing rules persist during failover
- Automatic routing eliminates connection reconfiguration
- Applications require no code changes for failover

**Q73: How do you measure RTO and RPO?**
- RTO (Recovery Time Objective) is maximum acceptable downtime
- Measure RTO from failure detection to service restoration
- RPO (Recovery Point Objective) is maximum acceptable data loss
- Measure RPO in time from last successful backup
- Always On typically achieves RTO under 60 seconds
- Always On achieves RPO near zero
- Document RTO and RPO in SLAs

**Q74: What are realistic RTO and RPO values for different HA solutions?**
- Always On synchronous achieves RTO 60-120 seconds and RPO near zero
- Always On asynchronous achieves RTO 60-120 seconds but RPO exceeds minutes
- FCI achieves similar RTO but depends on storage failure detection
- Manual failover adds minutes to RTO
- Automatic failover minimizes downtime
- Plan deployments to meet business requirements
- Test actual RTO and RPO values before production

**Q75: How do you test failover without impacting production?**
- Create test plan with defined failover procedures
- Test during maintenance windows with notification
- Monitor application behavior during failover
- Measure actual failover time and data consistency
- Document any issues for improvement
- Run failover tests quarterly minimum
- Update runbooks based on test findings

---

## SECTION 6: BACKUP & DISASTER RECOVERY (Q76-85)

### Backup Strategies

**Q76: What backup strategy should you implement?**
- Use full backup once weekly as baseline
- Perform differential backups daily or every 6-12 hours
- Perform transaction log backups every 5-15 minutes for VLDB
- Store multiple backup copies in geographically diverse locations
- Retain backups per business requirements, typically 30-90 days
- Document backup strategy for team reference
- Test restore procedures quarterly

**Q77: How do you automate SQL Server backups?**
- Use SQL Server Agent jobs to schedule backups
- Implement backup maintenance plans via Management Studio
- Use Azure Backup for managed backup service
- Write custom PowerShell scripts for advanced backup logic
- Test backup automation before relying on it
- Monitor backup jobs for failures daily
- Alert on backup job failures immediately

**Q78: What is Azure Backup and how does it work with SQL Server?**
- Provides managed backup service for SQL Server databases
- Handles automatic backups and retention policies
- Stores backups in Azure Recovery Services Vault
- Supports point-in-time restore and cross-region restore
- Simplifies backup management compared to manual approaches
- Provides automatic backup encryption
- Reduces manual backup operations significantly

**Q79: How do you configure backup retention policies?**
- Daily backups retained 7-30 days based on recovery needs
- Weekly full backups retained 4-12 weeks
- Monthly backups retained 12-36 months for archival
- Longer retention adds storage cost
- Balance retention requirements against cost and compliance needs
- Archive old backups to cold storage for cost savings
- Document retention policy for compliance

**Q80: What is the difference between backup to local disk versus Azure Storage?**
- Local disk backups are fast but limited by VM disk size
- Azure Storage backups scale to unlimited size
- Local backups require manual copy to offsite location
- Azure Storage backups automatically handle geo-redundancy
- Azure Storage is preferred for most deployments
- Local disk faster for initial backup but requires additional steps
- Azure Storage provides better disaster recovery capabilities

### Disaster Recovery

**Q81: How do you plan disaster recovery for SQL Server on Azure?**
- Document RTO and RPO requirements from business
- Design replication strategy using Always On or manual replication
- Plan failover procedures and communication
- Test failover and recovery procedures
- Document recovery steps for team execution
- Identify critical applications requiring fastest recovery
- Prioritize recovery based on business criticality

**Q82: What is geo-replication and when should you use it?**
- Maintains synchronized copy in different region
- Protects against region-wide outages
- Asynchronous replication prevents primary from blocking
- Use for mission-critical workloads
- Cross-region traffic adds latency and cost
- Consider for business-critical applications only
- Implement with proper monitoring and alerting

**Q83: How do you perform disaster recovery failover?**
- Activate secondary region database and verify consistency
- Update DNS or connection strings to point to secondary
- Monitor secondary for stability before declaring success
- After primary is recovered, plan failback carefully
- Failback requires careful synchronization to avoid data loss
- Verify application functionality on secondary
- Communicate failover status to stakeholders

**Q84: What testing should you perform for disaster recovery?**
- Test backup restore procedure quarterly at minimum
- Test failover to secondary region annually
- Verify restore time meets RTO requirements
- Verify data consistency after failover
- Document actual recovery time and update procedures
- Run tests with production-like data volumes
- Test during off-hours to minimize impact

**Q85: How do you handle disaster recovery documentation?**
- Document detailed recovery procedures step-by-step
- Maintain runbooks for failover and failback scenarios
- Document contact lists and escalation procedures
- Store documentation in multiple locations for accessibility
- Update documentation after each recovery test
- Update documentation after production incidents
- Share documentation with backup team members

---

## SECTION 7: PERFORMANCE TUNING (Q86-95)

### Resource Optimization

**Q86: How do you optimize CPU performance?**
- Monitor CPU usage with Performance Monitor or Extended Events
- Identify queries consuming excessive CPU
- Optimize queries with better indexes or execution plans
- Reduce parallelism if causing context switching
- Scale to VM with more cores if optimization isn't sufficient
- Use multi-core VMs for compute-intensive workloads
- Monitor context switching for CPU efficiency

**Q87: How do you optimize memory performance?**
- Set Max Server Memory correctly to prevent OS memory starvation
- Monitor buffer pool for efficient cache hits
- Identify missing indexes causing full scans
- Increase memory for OLAP workloads if appropriate
- Add more RAM if memory is consistently over 80% utilized
- Monitor memory usage patterns
- Adjust memory allocation based on workload type

**Q88: How do you optimize disk performance?**
- Separate data, log, and tempdb to different disks
- Use Premium SSD for production workloads
- Enable write caching for data disks only
- Monitor disk queue length and IOPS usage
- Add additional disks or upgrade disk tier if IOPS limits reached
- Monitor disk latency for performance issues
- Balance cost and performance when selecting disk type

**Q89: What monitoring tools help identify performance issues?**
- Performance Monitor captures OS-level CPU, memory, disk metrics
- SQL Server Management Studio shows query execution plans
- Extended Events provide detailed diagnostic tracing
- Query Store tracks query performance over time
- Azure Monitor visualizes resource usage trends
- Dynamic Management Views show real-time performance
- Combine multiple tools for comprehensive monitoring

**Q90: How do you identify performance bottlenecks?**
- Establish baseline performance metrics before issues occur
- Monitor key metrics like CPU, memory, disk IOPS
- Identify top resource-consuming queries
- Check for blocking and deadlocks
- Review wait statistics to pinpoint contention
- Compare current performance against baseline
- Document performance issues for root cause analysis

### Query and Index Optimization

**Q91: How do you improve query performance?**
- Analyze actual execution plans to find inefficiencies
- Add indexes on frequently filtered columns
- Remove unused indexes reducing write overhead
- Rewrite queries to eliminate subqueries where possible
- Update statistics for better query optimization
- Monitor query execution time trends
- Test query changes in lower environments first

**Q92: What is index fragmentation and how do you address it?**
- Fragmentation occurs when index pages are scattered across disk
- Reorganize indexes with less than 30% fragmentation
- Rebuild indexes with greater than 30% fragmentation
- Schedule maintenance during off-peak hours
- Monitor fragmentation trends over time
- Automatic index maintenance reduces manual work
- Defragmentation improves query performance significantly

**Q93: How do you use SQL Profiler for performance diagnostics?**
- Capture SQL statements and execution details with Profiler
- Filter traces to focus on problem queries
- Identify long-running queries and deadlocks
- Export trace data for analysis
- Use Extended Events instead of Profiler for production
- Extended Events have lower overhead than Profiler
- Profiler useful for development and testing

**Q94: What statistics maintenance is required?**
- Update statistics on tables after significant changes
- Auto-update statistics should be enabled
- Manually update statistics on heavily modified tables
- Outdated statistics cause poor query optimization
- Monitor statistics freshness regularly
- Statistics updates improve query plan quality
- Schedule statistics updates during maintenance windows

**Q95: How do you implement query plan analysis?**
- Capture actual execution plans for slow queries
- Compare estimated and actual row counts
- Identify estimation issues for optimization
- Identify expensive operators like nested loops or sorts
- Check if plan uses indexes efficiently
- Test query plan changes in test environment first
- Monitor plan performance changes in production

---

## SECTION 8: REAL-TIME SCENARIOS (Q96-100)

### Common Issues and Troubleshooting

**Q96: What do you do if SQL Server becomes unresponsive?**
- Check CPU, memory, and disk utilization first
- Review error logs for critical errors
- Identify blocking queries using DMVs
- Kill blocking sessions if safe to do so
- Restart SQL Server if necessary
- Investigate root cause to prevent recurrence
- Monitor resource utilization after restart

**Q97: How do you handle failed backups?**
- Check backup job logs for failure reason
- Verify backup storage has sufficient space
- Confirm network connectivity to backup destination
- Re-run backup manually to test
- Update backup path if storage moved
- Implement backup verification job to catch failures early
- Alert on backup failures within 1 hour of failure

**Q98: What do you do if failover cluster loses quorum?**
- Identify which nodes are offline
- Restart offline nodes if infrastructure issue
- Force quorum if majority nodes are unavailable temporarily
- Investigate why nodes became unavailable
- Prevent single points of failure in future deployments
- Document quorum loss for post-incident analysis
- Update monitoring to detect quorum loss earlier

### Post-Incident Analysis

**Q99: How do you conduct post-incident analysis?**
- Document incident timeline from detection to resolution
- Identify root cause from logs and diagnostic data
- Review what worked well and what didn't
- Determine preventive measures for future
- Schedule follow-up to verify preventive measures
- Include team members in post-incident review
- Document analysis findings for future reference

**Q100: How do you implement lessons learned from incidents?**
- Update runbooks with insights from incident response
- Improve monitoring to detect similar issues earlier
- Automate manual troubleshooting steps
- Share learning across team through documentation
- Schedule refresher training on updated procedures
- Track lessons learned to prevent repeated issues
- Test updated procedures before production use

---

## Quick Reference Checklist

### Pre-Deployment
- [ ] Assess workload requirements and document metrics
- [ ] Size VM and storage appropriately
- [ ] Choose licensing model and confirm costs
- [ ] Design network and security architecture
- [ ] Plan backup and disaster recovery strategy

### Deployment
- [ ] Deploy SQL Server VM from appropriate image
- [ ] Configure networking and security groups
- [ ] Apply initial patches and updates
- [ ] Configure storage and file placement
- [ ] Set up monitoring and diagnostics

### Post-Deployment
- [ ] Configure Always On or FCI if high availability needed
- [ ] Implement backup automation
- [ ] Test failover scenarios
- [ ] Establish performance baselines
- [ ] Document procedures and runbooks

### Ongoing
- [ ] Monitor key performance metrics daily
- [ ] Apply security patches promptly
- [ ] Review and optimize slow queries
- [ ] Maintain indexes and statistics
- [ ] Test disaster recovery procedures quarterly

---

## Additional Resources

- Microsoft Learn SQL Server on Azure VM courses
- Azure Architecture Center guidance and patterns
- SQL Server official documentation and release notes
- Azure cost calculator for budget planning
- Community forums for peer support and best practices
