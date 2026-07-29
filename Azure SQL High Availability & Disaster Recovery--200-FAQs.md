Azure SQL High Availability & Disaster Recovery: 200 Questions and Answers

Beginner to Advanced with Real-Time Scenarios

BEGINNER LEVEL (Questions 1-40)

Q1: What is the core difference between High Availability and Disaster Recovery?

* High Availability (HA)
  * Survives small, localized failures within same region
  * Node crashes, storage issues, datacenter problems
  * Minimal or no downtime for end users
  * Focus on keeping service running during component failures
* Disaster Recovery (DR)
  * Survives catastrophic events across entire regions
  * Regional outages, natural disasters, infrastructure collapse
  * Involves failover to different geography
  * Accepts longer recovery time and planned data loss tolerance
  * Separate from HA, requires distinct strategy

Q2: Does Azure SQL Database include high availability by default?

* Yes, built-in at all service tiers
* No configuration required by user
* Architecture varies by tier but HA is always present
* Automatic failover happens without manual intervention
* User pays for this through service tier cost
* Different tiers offer different failover speeds

Q3: What are the two main HA architectures in Azure SQL Database?

* General Purpose tier
  * Separates compute and storage layers
  * Single compute node with remote, redundant Azure storage
  * Failover involves provisioning new compute and reattaching to storage
  * More cost-effective option
  * Slower failover time than Business Critical
  * Suitable for non-critical workloads
* Business Critical tier
  * Multiple synchronized replicas with local storage
  * Similar to SQL Server Always On Availability Groups
  * Each replica has own dedicated storage
  * Faster failover under 30-60 seconds
  * Higher cost due to redundant infrastructure
  * Suitable for mission-critical applications

Q4: What is Recovery Time Objective (RTO)?

* Time business can tolerate being without database access
* Measured in minutes or seconds
* Drives all architecture decisions for HA/DR strategy
* Five-minute RTO requires different design than four-hour RTO
* Cost increases significantly as RTO requirement tightens
* Must be established before designing any solution
* Not a technical metric but business requirement

Q5: What is Recovery Point Objective (RPO)?

* Maximum acceptable data loss measured in time
* Example: RPO of 5 minutes means losing up to 5 minutes of transactions is acceptable
* Different from RTO which measures recovery time
* Determines replication strategy choice
* Affects tier selection and architecture
* Both RTO and RPO needed for complete requirements picture

Q6: What is a failover group in Azure SQL?

* Logical grouping of one or more databases
* Includes primary and secondary regions
* Provides stable, unchanging endpoint for applications
* Connection string remains same during failover
* Automatic DNS redirection to current primary
* Simplifies application configuration during regional outages
* Reduces manual intervention during incidents

Q7: What is active geo-replication?

* Continuous near-synchronization of database to different region
* Creates readable secondary copy automatically
* Underlying mechanism behind failover groups
* Can be used directly without failover group wrapper
* Enables manual failover control when needed
* Supports multiple secondary replicas from single primary
* Useful for distributing read-only workloads

Q8: What is a hot standby versus cold standby?

* Hot standby
  * Already running and synchronized
  * Ready for immediate takeover
  * Example: geo-replica in standby mode
  * Higher ongoing cost due to running infrastructure
  * Achieves lower RTO values
* Cold standby
  * Does not exist until needed
  * Provisioned and restored from backup during disaster
  * Lower cost during normal operations
  * Slower recovery time
  * Acceptable for less critical systems

Q9: Does Azure SQL's built-in HA equal DR capability?

* No, these are separate concerns
* Built-in HA protects only within single region
* Only handles node or storage failures
* Does not protect against regional outage
* Entire region failure leaves built-in HA useless
* Must add separate DR mechanism for regional resilience
* Common beginner misconception to treat as equivalent

Q10: What is read-only replica and how does it support HA?

* Secondary replica serving read-only query traffic
* Available in Business Critical and Premium tiers
* Accessed via read scale-out feature
* Same infrastructure protecting availability also handles reads
* Reduces load on primary during normal operations
* Improves query performance for reporting workloads
* Replicas used for both HA and workload distribution

Q11: What happens to database connections during failover?

* Active connections are dropped immediately
* Applications must reconnect to continue working
* Failover is not invisible at connection layer
* HA failover typically completes in seconds
* DR failover takes longer depending on region distance
* Applications need retry logic to handle gracefully
* Connection pooling helps recover faster

Q12: What is zone-redundant configuration?

* Spreads replicas across availability zones in same region
* Availability zones are physically separate datacenters nearby
* Protects against single datacenter failure
* Lower latency than cross-region redundancy
* Lower cost than full geo-redundancy
* Complements geo-redundancy for defense in depth
* Different from geo-redundancy which spans regions

Q13: What is geo-redundancy?

* Replicates data to completely different region
* Protects against regional-level outages
* Significantly higher latency than zone redundancy
* Higher cost due to cross-region data transfer
* Necessary for genuine disaster recovery
* Complements zone redundancy within single region
* Only solution for surviving full region failures

Q14: Can you choose secondary region for failover group?

* Yes, within Azure-supported regional pairs
* Not limited to automatically paired region
* Can select any compatible region
* Paired regions often recommended for compliance
* Distance affects latency and replication lag
* Regulatory requirements may dictate region choice
* Cost varies by region selection

Q15: Why does Business Critical tier cost more than General Purpose?

* Maintains multiple continuously running replicas
* Each replica has own local storage
* All replicas kept fully synchronized at all times
* Dedicated resources not shared with other customers
* Faster failover readiness requires this overhead
* General Purpose shares compute and storage resources
* Price difference reflects infrastructure cost difference

Q16: What is automatic failover policy in failover groups?

* Platform detects outage and switches roles automatically
* No human decision or intervention required
* Minimizes downtime by eliminating human delay
* Trusts platform detection logic for accuracy
* Risk of triggering unnecessary failover
* Better for systems tolerating brief false positives
* Alternative is manual failover requiring human approval

Q17: Why test failover before real emergency?

* Unknown actual failover duration until tested
* Application behavior during reconnect unvalidated
* Documented plan may not work as expected
* Real infrastructure differs from documentation
* Network routing during failover needs verification
* Testing surfaces unexpected dependencies
* Untested DR plan is essentially unproven

Q18: What question should you ask first when designing HA/DR?

* What are the actual RTO and RPO requirements?
* This single answer drives all subsequent decisions
* Which tier to select depends on RTO
* Whether geo-replication needed depends on DR scope
* How many regions required depends on requirements
* Automatic versus manual failover depends on RTO tolerance
* Skipping this step leads to over or under engineering

Q19: What is the difference between sync and async replication?

* Synchronous replication
  * Secondary must acknowledge before primary commits
  * Ensures zero data loss during failover
  * Higher latency for write operations
  * Used within regions for HA
* Asynchronous replication
  * Primary commits without waiting for secondary
  * Better performance but possible data loss
  * Used for cross-region geo-replication
  * Secondary eventually catches up
  * Faster writes but RPO risk increases

Q20: How does Azure ensure data durability in remote storage?

* Multiple redundant copies across datacenters
* Built-in replication within storage layer
* Checksums verify data integrity
* Automatic repair if corruption detected
* Three copies minimum within region
* Geo-redundant storage option available
* Storage redundancy independent of database tier

Q21: What is the role of Azure Service Health in DR planning?

* Alerts about known Azure platform issues
* Helps distinguish regional outage from local problem
* Shows status across regions and services
* Early warning for developing issues
* Supports decision making during ambiguous incidents
* Different from application-level monitoring
* Should trigger DR response procedures

Q22: What is the relationship between availability zones and regions?

* Region contains multiple availability zones
* Each zone has separate power, cooling, networking
* Zones are geographically close but isolated
* Database can spread across zones in region
* Regions are geographically far apart
* Zone redundancy cheaper than regional redundancy
* Both needed for comprehensive resilience

Q23: How does connection pooling help with failover recovery?

* Pooled connections allow rapid reconnection
* Failed connections removed from pool automatically
* New connections get fresh database connection
* Application sees quicker recovery
* Without pooling every connection must retry
* Reduces time users experience unavailability
* Should be configured with appropriate timeouts

Q24: What is the purpose of Application Intent connection string parameter?

* Tells Azure which replica to route connection to
* ApplicationIntent=ReadOnly routes to secondary
* ApplicationIntent=ReadWrite routes to primary
* Enables intelligent read distribution
* Reduces load on primary for reporting queries
* Read-only queries fail if sent with ReadWrite intent
* Application responsible for correct parameter usage

Q25: What backups are available for Azure SQL Database?

* Automatic full backups
  * Taken weekly or at service tier default
  * Retained based on service tier
  * Geo-redundant backups available
* Automatic transaction log backups
  * Captured continuously
  * Enable point-in-time restore
  * Five to seven days retention standard
* Automatic differential backups
  * Capture changes since full backup
  * Reduce backup size and time
* User-initiated backups
  * Long-term retention via Azure Backup
  * Manual backups for compliance

Q26: What is the difference between backup and geo-replication?

* Backup
  * Point-in-time snapshots taken periodically
  * Survives data corruption or user mistakes
  * Longer recovery process
  * Lower ongoing cost
* Geo-replication
  * Continuous near-real-time synchronization
  * Survives regional outages
  * Faster recovery
  * Higher ongoing cost
* Both needed for comprehensive protection

Q27: How does Private Endpoint affect HA/DR strategy?

* Restricts database access to private network
* Must exist in both primary and secondary regions
* DNS resolution must work across regions
* Network connectivity must span regions
* Failover group DNS changes may affect routing
* Requires careful network planning for DR
* Can become single point of failure if not redundant

Q28: What is the cost of unused failover group secondary?

* Secondary region incurs compute costs
* Storage costs for replicated data
* Network bandwidth for geo-replication
* No discount for standby role
* Costs accrue even if never used
* Cost-benefit analysis critical for decision
* Some organizations accept cost for compliance

Q29: What is meant by "near-synchronous" replication?

* Not truly synchronous
* Small acceptable lag between primary and secondary
* Typically milliseconds to seconds
* Faster than async but not zero-lag
* Some transactions may not reach secondary immediately
* Acceptable compromise for geo-replication
* Actual lag varies with network conditions

Q30: How does database size affect failover time?

* Larger databases generally fail over faster
* Counterintuitive because more data to sync
* Secondary already fully populated
* No need to copy data during failover
* Failover is metadata operation not data copy
* Network distance affects replication lag more
* Size matters less for failover speed

Q31: What is the purpose of planned failover testing?

* Validates failover mechanism actually works
* Tests application reconnection behavior
* Measures actual failover duration
* Discovers unexpected dependencies
* Validates monitoring and alerting
* Confirms runbook accuracy
* Should be done regularly, not just at launch

Q32: What is the difference between failover and failback?

* Failover
  * Moves primary role to secondary during incident
  * One-way operation during crisis
  * Temporary solution until primary recovers
* Failback
  * Moves primary role back to original region
  * Often treated as afterthought but equally risky
  * Involves same connection disruptions
  * Should be tested with same rigor as failover
  * Proper planning needed for both directions

Q33: Can you have multiple secondaries for one primary database?

* Yes, active geo-replication supports multiple secondaries
* Each secondary maintains independent copy
* Primary sends changes to all secondaries
* Useful for distributing read-only workloads
* Each secondary can serve different purpose
* Failover group typically uses one active secondary
* Additional secondaries increase complexity and cost

Q34: What happens to in-flight transactions during failover?

* Uncommitted transactions are rolled back
* Committed transactions persist
* Business Critical tier loses no committed transactions
* General Purpose may lose recent transactions
* Async geo-replication loses more transactions
* Sync replication within region loses none
* Applications should handle rolled-back transactions

Q35: How does database collation affect replication?

* Secondary must use identical collation
* Mismatch causes replication failures
* Collation set at database creation time
* Cannot change collation after creation
* Must match primary exactly
* Affects string comparison and sorting
* Plan collation before creating replica

Q36: What is the role of transaction log in HA/DR?

* Records all database changes chronologically
* Enables point-in-time restore capability
* Transmitted to secondaries for replication
* Allows recovery of specific transactions
* Cleared after backup completion
* Size affects database performance
* Infinite retention not possible due to cost

Q37: What is database snapshot and when is it useful?

* Read-only copy of database at specific point in time
* Created instantly from snapshot isolation
* Useful for reporting without impacting primary
* Different from geo-replication replica
* Exists only in same region
* Minimal storage overhead using copy-on-write
* Good for ad-hoc analysis without DR benefits

Q38: How does Azure handle maintenance windows for HA databases?

* Planned maintenance uses failover to secondary
* Brief connection interruption during failover
* Users experience transparent maintenance
* Secondary still available for reads during primary maintenance
* No downtime announced to users typically
* Failover process same as during real incident
* Business Critical tier handles maintenance better

Q39: What is the purpose of read scale-out feature?

* Routes read-only connections to secondary replicas
* Offloads reporting queries from primary
* Reduces CPU and memory pressure on primary
* Application must specify ApplicationIntent=ReadOnly
* Secondary handles queries while primary handles OLTP
* Improves overall system throughput
* Only available in Business Critical tier

Q40: What metrics should be monitored for failover group health?

* Replication lag indicates secondary sync status
* CPU and memory on both primary and secondary
* Network bandwidth for geo-replication traffic
* Connection counts on each replica
* Query performance trending
* Backup completion and retention
* Service Health status for regions


INTERMEDIATE LEVEL (Questions 41-100)

Q41: Explain the architectural differences between General Purpose and Business Critical in detail.

* General Purpose
  * Remote Azure Premium storage layer
  * Single compute node per database
  * Compute and storage separated
  * Failover requires compute provisioning
  * Storage already has redundancy built-in
  * Multi-step failover process
  * More affordable tier
* Business Critical
  * Four total replicas per database
  * Primary plus three secondaries
  * Each replica has local storage
  * Sync replication between all
  * Failover is simple promotion
  * Optimized for low-latency operations
  * Premium pricing

Q42: What causes extended replication lag in geo-replication?

* Heavy write workload exceeding secondary capacity
* Network latency or bandwidth constraints
* Blocked transactions on secondary
* Long-running queries holding locks
* High CPU on secondary preventing catches-up
* Undersized secondary tier
* Geographic distance affecting latency

Q43: How do you determine if replication lag is acceptable?

* Compare lag to RPO requirement
* Lag should stay well below RPO threshold
* Monitor lag under peak load conditions
* Sustained lag indicates secondary undersized
* Spikes acceptable if brief and infrequent
* Test failover to validate lag assumption
* Adjust secondary tier if lag unacceptable

Q44: What is a read-write split strategy in distributed applications?

* Routing reads to secondary replicas
* Routing writes to primary only
* Reduces load on primary significantly
* ApplicationIntent parameter controls routing
* Application must handle read-after-write consistency
* Cross-region read routing increases latency
* Useful for read-heavy applications

Q45: How does failover group differ from plain geo-replication?

* Failover group
  * Provides stable listener endpoint
  * Automatic DNS redirection
  * Application needs no config change
  * Built-in failover automation available
  * Simpler operational model
* Plain geo-replication
  * Direct connection to secondary
  * Manual failover required
  * Connection string must change
  * More control for advanced scenarios
  * Lower overhead if only needing replicas
  * Better for multi-secondary setups

Q46: What is the purpose of failover group listener endpoint?

* Single, unchanging DNS name for applications
* Points to current primary database
* Automatically redirects during failover
* Survives multiple failover cycles
* No application code changes needed
* Applications use same connection string always
* Simplifies disaster recovery procedures

Q47: How do you manage connection strings with failover groups?

* Use failover group listener endpoint
* Not individual server names
* Single connection string for all scenarios
* DNS resolution handles routing
* Connection pooling maintains connections
* Retry logic handles brief interruptions
* No code changes during failover events

Q48: What is a planned failover in failover groups?

* Initiated deliberately for testing or maintenance
* Waits for secondary to catch up completely
* No data loss occurs
* Clean, synchronized role switch
* Safe to perform during business hours with planning
* Can failback easily afterward
* Best way to test DR procedures

Q49: What is a forced failover in failover groups?

* Initiated when primary is completely unavailable
* Does not wait for secondary synchronization
* May lose recent transactions
* Used only in genuine emergencies
* Faster than planned failover
* Riskier due to potential data loss
* Cannot easily revert if primary recovers

Q50: When would you use forced failover versus planned failover?

* Forced failover
  * Primary region completely down
  * Cannot afford to wait for sync
  * Willing to accept data loss
  * Real disaster scenario
* Planned failover
  * Primary still responding
  * Testing DR procedures
  * Performing maintenance
  * Demonstrating functionality
  * Switching regions deliberately
  * Practicing incident response

Q51: How do you validate successful database failover?

* Connect to database and run queries
* Check database role and status
* Verify replication status
* Test application connectivity
* Confirm user access restored
* Monitor performance metrics
* Review transaction logs if concerned about data loss

Q52: What is the relationship between Business Critical and Always On Availability Groups?

* Business Critical mimics Always On architecture
* Not actual Always On, but similar principles
* Multiple synchronized replicas with local storage
* Quorum-based commit required for transactions
* Automatic failover when primary fails
* Read scale-out like Always On
* Azure-managed so users don't manage Always On directly

Q53: How does Hyperscale tier differ in HA/DR approach?

* Different storage architecture than General Purpose
* Page servers handle distributed storage
* HA replicas provide failover protection
* Named replicas for workload-isolated reads
* Very large database support
* Different failover and backup characteristics
* Still supports geo-replication but different behavior

Q54: What is a named replica in Hyperscale?

* Read-only copy not part of failover topology
* Separate from HA replicas
* Can be in different region independently
* Good for analytics or reporting workloads
* Different SLA than HA replicas
* Not automatically failed over
* User must manage failover manually

Q55: How do you architect multi-database failover groups?

* Multiple databases in same failover group
* All databases failover together
* Maintains transactional consistency
* Single failover event triggers all
* Simplifies operational management
* Ensures no data inconsistency during failover
* Preferred over managing individual database failovers

Q56: What is the challenge of sharded database architecture in DR?

* Multiple shards each need failover protection
* Coordinating failover across all shards critical
* Partial failover leaves application inconsistent
* Shard map must reflect new topology
* Routing layer must handle failover
* Testing becomes significantly more complex
* Requires automation to ensure atomic failover

Q57: How do you handle shard map updates during failover?

* Shard map contains routing information
* Must be updated to reflect new primary locations
* Typically kept in separate management database
* Can be replicated to secondary region
* Update must be coordinated with failover
* Routing uses map to direct connections
* Incorrect shard map breaks application routing

Q58: What is geo-restore capability?

* Restore database from backup to any region
* Uses geo-redundant backup storage
* Different from geo-replication
* Slower recovery than geo-replication
* Useful for recovering from data corruption
* Last resort when replication unavailable
* Only recovers to point backup was taken

Q59: When would you use geo-restore instead of failover?

* Data corruption in primary database
* Accidental deletion of important data
* Replication somehow corrupted too
* Need to recover to specific point in time
* Failover group not protecting this scenario
* Building second copy for analytics
* Testing backup and restore processes

Q60: How do you recover from data corruption across all replicas?

* Corruption replicates to all replicas
* Failover does not help
* Must restore from prior backup
* Geo-restore recovers clean copy
* Point-in-time restore to pre-corruption time
* Long-term retention useful for this scenario
* Early detection critical to minimize impact

Q61: What is the purpose of Resource Health in incident response?

* Shows health status of specific resources
* More detailed than general Service Health
* Identifies specific database impact
* Helps decide if failover is necessary
* Different resource states indicate issues
* Transitions show when issue started and resolved
* Better than guessing based on region status

Q62: How does Service Health alert you to issues?

* Proactive notifications about known issues
* Affects your resources specifically
* Different from general status page
* Can set up alerts for subscriptions
* Helps distinguish your issue from others
* Enables informed decision making
* Supports planning around maintenance

Q63: What is the role of Application Insights in failover scenarios?

* Monitors application performance
* Detects when database unavailable to app
* Shows user-perceived impact timing
* Correlates with database events
* Helps validate DR testing success
* Alerts on degraded performance
* Different from database-level monitoring

Q64: How do you measure actual Recovery Time Objective achieved?

* Track timeline from outage start to user recovery
* Not just database failover completion
* Application restart time included
* Network reconnection delays matter
* User sessions may take time to reestablish
* Use application monitoring for timing
* Measure from user perspective, not database

Q65: How do you measure actual Recovery Point Objective achieved?

* Compare last transaction on new primary
* Against last known transaction on old primary
* Query Store or transaction logs provide evidence
* Not assumed, but measured from events
* May reveal data loss not expected
* Important for post-incident analysis
* Helps validate architecture assumptions

Q66: What is the purpose of post-incident reviews for DR?

* Identify gaps in procedure or testing
* Update runbook based on what happened
* Fix monitoring that didn't catch issue
* Improve alerting for next time
* Document lessons learned
* Validate assumptions about failover
* Continuous improvement cycle

Q67: How does encryption affect failover operations?

* Encrypted data replicates as ciphertext
* Encryption keys must be accessible at failover site
* Column encryption keys needed for decryption
* Always Encrypted adds complexity to DR
* Keys in Azure Key Vault should be geo-redundant
* Test key access in secondary region
* Encryption key unavailability breaks recovery

Q68: What is Transparent Data Encryption and its HA/DR impact?

* Encrypts data at rest automatically
* Encryption key in Azure Key Vault
* Key must be accessible during failover
* Key Vault should be geo-redundant
* Simpler than Always Encrypted
* No application-level encryption management
* Replication and failover not impacted

Q69: How does Always Encrypted complicate DR procedures?

* Column master keys required for decryption
* Keys typically in Azure Key Vault
* Application must retrieve keys
* Secondary region must access same Key Vault
* Key Vault access might be restricted
* Testing must validate key access from DR region
* Configuration complexity increases significantly

Q70: What is the purpose of failover group grace period setting?

* Delays automatic failover after outage detected
* Prevents unnecessary failover for brief blips
* Brief transient outages resolve without failover
* Failing over unnecessarily has cost
* New primary must warm up under load
* Grace period balances false positives and speed
* Tuned based on environment characteristics

Q71: How do you set appropriate grace period value?

* Analyze historical outage patterns
* How long do transient outages typically last
* How long to distinguish real from false outage
* Too short risks unnecessary failovers
* Too long eats into achievable RTO
* Data-driven approach better than guessing
* Adjust based on incident experience

Q72: What is the difference between automatic and manual failover policy?

* Automatic
  * Platform initiates failover after grace period
  * Minimizes human decision time
  * Risk of wrong failover decision
  * Good for sub-minute RTO requirements
* Manual
  * Human must explicitly approve failover
  * Adds delay but provides control
  * Better for partial outage ambiguity
  * Slower but more deliberate
  * Requires human availability

Q73: When would you choose manual failover policy over automatic?

* Regulatory requirement for human approval
* History of false positive outage detection
* Prefer control over speed
* Can tolerate longer RTO
* Partial outages common in environment
* Want explicit sign-off from leadership
* Organization has 24/7 on-call team

Q74: What are the costs of unnecessary failover?

* New primary takes time to warm up
* Failback when original recovers costly
* Customers might lose confidence
* Connection drops impact user experience
* Unnecessary risk of data loss
* Operational overhead and stress
* Why grace period and testing important

Q75: How does load testing validate DR readiness?

* Tests secondary region capacity
* Simulates production workload
* Validates performance assumptions
* Identifies bottlenecks in secondary
* Confirms application works under load
* Exercises failover with realistic traffic
* Reveals scaling issues before real incident

Q76: What is chaos engineering for databases?

* Deliberately introduce failures
* Test system resilience under adversity
* Simulate forced failover conditions
* Network latency and packet loss
* Cascading failures of dependencies
* Validates runbooks against reality
* More realistic than clean planned tests

Q77: How do you test failover without impacting production?

* Use planned failover for clean test
* Scheduled during low-traffic window
* Clear communication to stakeholders
* Monitor closely for unexpected issues
* Document everything that happens
* Failback afterward to restore primary role
* Repeat periodically to stay ready

Q78: What is blue-green deployment pattern for databases?

* Run two identical database environments
* One active (blue), one standby (green)
* Switch traffic between them
* Zero-downtime updates possible
* Good for testing schema changes
* Similar concept to failover groups
* Requires application aware of both

Q79: How does database size affect recovery speed?

* Larger databases take longer to restore
* Failover speed not affected by size
* Backup and restore time increases linearly
* Replication bandwidth for large databases
* Archive storage for long-term retention
* Compression helps but not always applicable
* Plan for largest expected size

Q80: What is the impact of database growth on DR strategy?

* Secondary region sizing must match growth
* Costs increase proportionally
* Replication lag may increase
* Bandwidth requirements grow
* Backup storage costs increase
* Must monitor and adjust periodically
* Growth planning part of DR review

Q81: How do you protect against multiple simultaneous failures?

* Assume worst case always possible
* Design for loss of multiple components
* Multiple regions for geographic diversity
* Multiple availability zones within region
* Separate Key Vault redundancy
* Application tier redundancy
* Layered defense with no single point of failure

Q82: What is the concept of defense in depth for databases?

* Multiple protection layers instead of one
* Zone redundancy and geo-redundancy combined
* Backup and geo-replication both enabled
* Application retry logic and health checks
* Monitoring and alerting at multiple levels
* Reduces risk from any single failure
* Each layer compensates for others' gaps

Q83: How does Private Endpoint affect DR network design?

* Restricts connectivity to private network
* Must exist in both primary and secondary regions
* DNS must resolve correctly in both regions
* VPN or ExpressRoute must span regions
* Network connectivity becomes critical dependency
* Failover must account for network layer
* Test network access during DR validation

Q84: What is ExpressRoute redundancy for DR?

* Direct network connection between Azure regions
* Redundancy critical for DR readiness
* Single ExpressRoute circuit single point of failure
* Dual circuits recommended
* Different providers preferred
* Bandwidth must accommodate failover traffic
* More expensive but essential for availability

Q85: How does VPN backup differ from ExpressRoute for DR?

* VPN less expensive than ExpressRoute
  * Can serve as backup to ExpressRoute
  * Slower but better than nothing
  * Automatic failover possible
  * Setup requires planning
  * Test failover before relying on it
  * Good for smaller applications

Q86: What is the purpose of Application Gateway in DR scenarios?

* Load balances traffic across regions
* Geo-routing possible
* Health probes check backend health
* Automatic failover to healthy region
* Different from database failover
* Works with failover groups
* Improves application tier resilience

Q87: What is the purpose of Traffic Manager in multi-region scenarios?

* DNS-based traffic routing
* Routes users to nearest region
* Health checks for endpoint availability
* Automatic failover at DNS level
* Different from Application Gateway
* Works with failover groups
* Global load balancing capability

Q88: How do you handle DNS propagation delays during failover?

* DNS changes take time to propagate globally
* TTL controls time before change is observed
* Lower TTL faster propagation but higher load
* Clients cache DNS for TTL duration
* Some clients ignore TTL
* Plan for DNS delay in recovery timeline
* Test actual DNS propagation in your regions

Q89: What is connection retry strategy for applications?

* Exponential backoff avoids overwhelming system
* Jitter prevents thundering herd problem
* Short timeout prevents long waits
* Limited retry attempts prevent spinning
* Different timeout for different operations
* Test retry behavior during failover
* Critical for graceful failover handling

Q90: How does connection pooling affect failover recovery?

* Pooled connections fail and get recreated
* Faster than creating new connections
* Pool automatically detects bad connections
* Fresh connection goes to current primary
* Reduces time before app works again
* Connection pool size affects concurrency
* Timeout settings critical for failover

Q91: What monitoring should be in place for failover readiness?

* Replication lag trending
* Secondary region capacity utilization
* Network bandwidth saturation
* Backup completion and size
* Encryption key access from secondary
* Application connectivity testing
* Cost monitoring for secondary resources

Q92: What dashboards help visualize failover group health?

* Primary and secondary region metrics side by side
* Replication lag trending over time
* Database size growth projection
* Cost allocation by region
* Failover test success history
* Recovery time metrics
* Summarize key HA/DR indicators

Q93: What is the purpose of capacity planning for DR regions?

* Secondary must handle full production load
* Underprovisioned secondary defeats purpose
* Test with realistic production load
* Monitor performance during test failover
* Plan for growth over time
* Cost increases with capacity
* Review and adjust regularly

Q94: Should secondary region be same tier as primary?

* Generally yes, should match primary
* If not same tier, failover degrades performance
* Different workload characteristics possible but risky
* Downsized secondary saves cost but reduces protection
* Test failover with planned tier to validate
* Scaling up during failover takes time
* Most organizations use same tier

Q95: How do you handle licensing during DR scenarios?

* Azure Hybrid Benefit applies in secondary region
* SQL Server licenses work in DR region
* Licensing cost part of DR budget
* Free trial subscriptions don't count
* Reserved instances work in both regions
* Spot instances not suitable for DR
* Plan licensing as part of cost

Q96: What is the impact of schema changes on replicas?

* Schema must match between primary and secondary
* Replication forwards DDL statements
* Both must apply changes identically
* Locks during schema change affect replication
* Large table changes may cause lag
* Test schema changes on non-production replica first
* Coordinate schema deployments carefully

Q97: How do you handle index maintenance on replicas?

* Maintenance applies only to primary
* Replication forwards maintenance operations
* Secondary eventually catches up
* Large index maintenance causes replication lag
* Schedule maintenance during low-traffic times
* Parallel operations reduce lag
* HA replicas may lag during maintenance

Q98: What is the purpose of trace flags in failover scenarios?

* Special SQL Server behavior modifications
* Some trace flags help with replication
* Documented trace flags only
* Unsupported trace flags risky
* Require SQL Server restart typically
* Share across all replicas identically
* Generally avoid unless necessary

Q99: How do you prevent accidental data loss during testing?

* Use failover group testing features
* Never delete production database
* Separate test failover group recommended
* Clear labeling of test versus production
* Remove test resources after testing
* Automation reduces manual errors
* Runbooks prevent accidental destructive actions

Q100: What is the purpose of disaster recovery drills?

* Validate that procedures actually work
* Practice team coordination
* Identify gaps in documentation
* Build confidence in procedures
* Measure actual recovery metrics
* Discover unexpected dependencies
* Regular drills keep team sharp


ADVANCED LEVEL (Questions 101-150)

Q101: Design a complete HA/DR architecture for globally distributed financial application with sub-minute RTO and near-zero RPO.

* Tier selection
  * Business Critical for sub-30-second local failover
  * Synchronous replication ensures RPO compliance
  * Automatic failover to meet RTO target
* Geo-replication configuration
  * Failover group to secondary region
  * Automatic failover policy enabled
  * Grace period minimal but tuned to prevent false positives
* Zone redundancy
  * Within primary region across availability zones
  * Within secondary region similarly
  * Defense in depth approach
* Validation
  * Planned failover tests quarterly minimum
  * Load testing in secondary region
  * Document actual observed RTO/RPO
  * Update architecture if actual exceeds target

Q102: Explain synchronous replication in Business Critical tier architecture.

* Write commit process
  * Application submits write
  * Primary writes to transaction log
  * Primary requests acknowledgment from secondaries
  * Quorum of secondaries must acknowledge
  * Primary confirms write to application only after quorum
* RPO guarantee
  * No committed transactions lost during failover
  * Synchronous means guarantee if system healthy
  * Degradation under heavy load possible
  * Sustained high lag indicates capacity issue
* Forced failover exception
  * Forced failover may lose unacknowledged transactions
  * Planned failover loses nothing
  * Async geo-replication to different region may lose data

Q103: Design HA/DR for Hyperscale database at extreme scale.

* Hyperscale characteristics
  * Very large database support terabytes to petabytes
  * Distributed storage architecture
  * Page servers handle database pages
  * Different failover characteristics than other tiers
* HA configuration
  * HA replicas within region for failover
  * Separate from named replicas
  * Both types necessary for different purposes
* Geo-replication
  * Test failover timing at actual scale
  * Benchmark shows behavior at size
  * May perform differently than smaller databases
* Named replicas
  * Analytics workloads on separate named replicas
  * Not part of automatic failover
  * Different placement strategy

Q104: Architect HA/DR for multi-tenant sharded database solution.

* Shard topology
  * Each shard gets failover group
  * Consistent across all shards
  * Shard map database itself geo-replicated
* Failover orchestration
  * Atomic failover of all shards together
  * Partial failover leaves system inconsistent
  * Automation essential to coordinate
  * Runbook documents exact sequence
* Shard map updates
  * Must reflect new primary locations
  * DNS changes for failover group endpoints
  * Routing layer uses updated map
  * Test with realistic shard distribution

Q105: Handle HA/DR for Always Encrypted columns during failover.

* Encryption key management
  * Column master keys required for decryption
  * Typically in Azure Key Vault
  * Key Vault must be geo-redundant
* Failover scenario
  * Application must access keys from secondary region
  * Key Vault endpoint resolution in secondary
  * RBAC access must work from secondary region
  * Test encryption key retrieval during DR test
* Failback scenario
  * Keys must still be accessible
  * No key rotation during failover
  * Failback same requirements as failover

Q106: Design backup and restore strategy complementing geo-replication.

* Three-layer protection
  * Geo-replication for fast regional failover
  * Long-term retention for point-in-time recovery
  * Cross-region backup copies
* Use cases
  * Geo-replication for regional outage
  * Backup for data corruption scenarios
  * Long-term retention for compliance
  * All three layers provide different protection
* Testing
  * Regularly test restore from backup
  * Restore to test environment
  * Validate data integrity after restore
  * Document restore procedure

Q107: Handle Distributed Transactions across databases in Managed Instance failover.

* Distributed transaction challenges
  * Span multiple databases within instance
  * Failover affects all databases atomically
  * Transaction coordination complex
* Failover behavior
  * All databases promoted together
  * Distributed transaction state persists
  * Some transactions may abort
* Application handling
  * Retry logic for aborted transactions
  * Avoid multi-database transactions if possible
  * Use local transactions when feasible
  * Test distributed transaction failover

Q108: Design HA/DR for mission-critical application running 24/7.

* Continuous operation requirement
  * Planned tests only during maintenance windows
  * Failover must be rehearsed repeatedly
  * No downtime acceptable
* Architecture choices
  * Business Critical tier mandatory
  * Multiple secondary regions possible
  * Automatic failover essential
  * Read scale-out for load distribution
* Testing approach
  * Small failover group for testing
  * Separate from production failover group
  * Doesn't affect live traffic
  * Run tests during all hours to cover shift handoffs

Q109: Implement cost-optimized HA/DR for non-critical development environment.

* Cost reduction strategies
  * General Purpose tier adequate
  * Cold standby instead of hot standby
  * Geo-restore as DR mechanism
  * No redundancy during development
* Trade-offs accepted
  * Longer RTO acceptable for dev
  * Some data loss acceptable
  * Rebuild capability sufficient
* Activation during issues
  * Geo-restore only when needed
  * Minimal ongoing cost
  * Restore process documented
  * Testing periodic only

Q110: Design HA/DR for hybrid environment with on-premises data.

* Separation of concerns
  * Azure SQL HA/DR separate from on-prem
  * On-prem needs own traditional DR
  * Different technologies and strategies
* Connectivity layer
  * ExpressRoute or VPN between sites
  * Redundancy critical for connectivity
  * Failover must account for link redundancy
* Data synchronization
  * Azure Data Factory for replication
  * Log shipping for near-real-time sync
  * Backup and restore for periodic sync
  * All have different RTO/RPO characteristics

Q111: Manage Automatic Tuning state during failover transitions.

* Tuning state considerations
  * Forced plans may not apply to secondary
  * Index recommendations specific to primary
  * Statistics may be different in secondary
  * Query optimization differs between regions
* Post-failover validation
  * Monitor query performance after failover
  * Forced plans may not be optimal in new region
  * Consider disabling Automatic Tuning during major failovers
  * Re-enable after stability achieved
  * Review and clean up invalid tuning

Q112: Communicate HA/DR trade-offs to business stakeholders.

* Business language translation
  * RTO measured in business impact terms
  * RPO measured in transaction value terms
  * Availability expressed as percentage uptime
* Cost-benefit analysis
  * Total cost of ownership for each architecture
  * Cost per minute of downtime avoided
  * Compliance requirements driving choices
  * Compare to cost of downtime
* Residual risk transparency
  * No system achieves absolute zero risk
  * Catastrophic scenarios still possible
  * Mitigation strategies explained
  * Acceptance of some risk necessary

Q113: Conduct post-incident analysis for failed failover.

* Timeline reconstruction
  * When outage first detected
  * When failover initiated
  * When failover completed
  * When service restored to users
  * When failback occurred
* Data analysis
  * Actual data loss measured
  * Transactions lost identified
  * Cause of data loss determined
* Procedure review
  * Steps that worked correctly
  * Steps that failed or delayed
  * Undocumented steps taken
  * Decisions that needed refinement
* Continuous improvement
  * Update runbooks based on findings
  * Adjust monitoring based on blind spots
  * Retrain team on lessons learned
  * Schedule follow-up testing

Q114: Design for graceful degradation when failover partially succeeds.

* Partial failure scenario
  * Some databases failover, others don't
  * Application unable to proceed
  * Database partially inconsistent
* Graceful handling
  * Application detects inconsistency
  * Enters read-only mode temporarily
  * Queues transactions for later replay
  * Notifies users of temporary limitation
  * Continues providing degraded service
* Recovery
  * Failback when original region healthy
  * Replay queued transactions
  * Validate data consistency
  * Resume normal operations

Q115: Plan for security token expiration during regional failover.

* Token considerations
  * Access tokens have expiration time
  * Failover may take longer than token life
  * Refresh tokens needed to obtain new tokens
  * Different scenario for each token type
* Application handling
  * Background token refresh capability
  * Fallback to credential re-entry
  * Queue operations until tokens valid
  * Test token handling during failover
* Infrastructure
  * Token endpoint accessible from secondary
  * Same identity provider in both regions
  * No region-specific token restrictions

Q116: Implement chaos engineering tests for database failures.

* Test scenarios
  * Forced failover under load
  * Network latency and packet loss
  * Cascading failures of dependencies
  * Concurrent failures
  * Extended outage duration
* Execution approach
  * Scheduled during test windows
  * Non-production environment preferred
  * Controlled blast radius
  * Monitoring captures all telemetry
* Analysis
  * Where did system fail
  * How quickly detected
  * Recovery effectiveness
  * Gaps in procedures or monitoring

Q117: Design for zero-downtime schema migrations.

* Approach options
  * Big-bang migration with downtime
  * Blue-green deployment strategy
  * Online schema change tools
  * Separate read and write paths
* Implementation
  * Verify replica has schema change capability
  * Test on non-production replica
  * Coordinate timing across regions
  * Validation that new schema works
  * Rollback plan if needed

Q118: Handle connection pooling challenges during failover.

* Pool problems during failover
  * Stale connections to old primary
  * Pool still holds connections to down server
  * New connections get correct primary
  * Mix of working and broken connections
* Solutions
  * Connection timeout forces reconnection
  * Health check invalidates bad connections
  * Drain and recreate pool periodically
  * Application restart cleanest option
  * Test pool behavior during failover

Q119: Optimize network latency for geo-replication.

* Latency impact
  * Replication lag related to latency
  * Write commit time affected
  * Failover timing not affected by latency
  * Application query performance affected
* Optimization
  * Choose closest secondary region
  * Use ExpressRoute instead of VPN
  * Reduce network hops where possible
  * Prioritize database replication traffic
  * Monitor replication lag under load

Q120: Plan for license compliance during failover.

* License scenarios
  * Azure Hybrid Benefit licenses
  * Reserved instances in both regions
  * Pay-as-you-go fallback
  * Mixed licensing models
* Failover considerations
  * Ensure licenses valid in both regions
  * No additional licenses needed in secondary
  * Failback uses same licenses
  * Temporary cost increase during failover acceptable
  * Plan licensing budget accordingly

Q121: Design monitoring strategy for hybrid workloads.

* Visibility requirements
  * On-premises metrics need integration
  * Azure metrics collected separately
  * Single pane of glass preferred
  * Cross-premises alerting critical
* Implementation
  * Log Analytics workspace central
  * Application Insights for app tier
  * On-prem agents reporting to cloud
  * Custom metrics for business KPIs
  * Correlated alerting across systems

Q122: Implement automated failover tests.

* Automation approach
  * Scheduled failover during low traffic
  * Automated validation afterward
  * Automated failback to original
  * Daily or weekly execution possible
  * Non-impacting to production
* Validation
  * Health checks post-failover
  * Application connectivity tests
  * Data consistency checks
  * Performance baseline comparison
  * Alert if anomalies detected
* Results
  * Historical failover metrics
  * Trend analysis for degradation
  * Alert if test fails
  * Document any manual interventions

Q123: Handle identity and access during regional failover.

* Authentication scenarios
  * Managed identity tokens expire
  * Service principal credentials needed
  * User identities may not span regions
  * Multi-factor authentication may fail
* Cross-region identity
  * Azure AD available in all regions
  * Managed identities work cross-region
  * Connection identity must be valid in secondary
  * Test authentication from secondary region
  * User access permissions replicate

Q124: Design for application tier failover synchronization.

* Coordination challenge
  * Database and application tier may failover at different times
  * Application may failover before database
  * Database failover may complete before app tier
  * Mismatch causes connection failures
* Solutions
  * Orchestrate together if possible
  * Application waits for database
  * Or database waits for application
  * Health checks validate readiness
  * Test combined failover

Q125: Implement split-brain prevention in multi-region setup.

* Split-brain problem
  * Both primary and secondary think they are primary
  * Conflicting writes occur
  * Data inconsistency results
  * Communication between regions lost
* Prevention
  * Quorum-based approach
  * Master in one region only
  * Automatic failover on detection
  * Manual override requires safeguards
  * Monitoring detects conflict

Q126: Design for observability during incidents.

* Observability requirements
  * Real-time visibility into failures
  * Tracing of requests through system
  * Metrics at all layers
  * Logs from all components
  * Event correlation across systems
* Implementation
  * Distributed tracing enabled
  * Metrics collected at high frequency
  * Logs centralized immediately
  * Alerts trigger on key indicators
  * Custom dashboards for incident response

Q127: Plan geo-replication for read-heavy reporting workloads.

* Workload characteristics
  * Reporting queries isolated to secondary
  * Reduces load on primary
  * Reporting doesn't interfere with OLTP
  * Secondary may use read scale-out
* Implementation
  * Reporting application uses secondary
  * Connection string specifies secondary
  * Reports have delayed data freshness
  * Configure acceptable lag tolerance
  * Monitor reporting performance

Q128: Design update strategy for HA/DR procedures.

* Procedure management
  * Version control for runbooks
  * Change approval process
  * Regular review cycles
  * Keep procedures current
  * Incorporate lessons learned
* Validation
  * Test updated procedures
  * Dry run before next incident
  * Team training on changes
  * Clear communication of updates

Q129: Handle very large database restore scenarios.

* Scale challenges
  * Restore time measured in hours
  * Bandwidth constraints during restore
  * Storage space requirements during restore
  * CPU overhead of restore operation
* Optimization
  * Parallel restore if possible
  * Restore during off-hours
  * Pre-allocate storage
  * Monitor resource consumption
  * Test restore timing

Q130: Design for regulatory compliance in DR.

* Compliance scenarios
  * Data residency requirements
  * Audit trail requirements
  * Encryption requirements
  * Access control audit trail
  * Retention policy compliance
* Implementation
  * Choose regions meeting requirements
  * Audit logging enabled
  * Encryption for data in transit and rest
  * Access logged and auditable
  * Compliance validated regularly

Q131: Implement cost monitoring for HA/DR.

* Cost components
  * Secondary region compute cost
  * Storage replication cost
  * Network bandwidth cost
  * Backup storage cost
  * Licensing cost in secondary
* Monitoring
  * Budget alerts for each component
  * Cost trend analysis
  * Right-sizing recommendations
  * Compare to RTO/RPO value
  * Justify costs to business

Q132: Design for multi-language application databases.

* Collation complexity
  * Different languages need different collation
  * Must match across primary and secondary
  * Replication requires identical collation
  * Case sensitivity affects queries
* Planning
  * Determine collation requirements
  * All replicas use same collation
  * Test case-sensitive queries
  * Document collation choice
  * Cannot change after creation

Q133: Handle database compatibility level during failover.

* Compatibility considerations
  * Compatibility level affects query behavior
  * Must match between primary and secondary
  * Upgrade process requires testing
  * Failover may reveal compatibility issues
* Process
  * Test compatibility level change
  * Apply to secondary first
  * Verify no query issues
  * Apply to primary
  * Monitor for unexpected behavior

Q134: Design update strategy for Azure SQL features.

* Feature updates
  * Azure provides updates automatically
  * No downtime for most updates
  * Uses failover for Business Critical
  * General Purpose restarts compute
* Planning
  * Understand update frequency
  * Consider update timing
  * Monitor for issues after updates
  * Have rollback plan if needed
  * Log update impacts

Q135: Implement high-frequency failover testing.

* Testing frequency
  * Daily automated tests possible
  * Weekly team-involved tests
  * Monthly full end-to-end tests
  * Quarterly disaster simulation
  * Annual compliance audit
* Test types
  * Automated connectivity tests
  * Application functionality tests
  * Load testing failover capacity
  * Chaos engineering adversarial tests

Q136: Design for organizational continuity during incidents.

* People considerations
  * On-call team availability
  * Communication procedures
  * Escalation paths
  * Decision making authority
  * Training and drills
* Communication
  * Internal status updates
  * Customer communications
  * Media communications if needed
  * Post-incident analysis communication

Q137: Handle failover when primary region partially recovers.

* Ambiguous situation
  * Some services recover, others don't
  * Unknown if recovery sustainable
  * Uncertain if failback is correct choice
  * Risk of oscillating failovers
* Decision process
  * Wait for complete recovery before failback
  * Validate stability before failing back
  * Plan for extended dual-primary state
  * Communicate timeline to stakeholders
  * Document decision rationale

Q138: Design multi-tier application failover coordination.

* Application layers
  * Web tier
  * Application logic tier
  * Cache layer
  * Database tier
  * Message queue tier
* Coordination
  * Each tier fails over independently
  * Must coordinate timing
  * Application tier waits for database
  * Cache invalidated after failover
  * Messages replayed if lost

Q139: Implement vendor requirements in DR design.

* Vendor constraints
  * Some vendors restrict multi-region
  * License compliance requirements
  * Support model restrictions
  * Data residency mandates
  * Uptime guarantees limit DR options
* Planning
  * Document vendor restrictions
  * Work within constraints
  * Communicate limitations to business
  * Evaluate vendor fitness
  * Consider alternatives if too restrictive

Q140: Handle timeouts and connection resets during failover.

* Connection behavior
  * Existing connections drop
  * Connection timeout occurs
  * Connection reset errors seen
  * Some clients retry, others don't
* Application handling
  * Implement connection retry logic
  * Exponential backoff prevents overwhelming
  * Jitter prevents thundering herd
  * Timeout configuration appropriate
  * Test retry behavior under failover

Q141: Design for data drift detection.

* Data drift causes
  * Replication lag causing momentary mismatch
  * Application-level inconsistency
  * Bugs in replication logic
  * Cascading failures affecting multiple systems
* Detection
  * Checksum validation between primary and secondary
  * Row count comparison
  * Query result comparison
  * Automated drift detection jobs
  * Alert if drift detected

Q142: Implement defensive programming for failovers.

* Application patterns
  * Assume connections will fail
  * Retry failed operations
  * Validate data after recovery
  * Tolerate stale data temporarily
  * Don't assume availability
* Coding
  * Retry logic in critical paths
  * Timeouts on all calls
  * Circuit breaker pattern
  * Graceful degradation
  * Comprehensive error handling

Q143: Design versioning strategy for schema and data.

* Schema versioning
  * Track schema versions
  * Support old and new schema temporarily
  * Migrate data gradually
  * Replication between different versions
  * Validate migration completeness

Q144: Handle secrets rotation during failover.

* Secret types
  * Database credentials
  * Encryption keys
  * API keys
  * Connection strings
* Rotation challenges
  * Cannot rotate during failover
  * Timing coordination difficult
  * New secrets not available immediately
  * Old secrets must remain valid
* Strategy
  * Rotate before incident if possible
  * Delay rotation if incident ongoing
  * Validate secret access post-failover

Q145: Implement observability for replication health.

* Metrics to track
  * Replication lag in seconds
  * Transaction throughput
  * Network bandwidth utilization
  * CPU on secondary during replication
  * Disk I/O on secondary
  * Memory utilization on secondary
* Alerting
  * Alert if lag exceeds threshold
  * Alert if lag trending up
  * Alert if bandwidth maxed
  * Alert if secondary CPU high

Q146: Design dependency mapping for DR.

* Dependencies to identify
  * Database external references
  * Jobs and automation
  * ETL processes
  * Reporting tools
  * Analytics systems
  * Third-party integrations
* Mapping
  * Catalog all dependencies
  * Identify which fail over
  * Identify which need manual intervention
  * Prioritize based on impact
  * Test each dependency post-failover

Q147: Implement rate limiting for failover events.

* Prevention strategy
  * Prevent cascading failovers
  * Throttle automatic failover
  * Grace period between failovers
  * Manual intervention for rapid re-failover
  * Alert on rapid failover pattern
* Implementation
  * Cooldown period after failover
  * Maximum failover attempts per hour
  * Escalation to manual control
  * Prevent flip-flopping between regions

Q148: Design for consistency during eventual failback.

* Failback challenges
  * Original primary may have missed updates
  * Replicate changes back during failback
  * Ensure no duplicate updates
  * Validate consistency after failback
* Process
  * Sync changes from new primary before failback
  * Validate consistency before promotion
  * Test failback before doing live
  * Monitor closely during failback

Q149: Implement tenant isolation in multi-tenant DR.

* Isolation requirements
  * One tenant failover shouldn't affect others
  * Resource contention avoided
  * Data privacy maintained across failover
  * Blast radius limited
* Implementation
  * Separate failover groups per tenant
  * Dedicated resources per tenant
  * Resource quotas enforced
  * Monitoring per tenant
  * Billing per tenant

Q150: Design for zero-knowledge architecture in DR.

* Zero-knowledge principle
  * Encryption keys never leave client
  * Service cannot decrypt data
  * DR still possible with encryption
  * Failover works on ciphertext
* Implementation challenges
  * Key management complexity
  * Client must manage keys
  * Recovery with client unavailable difficult
  * Performance implications
  * Balance security and usability


REAL-TIME SCENARIOS (Questions 151-200)

Q151: Production database suddenly unreachable. First action?

* Check Service Health for regional issues
* Check Resource Health for specific database
* Confirm issue not on application side
* Check network connectivity to database
* Review recent application or database changes
* Communicate status to stakeholders immediately
* Determine if failover necessary based on findings

Q152: Failover triggered automatically but primary still responding. What happened?

* Grace period expired and platform decided to failover
* Primary may be degraded, not completely down
* Health monitoring detected timeout or high latency
* Failover was preventive, not reactive
* Communication lag between regions
* Possible cascading failure not immediately visible
* Review Resource Health for what triggered failover

Q153: After failover, application cannot decrypt Always Encrypted columns.

* Column master keys not accessible in secondary region
* Key Vault not geo-redundant or access restricted
* RBAC permissions not configured for secondary
* Immediate mitigation
  * Disable Always Encrypted for this session if possible
  * Use app-level fallback without decryption
  * Route to primary if still available
* Fix
  * Configure Key Vault geo-redundancy
  * Validate RBAC allows secondary access
  * Test key retrieval from secondary region

Q154: Replication lag suddenly spikes during failover test.

* Secondary region capacity insufficient
* Network bandwidth saturated
* Heavy workload hitting secondary simultaneously
* Long-running query on secondary blocking replication
* Possible CPU saturation on secondary
* Possible disk I/O saturation
* Investigation
  * Check secondary tier matches primary
  * Monitor CPU, memory, disk during test
  * Reduce concurrent workload
  * Increase secondary tier if needed

Q155: Failback to primary fails with consistency error.

* Data diverged between primary and secondary
* Transactions on new primary don't exist on original
* Cannot reconcile state for failback
* Solution options
  * Fail forward, stay on secondary
  * Manual data reconciliation
  * Restore from backup to reconcile
  * Replay missing transactions carefully
* Prevention
  * Maintain bidirectional replication if possible
  * Test failback scenarios regularly
  * Monitor data consistency

Q156: Customer reports they lost recent transactions after failover.

* Replication lag caused data loss
* Forced failover not fully synchronized
* Expected RPO different than actual
* Immediate actions
  * Confirm what transactions lost
  * Check if in backup or log
  * Communicate timeline to customer
  * Offer recovery options
* Investigation
  * Analyze replication lag patterns
  * Determine if RPO was realistic
  * Check if secondary properly sized
  * Review forced vs planned failover trigger

Q157: Network connectivity between regions becomes unavailable during incident.

* Geo-replication halts immediately
* RPO starts increasing
* Cannot perform graceful failover
* Immediate actions
  * Detect network issue quickly
  * Alert operations team
  * Prepare forced failover if necessary
  * Attempt network restoration
* During outage
  * Switch to cold standby if network stays down
  * Coordinate with network team
  * Document incident timeline

Q158: Secondary region health check fails repeatedly right before critical deadline.

* Timing could be worse
* Cannot validate readiness
* Uncertainty about failover success
* Proceed cautiously
  * Do not failover if untested
  * Extend deadline if possible
  * Run targeted tests instead of full DR test
  * Fix health check and validate
  * Document reason for not testing

Q159: Application continues working after failover but returns stale data.

* Read queries going to old primary still
* Connection string has hardcoded primary server
* Load balancer still routing to old primary
* Not using failover group listener
* Resolution
  * Update connection string to failover group endpoint
  * Restart application to pick up configuration
  * Validate read-only routing if using ApplicationIntent
  * Implement DNS reload in connection logic

Q160: Managed identities stop working after failover.

* Managed identity tokens may have region affinity
* Scope not configured for secondary region
* Resource access denied in secondary
* Mitigation
  * Verify managed identity has access in secondary
  * Use credentials instead temporarily
  * Test managed identity from secondary region
  * Configure cross-region resource access

Q161: Cascading failure takes down secondary region shortly after failover.

* Primary down, failover to secondary completes
* Secondary immediately fails
* System now completely down
* Emergency recovery
  * Activate cold standby in third region if available
  * Restore from most recent backup
  * Prepare for significant downtime
  * Communicate to customers
* Post-incident
  * Investigate why secondary failed
  * Were capacity requirements met
  * Was there hidden dependency

Q162: Database tier chosen during failover cannot handle production load.

* Failback to same tier as primary but replicated
* Secondary chose lower tier to save cost
* Load causes timeout and errors
* Immediate actions
  * Scale secondary tier up during failover
  * Takes time, application suffers
  * Accept degraded performance temporarily
  * Prioritize critical workloads
* Prevention
  * Secondary must match primary tier
  * Load test secondary with production data
  * Cost-benefit analysis before downsizing

Q163: License agreements prohibit operating in secondary region.

* Vendor restriction blocks failover
* Cannot legally run in secondary
* Dramatic problem during crisis
* Options
  * Emergency waiver from vendor
  * Switch to licensed product
  * Violate license and deal with consequences later
  * Accept unavailability
* Prevention
  * Audit vendor agreements before crisis
  * Clarify secondary region usage
  * Get written approval for DR scenario

Q164: After failover, firewall rules don't match between regions.

* Connection strings work but queries fail
* Security rules blocking access
* Network ACLs different between regions
* Debugging challenge
  * Connections succeed but queries timeout
  * Looks like database issue but network is culprit
  * Check firewall rules in secondary
  * Validate same rules in both regions

Q165: DNS changes during failover don't propagate to all client locations globally.

* Some clients still connecting to old primary
* Split-brain scenario with partial failover
* Some users experience failures, others don't
* Geographic routing for DNS changes
* TTL too high, changes slow to propagate
* Clients ignoring TTL
* Mitigation
  * Lower TTL before expected failover
  * Force DNS refresh on clients
  * Dual-primary temporary state
  * Wait for global DNS propagation

Q166: Transaction log fills secondary region storage during failover.

* Replication cannot continue
* Secondary storage exhausted
* Immediate intervention needed
  * Delete transaction logs if possible
  * Extend storage capacity
  * Throttle incoming writes temporarily
  * Prioritize critical workloads
* Prevention
  * Monitor log growth
  * Allocate sufficient storage in secondary
  * Understand log growth patterns

Q167: Critical job scheduled during failover window executes against wrong database.

* Job runs on old primary assuming primary
* Job runs on secondary, data changes conflict
* Data inconsistency results
* Immediate investigation
  * Identify which job ran where
  * Assess data impact
  * Determine if reconciliation needed
  * Restart affected processes
* Prevention
  * Disable jobs during failover window
  * Jobs aware of which is primary
  * Idempotent job design
  * Test job behavior during DR test

Q168: Backup storage in secondary region deleted before failover completes.

* Recent backups only in primary region
* Secondary region has no backup recovery option
* Geo-restore not possible from deleted backups
* Cannot recover if data corrupted
* Incident response
  * Geo-restore from backup in primary if available
  * Restore to secondary after failover completes
  * Accept potential data loss
* Prevention
  * Long-term retention in multiple regions
  * Automate backup replication
  * Backup storage protected like production

Q169: After failover, connection pools on application tier remain stale.

* Old connections still point to old primary
* New connections get routed correctly
* Partial application functionality works
* Some users see errors
* Resolution
  * Force connection pool refresh
  * Restart application tier
  * Reduce connection timeout to force reconnection faster
  * Scale up application instances to bypass stale pool

Q170: Schema changes queued in deployment pipeline complete during failover.

* Schema change applied only to primary
* Secondary doesn't have schema change
* Replication breaks due to mismatch
* Data corruption or loss possible
* Immediate action
  * Halt deployment during failover
  * Manually apply schema to secondary if needed
  * Validate replication after schema applied
  * Test critical queries
* Prevention
  * Pause deployments during DR testing
  * Schema changes applied to both regions
  * Coordination between database and app teams

Q171: Client certificate authentication fails after failover to different region.

* Certificates issued for specific server name
* Failover group endpoint different name
* Certificate validation fails
* HTTPS connections cannot establish
* Mitigation
  * Update certificates for failover group endpoint
  * Wildcard certificate simplifies multi-region
  * Test certificate validation during DR test
  * Document certificate management

Q172: Reporting queries that depend on recent data fail after failover to read-only secondary.

* Application still using old query routing
* Lag in secondary means reports inaccurate
* Business relies on fresh data
* Cannot accept staleness during incident
* Options
  * Route critical reports to primary if available
  * Accept data staleness temporarily
  * Redirect reports to batch process
  * Wait until failback to resume accurate reporting

Q173: Encryption key rotation scheduled for middle of incident.

* Worst possible timing
* Cannot proceed with key rotation
* Key rotation blocks other operations
* Immediate action
  * Cancel scheduled key rotation
  * Delay until incident resolved
  * Communicate to security team
  * Retry after failback complete

Q174: Application attempt to connection failover immediately after deployment.

* Deployment added failover group usage
* Failover happens before application tested
* Untested code path in production
* Catastrophic failure possible
* Incident response
  * Rollback deployment if possible
  * Switch to manual failover mode
  * Use tested failover processes
  * Deploy fix and test before retry
* Prevention
  * Test failover group changes thoroughly
  * Staged rollout of new endpoints
  * Monitoring for connection failures

Q175: Cold standby restore takes longer than RTO allows.

* Restore from backup slower than acceptable
* RTO of four hours but restore takes six hours
* Database size increased since baseline
* Cannot complete recovery in time
* Immediate action
  * Restore to larger compute size if available
  * Parallel restore if supported
  * Accept exceeding RTO if necessary
  * Communicate delay to stakeholders
* Prevention
  * Test restore time at current database size
  * Use hot standby if RTO critical
  * Plan restore procedures carefully

Q176: Monitoring and alerting infrastructure itself fails during incident.

* Monitoring cannot see primary down
* Alerts not triggered
* Team doesn't know about incident
* Discovery delayed significantly
* Parallel issue
  * Database problem and monitoring problem
  * Double impact on recovery
  * Manual detection needed
  * Dramatic incident management challenge
* Prevention
  * Monitor the monitors
  * Redundant monitoring systems
  * Backup alerting mechanisms
  * On-call team checks independently

Q177: Audit logging stopped during failover and compliance audit later finds gap.

* Compliance breach due to logging gap
* Audit trail missing during incident
* Regulatory requirements not met
* Significant compliance consequences
* Investigation
  * Determine when logging stopped
  * Why logging failed during failover
  * Assess compliance impact
  * Document incident for auditor
* Prevention
  * Audit logging setup same in both regions
  * Verify logging works during failover test
  * Monitoring for logging failures
  * Compliance team aware of known gaps

Q178: Customer supports other customers who depend on your database availability.

* Cascade effect amplifies incident impact
* Your unavailability causes their customers unavailable
* Incident escalates beyond direct impact
* Urgent remediation needed
* Communication challenge
  * Direct customers need updates
  * Their customers need updates
  * Ecosystem communication critical
* Resolution
  * Incident response team focused
  * Communication rapid and frequent
  * Transparency about timeline
  * Post-incident support for affected parties

Q179: Failover group accidentally created in production without testing.

* No baseline for what to expect
* Unknown if secondary configuration correct
* No runbook for this failover group
* First time it's used is during real incident
* High risk scenario
* Immediate action
  * Do not rely on untested failover
  * Use cold standby restore instead
  * Run quick smoke test of failover group
  * If failover necessary anyway, extreme caution
* Prevention
  * Require testing before production deployment
  * Failover group creation includes test run
  * Documentation before production use

Q180: Multi-region failover coordinated manually becomes confused about which is primary.

* Manual processes error-prone
* Wrong region designated as primary
* Writes split between regions
* Data corruption results
* Immediate containment
  * Identify which region actually primary
  * Stop writes to secondary immediately
  * Reconcile data divergence
  * Designate single primary region
  * Replay missed transactions if possible
* Prevention
  * Automation enforces single primary
  * Quorum-based approach prevents ambiguity
  * Clear procedures and checklist
  * Strict adherence to procedures

Q181: Recent configuration changes deployed right before incident.

* Unclear if changes contributed to incident
* Changes reversed during troubleshooting
* Actual cause obscured
* Incident response complexity increased
* Investigation challenge
  * Separate change impact from incident cause
  * Test if reverting change helps
  * Monitor impact of reverting changes
  * Document relationship if any
* Prevention
  * Change management during incident
  * No changes allowed during active incident
  * Coordinate changes with maintenance window
  * Test changes thoroughly before production

Q182: Multiple teams claiming authority over failover decision.

* Database team, infrastructure team, platform team
* Unclear who has decision authority
* Delays incident response
* Decision not made while debating
* Governance breakdown
  * Immediate clarification needed
  * Executive decision required
  * Follow escalation path
  * Senior person decides
* Prevention
  * Incident command structure defined
  * Clear chain of command
  * Documented authority levels
  * Training on escalation procedures

Q183: Failover succeeds but application debugging becomes much harder in secondary region.

* Development tools not configured same way
* Debugging tools not present
* Logs not accessible from secondary
* Production support team needs to investigate
* Operational nightmare
  * Cannot reproduce locally
  * Cannot step through code
  * Only have production monitoring
  * Difficulty tracking down bugs
* Prevention
  * Mirror development environment in secondary
  * Same tools and configurations
  * Debugging capability in secondary
  * Documentation of setup differences

Q184: Failover changes latency characteristics, breaking application performance assumptions.

* Application designed for primary region latency
* Secondary region has different network characteristics
* Queries suddenly timeout
* Expected throughput not achieved
* Performance debugging
  * Application may handle differently
  * Query plans may differ
  * Indexes differently effective
  * Data distribution different
* Resolution
  * Accept degraded performance temporarily
  * Optimize for secondary region characteristics
  * Scale up secondary for performance
  * Cache more aggressively
  * Batch operations instead of real-time

Q185: Failover discovered dependency on resource outside Azure.

* On-premises system not reachable from secondary region
* Network path different
* Firewall rules different
* On-prem system has failover of its own
* Cascading failure if on-prem down too
* Recovery
  * Restore on-premises connectivity
  * Route through different path if possible
  * Coordinate on-premises failover timing
  * Accept temporary unavailability of feature

Q186: Business decides to fail back to primary immediately without testing.

* Pressure to return to "normal"
* Skip validation and testing
* Fallback may fail
* High risk decision
* Counsel of caution
  * Validate primary region is stable
  * Test failback procedure
  * Confirm data consistency before failback
  * Communication about risks
  * Proceed with monitoring if decision stands
* Damage from fallback failure
  * Back to secondary scenario again
  * Customers confused by second outage
  * Business confidence shaken
  * Repeat incident recovery

Q187: Failover uncovers data inconsistency not noticed before.

* Data corruption in primary replicated to secondary
* Failover reveals inconsistency
* Both copies corrupted
* Cannot recover from corruption
* Investigation
  * When did corruption start
  * What caused data inconsistency
  * How much data affected
  * Business impact assessment
* Recovery options
  * Geo-restore to pre-corruption point
  * Manual correction if identifiable
  * Accept data loss if minor
  * Customer communication and compensation

Q188: Customer database size exceeds secondary tier allocation during failover.

* Secondary storage full
* No space for replicated data
* Failover cannot complete
* Immediate action
  * Expand secondary storage
  * May take time despite cloud convenience
  * Disable non-critical features to save space
  * Archive data if possible
  * Increase secondary tier
* Prevention
  * Secondary storage sized for growth
  * Monitor storage growth trending
  * Capacity planning for six months out

Q189: Failover triggered for wrong database due to operator error.

* Wrong database failed over
* Correct database still down
* Wrong database has failover group consumed
* Application using wrong database
* Major incident
  * Failback immediately
  * Failover correct database next
  * Application may not handle change
  * Data consistency issues possible
  * Communication nightmare
* Prevention
  * Clear naming conventions
  * Checklists prevent wrong target
  * Confirmation step before failover
  * Automation reduces human error

Q190: Message queue fails during failover causing applications to crash.

* Application depends on message queue
* Queue also needs failover strategy
* Failover only handles database
* Message loss or duplicates
* Application restart failures
* Incident management
  * Restart application with retry logic
  * Manually process critical messages
  * Coordinate queue and database failover
  * Message replay procedures
* Coordination challenge
  * Different systems with different RTO/RPO
  * Synchronization complex
  * Testing all dependencies together critical

Q191: Legacy system cannot connect to failover group endpoint format.

* Old application doesn't understand DNS redirect
* Hardcoded IP addresses
* Connection string format unsupported
* Cannot easily change legacy system
* Workaround during incident
  * Run reverse proxy to provide old endpoint
  * Host file workarounds temporarily
  * Network redirect at load balancer
  * Accept connection failures for that system
* Long-term fix
  * Update legacy system
  * Modernize connection handling
  * Continuous improvement approach

Q192: Failover completed but database immediately shows as unhealthy.

* Secondary immediately fails after promotion
* Hardware issue in secondary region
* Resource contention
* Cascade failure
* Secondary unusable for incident
* Response
  * Use cold standby instead
  * Try again to secondary region
  * Accept continued outage
  * Manual troubleshooting while down
* Investigation
  * Hardware diagnostics
  * Resource allocation issue
  * Configuration problem
  * Infrastructure issue in region

Q193: During incident response, primary region suffers additional unrelated outage.

* During failover recovery, primary has new issue
* Triple layer failure
  * Primary down for original reason
  * New issue prevents recovery
  * Cannot go back to primary
  * Stuck on secondary
* Extended incident
  * Failover stays in place longer
  * Becomes permanent migration
  * Costs increase substantially
  * Urgent fix needed for primary
* Prevention
  * Fix original issue before failback
  * Validate stability before declaring recovered
  * Multiple independent issues possible

Q194: Customer access restrictions prevent emergency operations during incident.

* Database in restricted region due to compliance
* Cannot access from operations office
* Timing needed for failover difficult
* Personnel cannot directly intervene
* Work-around needed
  * Use bastion host in compliant network
  * Remote session to compliant location
  * Delegate to team member in compliant region
  * VPN to compliant network
* Compliance complex issue
  * Security cannot be bypassed for speed
  * Incident response plan must account
  * Pre-authorize emergency access if possible
  * Automation for authorized operations

Q195: Failover succeeds but team has no way to communicate with management.

* Communication infrastructure also down
* Cannot report status or get decisions
* Incident paralyzed by communication failure
* Blind incident response
* Prepare for communication failures
  * Multiple communication channels
  * Out of band communication planned
  * Phone trees and pre-determined numbers
  * Email for non-real-time updates
  * In-person communication if needed

Q196: Team member attempting failover lacks proper permissions.

* Permission structure changed recently
* Failover attempt fails due to insufficient access
* Delayed response while permissions fixed
* Frustration and confusion
* Immediate remediation
  * Grant emergency permissions
  * Use different team member with access
  * Escalate for permission override
  * Complete failover then audit permissions
* Process improvement
  * Verify permissions before crisis
  * Team members tested for access
  * Failover procedure validated with permissions
  * Regular permission audits

Q197: Failover completes but no documentation of new state.

* Nobody knows what is primary anymore
* Unclear if failback appropriate
* Risk of breaking changes to secondary thinking it's safe
* Operational confusion
* Immediate documentation
  * Document actual current state
  * Primary and secondary locations
  * Failover timestamp
  * Data and RPO status
  * Next steps decided
* Prevention
  * Runbook includes documentation steps
  * Automated state updates to team tools
  * Status dashboard updates automatically
  * Continuous state tracking

Q198: Failover causes unexpected data type conversion issues.

* Primary and secondary store data slightly differently
* Conversion happens during failover
* Data type changed unexpectedly
* Application errors on type mismatch
* Debugging challenge
  * Type differences not obvious
  * Application code expects specific types
  * Casting or parsing failures
  * Performance implications
* Prevention
  * Validate data types match exactly
  * Test failover with real application
  * Type conversion validation
  * Explicit casting in queries

Q199: Application connection pooling default size inadequate after failover.

* Failover with same connection pool size
* Spike in connection demand
* Connection pool maxed out
* Application cannot establish new connections
* Cascading timeout failure
* Immediate workaround
  * Increase connection pool size
  * Restart application to pick up change
  * Throttle requests temporarily
  * Scale application tier
* Prevention
  * Connection pool sized for load spike
  * Load testing with realistic connection patterns
  * Monitoring for pool exhaustion
  * Dynamic pool resizing if possible

Q200: Post-failover analysis shows different root cause than assumed during incident.

* Assumed cause was wrong
* Actual cause different
* Delayed proper remediation
* Procedures based on wrong assumption
* Incident took longer than necessary
* Lessons learned
  * Investigate actual root cause
  * Separate detection from diagnosis
  * Test multiple hypotheses
  * Update procedures with correct knowledge
  * Training team on actual scenario
* Continuous improvement
  * Update incident runbooks
  * Share findings across organization
  * Prevent same assumption mistake again
  * Culture of honest post-incident analysis
