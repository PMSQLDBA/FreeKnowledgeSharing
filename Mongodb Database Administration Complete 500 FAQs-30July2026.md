# MongoDB Database Administration Complete FAQ Guide

## Table of Contents

1. Installation & Setup
2. Configuration & Performance Tuning
3. User Management & Security
4. Database & Collection Operations
5. Replication & High Availability
6. Sharding & Distributed Architecture
7. Backup & Restore Operations
8. Monitoring & Diagnostics
9. Indexing & Query Optimization
10. Disaster Recovery Scenarios
11. Troubleshooting & Operations
12. Migration & Data Transfer

---

## 1. INSTALLATION & SETUP

### Q1: How do I install MongoDB on Ubuntu Linux for production?

- Download the latest stable version from official MongoDB repository
- Add GPG key and configure MongoDB package repository for Ubuntu
- Install MongoDB server package using apt package manager
- Ensure WiredTiger storage engine is included in your installation
- Verify installation by checking mongod version and service status
- Configure systemd to start MongoDB automatically on server reboot

### Q2: What are the minimum system requirements for MongoDB production server?

- RAM should be at least 8GB for single node, 16GB+ for replica sets
- CPU requires minimum 4 cores for production workloads
- Storage disk should have SSD for better I/O performance and IOPS
- Network bandwidth minimum 1Gbps for inter-node communication
- Available disk space should be 2.5 times your expected data size
- File descriptor limit should be set to 64000 or higher

### Q3: How do I configure MongoDB service to run on custom port?

- Edit MongoDB configuration file typically located at /etc/mongod.conf
- Locate the net section and change port from default 27017 to desired port
- Restart MongoDB service using systemctl restart mongod command
- Verify new port binding using ss or netstat command with grep filter
- Update firewall rules to allow traffic on new port
- Test connection using mongosh with --port parameter

### Q4: What is the difference between standalone and replica set deployment?

- Standalone is single MongoDB instance with no redundancy or failover capability
- Replica set consists of 3 or more nodes with automatic failover support
- Replica sets provide automatic data synchronization across all members
- Standalone offers simplicity but replica sets provide high availability
- Replica sets have operational complexity requiring monitoring and management
- Choose replica set for any production system requiring uptime guarantees

### Q5: How do I initialize a MongoDB replica set?

- Start each MongoDB instance with --replSet parameter specifying set name
- Connect to one member using mongosh and run rs.initiate() command
- Define replica set configuration with array of member objects
- Specify priority values where highest priority node becomes primary
- Add arbiter if needed to break ties with even number of members
- Use rs.status() to verify all members are connected and healthy

### Q6: What firewall ports does MongoDB require?

- Primary MongoDB port is 27017 for communication between mongod instances
- Sharded cluster requires 27018 for config servers and 27019 for shard servers
- MongoDB uses ephemeral ports for outbound connections starting at 27000
- Open bidirectional communication on all cluster ports between nodes
- Restrict access to MongoDB ports to only authorized application servers
- Use network security groups or firewall rules to enforce access control

### Q7: How do I enable MongoDB authentication on existing deployment?

- Create initial admin user first with root role before enabling auth
- Edit configuration file and set security.authorization to enabled
- Restart MongoDB service to activate authentication requirement
- All subsequent connections require username and password credentials
- Create application users with minimal required roles and databases
- Store credentials securely and rotate them periodically

### Q8: What is journaling in MongoDB and why is it important?

- Journaling writes all changes to journal file before applying to data files
- Ensures database consistency during unexpected shutdowns or crashes
- WiredTiger storage engine enables journaling by default in production
- Journal files are stored in dbPath/journal directory
- Recovery process replays journal entries to restore consistent state
- Disable journaling only for temporary caching scenarios, never production

### Q9: How do I perform initial MongoDB cluster setup across multiple servers?

- Install MongoDB on all servers with same version for compatibility
- Configure each server with identical configuration file settings
- Set server hostnames and ensure DNS resolution works correctly
- Initialize replica set on primary node with member list
- Verify all nodes connect to primary and maintain synchronization
- Test failover by stopping primary to verify automatic election

### Q10: What storage engine should I choose for MongoDB deployment?

- WiredTiger is default and recommended for all modern deployments
- MMAPv1 is deprecated and should not be used for new installations
- WiredTiger provides compression, better concurrency, and faster recovery
- Storage engine choice is permanent and requires full data migration to change
- Select WiredTiger for all new production MongoDB implementations
- Verify storage engine with db.serverStatus().storageEngine command

---

## 2. CONFIGURATION & PERFORMANCE TUNING

### Q11: How do I configure MongoDB for optimal query performance?

- Enable profiling to identify slow running queries with threshold setting
- Create appropriate indexes on frequently queried fields before production
- Set appropriate readPreference based on application requirements
- Configure connection pooling to reuse connections and reduce overhead
- Adjust cache size in configuration to allocate sufficient memory
- Monitor query execution time and explain plans to identify issues

### Q12: What is working set and how does it affect MongoDB performance?

- Working set is total amount of data your application accesses regularly
- If working set fits in RAM, MongoDB performs optimally with fast queries
- Working set larger than RAM causes frequent disk I/O and slower performance
- Increase physical RAM if working set exceeds available memory
- Monitor RAM usage and cache hit ratios regularly with mongostat tool
- Optimize data model if working set cannot fit in available memory

### Q13: How do I configure MongoDB cache (WT cache) size?

- Cache size defaults to 50% of available RAM in WiredTiger storage engine
- Manually set wiredTiger.engineConfig.cacheSizeGB in configuration file
- Size cache to leave sufficient RAM for operating system and connections
- Monitor cache hit ratio using db.serverStatus() command output
- Increase cache size if hit ratio is below 95% for working set data
- Test configuration changes in staging before applying to production

### Q14: What is the impact of different read concern levels on performance?

- Local read concern returns data immediately without replication wait
- Available read concern includes data not yet replicated to all members
- Majority read concern waits for majority of replicas before returning data
- Linearizable read concern provides strongest consistency but slowest performance
- Local is fastest but risks reading uncommitted or rolled back data
- Choose read concern based on application consistency requirements

### Q15: How do I optimize MongoDB configuration for high throughput scenarios?

- Increase maxIncomingConnections to handle more concurrent client connections
- Configure appropriate oplogSize to capture more write operations history
- Set wiredTiger.engineConfig.evictionThreads to handle memory pressure
- Enable compression for both data and indexes to reduce memory usage
- Monitor system metrics and adjust based on actual load patterns
- Test configuration changes incrementally with production-like workloads

### Q16: What is the recommended mongod configuration for 24x7 production use?

- Set storage.journal.enabled to true for crash recovery capability
- Configure security.authorization to enforce authentication globally
- Set net.bindIpAll or specific IP addresses to control access
- Enable replication with 3 or more members for fault tolerance
- Configure appropriate log verbosity for troubleshooting without overhead
- Set ulimit values for file descriptors to 64000 or higher

### Q17: How do I configure MongoDB for high availability deployment?

- Deploy at minimum 3 nodes for replica set to survive single node failure
- Use odd number of members for automatic election without external arbiter
- Configure all members on separate physical machines in different racks
- Set secondaryIndexBuildBehavior to ensure index consistency
- Enable heartbeat monitoring with shorter intervals to detect failures faster
- Use network timeouts appropriate for your datacenter latency

### Q18: What network configurations are required for multi-datacenter MongoDB?

- Configure replica set members across multiple datacenters for disaster recovery
- Set priority values to prefer local members as primary
- Configure hidden members in secondary datacenters for offline backups
- Ensure low latency network connectivity between datacenters (below 100ms)
- Use read preference secondaryPreferred to balance read load
- Monitor replication lag and network latency continuously

### Q19: How do I tune MongoDB for mixed read and write workloads?

- Use read preference secondaryPreferred to distribute reads across replicas
- Configure journal flushing interval based on durability requirements
- Monitor write concern levels and adjust based on latency impact
- Implement connection pooling to reuse connections efficiently
- Optimize indexes for both read and write patterns if possible
- Use bulk operations instead of individual inserts for better throughput

### Q20: What is the proper way to adjust MongoDB memory settings?

- Set cache size based on working set and total available RAM
- Leave 10-20% of RAM for operating system and connections buffer
- Monitor actual memory usage patterns before making changes
- Use db.serverStatus().mem to check current memory consumption
- Increase gradually in 1-2GB increments and test with real workload
- Never allocate more cache than physical RAM to avoid system swapping

---

## 3. USER MANAGEMENT & SECURITY

### Q21: How do I create a new database user with specific permissions?

- Connect to MongoDB with admin credentials using mongosh shell
- Run use admin command to switch to admin database
- Create user with db.createUser() specifying username and password
- Assign appropriate roles parameter with desired privilege level
- Verify user creation with db.getUser() command for confirmation
- Test login using new credentials to ensure proper access

### Q22: What are the different built-in roles available in MongoDB?

- Admin roles include dbAdmin, dbAdminAnyDatabase, root, and backup
- User roles include read, readWrite, dbOwner for specific database access
- Cluster administration roles include clusterAdmin, clusterManager, clusterMonitor
- Backup roles include backup and restore for backup management operations
- Super user roles include root and rootAnyDatabase for full cluster access
- Always use principle of least privilege assigning minimal required roles

### Q23: How do I implement fine-grained access control using custom roles?

- Create base custom role with specific resources and actions list
- Run db.createRole() with role name and privilege specifications
- Define actions like find, insert, update, delete on specific collections
- Apply custom role to users that need exact those permissions
- Test custom role behavior before deploying to production
- Document role purposes and access levels for audit and compliance

### Q24: How do I enforce strong password policies for MongoDB users?

- MongoDB does not enforce password complexity by default
- Use LDAP external authentication to leverage existing password policies
- Create wrapper scripts that validate password strength before user creation
- Implement password rotation mechanisms through application layer
- Store MongoDB credentials in secure vaults like HashiCorp Vault
- Require password change on first login for new users

### Q25: What is LDAP and Kerberos authentication in MongoDB?

- LDAP allows authentication against external directory services
- Kerberos provides single sign-on with encrypted credentials
- Configure security.ldap section in mongod.conf for LDAP setup
- Requires LDAP server accessible from all MongoDB cluster members
- Kerberos requires Kerberos realm and keytab files configured properly
- Both methods reduce password management burden for large organizations

### Q26: How do I enable encryption in transit between MongoDB clients and server?

- Configure net.ssl.mode to requireSSL for mandatory TLS encryption
- Create SSL certificates and keys for MongoDB server
- Specify certificate file path in net.ssl.certFile configuration
- Clients must connect using mongodb+srv:// URI with SSL parameters
- Use mongosh with --tls and --tlsCertificateKeyFile options
- Verify encryption works with network monitoring tools like tcpdump

### Q27: How do I set up end-to-end encryption for replica set members?

- Generate SSL certificates for each replica set member
- Configure net.ssl.certFile on each mongod instance
- Enable server certificate validation to prevent man-in-the-middle attacks
- Use net.ssl.clusterFile for internal cluster member communication
- Verify certificate chain and expiration dates regularly
- Implement certificate rotation process before expiration

### Q28: What is encryption at rest and how do I configure it?

- Encryption at rest protects data stored on disk if hardware is stolen
- Requires enterprise MongoDB version with encryption storage engine
- Configure security.encryptionCipherMode for encryption algorithm
- Specify key management service for storing encryption keys
- MongoDB community version does not support encryption at rest natively
- Alternative is filesystem-level encryption using LUKS or similar

### Q29: How do I audit user activities and access logs in MongoDB?

- Enable audit feature with security.auditLog.destination configured
- Specify audit log format as JSON for easier parsing
- Set auditLog.filter to include only relevant operations
- Audit logs capture authentication, authorization, and user actions
- Store audit logs separately from main data logs for security
- Regularly review audit logs for suspicious access patterns

### Q30: How do I revoke permissions from a MongoDB user?

- Use db.revokeRolesFromUser() to remove specific roles from user
- Specify database in second parameter for scope of revocation
- Changes take effect immediately for new connections
- Existing connections retain permissions until disconnection
- Consider creating new users instead of modifying existing ones
- Verify revocation with db.getUser() to confirm roles removed

---

## 4. DATABASE & COLLECTION OPERATIONS

### Q31: What is the maximum size for a single MongoDB document?

- MongoDB document size limit is 16MB per individual document
- This includes field names, values, and nested arrays
- Plan data model to split large documents into multiple smaller ones
- Use subdocuments and arrays carefully to avoid exceeding limit
- Monitor document sizes with aggregation pipeline queries
- Reject inserts that exceed size limit with error message

### Q32: How do I create a collection with specific options?

- Use db.createCollection() with name and options parameters
- Specify capped collection with size parameter for fixed size collections
- Set maximum document count with max parameter for capped collections
- Configure collation for default sort order and comparison behavior
- Enable validation with validator for JSON schema constraints
- Collections created without explicit command use default behavior

### Q33: What is a capped collection and when should I use it?

- Capped collection has fixed maximum size and automatically removes oldest documents
- Useful for time series data, logs, and session storage
- Documents maintain insertion order which capped collections preserve
- Cannot delete individual documents in capped collection
- Cannot increase size of capped collection, must recreate it
- Provides FIFO behavior automatically without application logic

### Q34: How do I set up collection validation with JSON schema?

- Run db.runCommand() with collMod command and validator parameter
- Define validationLevel as strict for all inserts and updates
- Specify validationAction as error to reject non-conforming documents
- Write JSON schema defining required fields and data types
- Test schema validation in development before applying to production
- Use validationLevel moderate for backward compatibility

### Q35: What is the difference between insert, insertOne, and insertMany?

- insert() is legacy method that handles single or multiple documents
- insertOne() explicitly inserts single document and returns operation result
- insertMany() inserts multiple documents in batch operation
- insertMany() is faster for bulk operations with lower network overhead
- All methods support write concerns to control replication behavior
- insertMany() with ordered:false continues despite individual document failures

### Q36: How do I perform bulk write operations efficiently?

- Use bulkWrite() or initializeUnorderedBulkOp() for multiple operations
- Unordered bulk operations execute in parallel for better performance
- Ordered bulk operations execute sequentially and stop on first error
- Combine insertOne, updateOne, deleteOne, replaceOne in single batch
- Reduces network round trips compared to individual operations
- Significantly improves throughput for large data imports

### Q37: What is the maximum number of documents in a MongoDB collection?

- MongoDB does not enforce maximum document count per collection
- Practical limit depends on available storage space and performance
- Very large collections require sharding for optimal query performance
- Consider archiving old documents to manage collection growth
- Monitor collection sizes with db.collection.stats() command
- Implement data retention policies for compliance requirements

### Q38: How do I rename a collection without data loss?

- Use db.collection.renameCollection() command with new name parameter
- Operation requires exclusive write lock on collection during rename
- Cannot rename system collections or into system namespace
- Verify old collection name becomes invalid after successful rename
- Update application code to use new collection name before renaming
- Test rename procedure in non-production environment first

### Q39: How do I drop a collection and what happens to indexes?

- Use db.collection.drop() command to remove collection and all documents
- All indexes associated with collection are automatically dropped
- Operation requires appropriate permissions on target collection
- Verify collection is no longer accessible after dropping
- Consider creating backup before dropping large collections
- Dropping collection is faster than deleting all documents

### Q40: What is the proper procedure for moving documents between collections?

- Use aggregation pipeline with $out stage to move data efficiently
- Alternatively use find().forEach() with batch inserts to new collection
- Verify all documents transferred before deleting source collection
- Compare document counts and checksums to ensure data integrity
- Recreate indexes on target collection for performance
- Monitor performance to detect issues during large data movements

---

## 5. REPLICATION & HIGH AVAILABILITY

### Q41: What are the different member types in a MongoDB replica set?

- Primary member handles all write operations and replicates to secondaries
- Secondary members receive and replicate data from primary
- Arbiter participates in elections but stores no data
- Hidden members cannot become primary and hidden from client drivers
- Delayed members replicate with configurable delay for accidental deletion recovery
- Priority 0 members never become primary regardless of availability

### Q42: How do I configure a replica set member to never become primary?

- Set member's priority value to 0 in replica set configuration
- Member participates in elections but cannot win primary position
- Useful for secondary datacenters or disaster recovery members
- Configure with priority 0 to force primary to stay in main datacenter
- Hidden members with priority 0 serve backup purposes only
- Use rs.conf() to verify priority settings for all members

### Q43: What is the oplog and why is it important for replication?

- Oplog is special capped collection named local.oplog.rs on each member
- Contains all write operations applied to primary in chronological order
- Secondary members read oplog to replicate changes to their datasets
- Defines maximum point-in-time recovery window for backup purposes
- Oplog size defaults to 5% of total disk space for replica sets
- Manual oplogSize configuration provides custom retention period

### Q44: How do I increase oplog size without rebuilding replica set?

- Oplog size can be modified for running replica sets using replSetResizeOplog
- Use db.adminCommand() with replSetResizeOplog command and size parameter
- Specify size in MB to allocate new oplog size
- Operation requires maintenance mode if downtime is acceptable
- Larger oplog provides longer point-in-time recovery capability
- Monitor oplog size growth to plan capacity requirements

### Q45: What causes replication lag and how do I troubleshoot it?

- Network latency between primary and secondary members causes delays
- Slow secondary disk I/O prevents writing replicated data quickly
- Insufficient secondary resources cause backlog of operations
- Long running queries on secondary block replication
- Use rs.status() to check members array for replicationLag field
- Monitor both network and disk I/O on secondary members

### Q46: How do I reduce replication lag in high throughput scenarios?

- Ensure secondary members have sufficient resources as primary
- Monitor disk I/O and increase throughput if bottleneck is storage
- Check network bandwidth and reduce network latency if possible
- Disable secondary indexes during high write periods if acceptable
- Increase oplog size to capture more operations for recovery
- Use multiple secondaries to distribute read load away from primary

### Q47: How do I perform safe maintenance on replica set members?

- Connect to secondary member and run rs.secondaryOk() or readPreference
- Issue maintenance mode commands without affecting primary
- For primary maintenance, perform stepDown to force election first
- Secondary stops replication during maintenance window
- Restart mongod service and verify member rejoins replica set
- Monitor replication lag until member catches up to primary

### Q48: What is replica set election and how does it work?

- Election occurs when primary becomes unavailable or issues rs.stepDown()
- Members vote for replacement based on priority and data recency
- Requires majority of members voting for single candidate to succeed
- Member with highest priority and latest oplog usually wins election
- Election timeout causes new election attempt if no majority reached
- Minimize election failures with odd number of members

### Q49: How do I handle replica set with even number of members?

- Add arbiter to break ties and provide majority with even member count
- Arbiter does not store data but participates in elections
- Three members with one arbiter is preferred configuration
- Avoid two member replica sets entirely without arbiter
- Consider full replica set member instead if storage available
- Use priority 0 members instead of arbiter when possible

### Q50: How do I configure read preference for replica sets?

- Primary reads all operations from primary member (default)
- PrimaryPreferred reads from primary, falls back to secondary
- Secondary reads from any available secondary member
- SecondaryPreferred reads from secondary, falls back to primary
- Nearest reads from member with lowest network latency
- Choose based on consistency requirements and load balancing needs

### Q51: What is the difference between heartbeat and helloOk in replica sets?

- Heartbeat is periodic status check between replica set members
- HelloOk field replaced ismaster field in newer MongoDB versions
- Members exchange heartbeat every 2 seconds by default
- Heartbeat timeout after 10 seconds triggers new election
- Configure heartbeatTimeoutSecs to adjust timeout duration
- Faster heartbeats reduce failover time but increase overhead

### Q52: How do I remove a member from existing replica set?

- Connect to current primary member using mongosh
- Run rs.remove() with member hostname and port specification
- Member gracefully steps down if currently primary
- Mongodb automatically finds replacement primary from remaining members
- Verify removal with rs.conf() to confirm member not in configuration
- Stop mongod service on removed member if desired

### Q53: How do I add new member to existing replica set without downtime?

- Start new MongoDB instance on separate server
- Connect to primary and run rs.add() with new member specification
- New member performs initial sync to copy all data from primary
- Monitor initial sync progress with db.serverStatus() replication
- Configure secondary with priority 0 during initial sync to prevent election
- Restore priority after initial sync completes successfully

### Q54: What is initial sync and how long does it take?

- Initial sync copies all data from primary to new secondary member
- Occurs automatically when new member joins replica set
- Duration depends on database size and network bandwidth
- Can take hours for very large databases
- Monitor with db.serverStatus().replication.initialSyncStatus
- Disable secondary to prevent accidental writes during initial sync

### Q55: How do I handle split brain scenario in replica set?

- Split brain occurs when network partition divides replica set
- Members on minority side step down to prevent conflicting writes
- Requires majority to elect primary and continue operations
- Minority members become secondaries with no primary elected
- Write concern majority prevents data loss during split brain
- After partition heals, no data loss occurs if majority write concern used

---

## 6. SHARDING & DISTRIBUTED ARCHITECTURE

### Q56: What is sharding and when should I implement it?

- Sharding distributes data across multiple servers horizontally
- Implement when single server cannot handle data volume or throughput
- Each shard stores subset of data based on shard key ranges
- Query router (mongos) directs operations to appropriate shards
- Requires sharding when working set exceeds single server capacity
- Add shards as database grows to continue scaling performance

### Q57: What makes a good shard key selection?

- Shard key should have high cardinality to distribute data evenly
- Avoid monotonically increasing values that cause uneven chunk distribution
- Ensure shard key exists in every document of sharded collection
- Query pattern should ideally filter on shard key for efficiency
- Write patterns should distribute across multiple shards equally
- Consider future growth and query patterns before choosing shard key

### Q58: What is chunk migration and why does it occur?

- Chunks are ranges of documents grouped by shard key
- Balancer automatically migrates chunks to redistribute data evenly
- Occurs when shard size difference exceeds threshold (default 8MB)
- Migration can impact query performance during large data movements
- Disable balancer during maintenance windows to prevent interference
- Monitor chunk migrations with sh.status() command

### Q59: How do I set up a sharded cluster from scratch?

- Deploy 3 or more config servers as replica set for metadata storage
- Start mongos instances pointing to config server replica set
- Create regular replica sets for each shard
- Use sh.addShard() to add each replica set as shard
- Use sh.shardCollection() to enable sharding on target collection
- Specify shard key during shardCollection command

### Q60: What are config servers and what role do they play?

- Config servers store metadata about cluster and shard keys
- Maintain authoritative source of sharded cluster state
- Deployed as replica set with minimum 3 members for production
- Cannot perform sharding without functioning config servers
- All mongos instances connect to same config server replica set
- Config server data typically smaller than shard data

### Q61: How do I reshard a collection with different shard key?

- MongoDB 5.0+ allows resharding without downtime
- Use sh.reshardCollection() command with new shard key
- Resharding performs chunk migrations to new distribution
- Application continues working during resharding operation
- Monitor progress with sh.reshardingStatus() command
- Validation confirms data integrity after resharding completes

### Q62: What is a mongos and what does it do?

- Mongos is query router that directs operations to appropriate shards
- Clients connect to mongos instead of directly to shard members
- Multiple mongos instances provide high availability and load balancing
- Mongos cache metadata but refresh periodically from config servers
- Install mongos on application servers for low latency routing
- Monitor mongos with standard MongoDB monitoring tools

### Q63: How do I handle uneven data distribution across shards?

- Use sh.balancer.status() to check if balancer is running
- Verify chunk count distribution with sh.status() output
- Issue was likely caused by poor shard key selection
- Resharding to new shard key is solution for severe imbalance
- Alternatively, archive old data to reduce imbalance
- Monitor chunk distribution continuously for early detection

### Q64: What is ranged sharding and how does it work?

- Ranged sharding groups documents into chunks by shard key value ranges
- Example: shard key values 1-1000 on shard1, 1001-2000 on shard2
- Works well for time series data when time range increases monotonically
- Can lead to uneven distribution if value ranges change over time
- Allows efficient range queries on shard key
- Risk of hotspot shard if writes concentrate in current range

### Q65: What is hashed sharding and when should I use it?

- Hashed sharding distributes data using hash function of shard key
- Provides more even distribution than ranged sharding
- Example: hash(user_id) determines which shard stores document
- Range queries on shard key become scattered across multiple shards
- Better for high cardinality fields with random write distribution
- Combine with compound shard key for optimal results

### Q66: How do I disable or enable sharding balancer?

- Use sh.stopBalancer() to pause chunk migrations
- Sh.startBalancer() resumes automatic balancing operations
- Disable balancer during maintenance, backups, or large operations
- Re-enable after maintenance completes
- Check balancer status with sh.isBalancerRunning()
- Schedule manual balancing during maintenance windows

### Q67: What is the impact of sharding on query performance?

- Scatter-gather queries hitting all shards are slower than single shard
- Shard key filtering queries go to single shard for optimal performance
- Aggregation pipelines execute on each shard then merge results
- Sort operations on non-shard key fields require additional merge step
- Join queries become complex and inefficient in sharded environment
- Design queries to use shard key filters whenever possible

### Q68: How do I migrate data between sharded and unsharded cluster?

- Unsharding requires dump and restore to unsharded cluster
- Use mongodump from sharded cluster and mongorestore to single node
- Ensure mongorestore has sufficient disk space for full database
- Drop existing database before restore to avoid conflicts
- Test restore in development environment first
- Verify data integrity after restore completes

### Q69: What are shard tags and how do I use them?

- Shard tags allow chunk assignment to specific shards
- Implement zone-based sharding by geographic region
- Add tag to shard with sh.addShardTag() command
- Define tag range with sh.addTagRange() using shard key values
- Balancer automatically routes tagged ranges to tagged shards
- Useful for data residency requirements and disaster recovery

### Q70: How do I add new shard to existing sharded cluster?

- Start new MongoDB replica set to serve as new shard
- Use sh.addShard() with connection string for new shard
- Balancer automatically redistributes chunks to new shard
- Verify new shard appears in sh.status() output
- Monitor balancing operation with sh.balancer.status()
- Consider disabling balancer during business hours

### Q71: What happens when a shard becomes unavailable?

- Queries targeting unavailable shard fail with error
- Queries on available shards succeed if not requiring unavailable shard
- Chunk balancing pauses until shard returns online
- Replication continues on available shards if shard is replica set
- Recover shard by fixing underlying issue and restarting mongod
- Use shard recovery procedures based on failure type

### Q72: How do I monitor sharded cluster health?

- Use sh.status() to view overall cluster configuration
- Monitor individual shard health with rs.status() on each shard
- Check mongos connectivity and version consistency
- Monitor chunk distribution with db.chunks.find().count()
- Watch for replication lag on shards with db.serverStatus()
- Set up alerting for shard unavailability and balancer failures

---

## 7. BACKUP & RESTORE OPERATIONS

### Q73: What are the different backup strategies for MongoDB?

- Filesystem snapshots provide fast backups for large databases
- mongodump creates logical backups in BSON format
- Continuous oplog backup enables point-in-time recovery
- Cloud provider snapshots work for cloud-hosted deployments
- MongoDB Atlas provides automated managed backups
- Choose based on RPO requirements and recovery time SLA

### Q74: How do I create backup using mongodump tool?

- Run mongodump without parameters to backup entire database
- Specify --db parameter to backup single database
- Use --collection to backup specific collection
- Specify --out parameter for output directory
- Run --gzip to compress backup files reducing storage
- Backup size typically smaller than original data due to no indexes

### Q75: What is the proper procedure to restore data using mongorestore?

- Run mongorestore pointing to backup directory created by mongodump
- Specify target host with --uri parameter if not localhost
- Use --db to restore to specific database name
- Drop existing data with --drop to replace old data
- Run --convertLegacySystemIndexes for backwards compatibility
- Restore indexes after data load completes for faster restore

### Q76: How do I backup only specific collections using mongodump?

- Use --collection parameter specifying collection name to backup
- Combine with --db for single database
- Run multiple mongodump commands for multiple collections
- Consider backup time for large number of collections
- Store separate collection backups for selective restore capability
- Document which collections are backed up together

### Q77: What is the difference between full backup and incremental backup?

- Full backup copies entire database on first backup run
- Incremental backup copies only changed data since last backup
- MongoDB does not natively support incremental backups with mongodump
- Filesystem snapshots can achieve incremental behavior with LVM/ZFS
- Continuous oplog backup provides effective incremental protection
- Calculate storage requirements based on change rate, not full size

### Q78: How do I implement point-in-time recovery for MongoDB?

- Create base backup using filesystem snapshot or mongodump
- Continuously backup oplog to capture all write operations
- Replay oplog entries up to desired timestamp for recovery
- Requires oplog size large enough to retain required time window
- Practice point-in-time recovery regularly in test environment
- Document recovery procedure including timestamp identification

### Q79: How do I backup replica set without interrupting service?

- Connect to secondary member and perform backup operations
- Primary continues accepting writes and replicating unchanged
- Use hidden secondary specifically for backup if available
- Configure secondary with priority 0 and hidden flag
- Perform full or incremental backups on secondary without impact
- Regular backups from secondary maintain recovery capability

### Q80: What is backup validation and how do I verify backup integrity?

- Perform test restore to separate environment regularly
- Verify document counts match between source and restored database
- Run data validation queries on restored database
- Compare checksums of critical collections to source
- Automate restore testing monthly to catch backup issues early
- Document validation results for audit compliance

### Q81: How do I securely backup MongoDB with sensitive data?

- Encrypt backup files at rest using file system encryption
- Use TLS encryption for backup transmission across network
- Restrict backup file access with file permissions
- Consider database-level encryption for sensitive data
- Store backups in secured location separate from production
- Implement backup rotation policy to limit exposure window

### Q82: How do I restore specific database from full MongoDB backup?

- Run mongorestore pointing to backup directory
- Use --nsInclude parameter to specify database.collection pattern
- Filter specific collections with wildcard patterns
- Verify restored collections exist and contain expected documents
- Compare restore time and success rate with requirements
- Consider restore parallelization for faster recovery

### Q83: What backup strategy should I use for very large MongoDB databases?

- Filesystem snapshots are fastest for large databases
- LVM or ZFS snapshots provide atomic consistent backups
- Incremental snapshots significantly reduce backup storage
- Configure snapshot retention policy to meet RPO requirements
- Test snapshot restore procedures regularly
- Schedule snapshots during low activity periods

### Q84: How do I backup MongoDB Atlas cluster?

- Atlas provides continuous automated backups by default
- Configure backup frequency from hourly to yearly as needed
- Set snapshot retention periods from 7 days to 35 years
- Restore snapshots to new cluster in same or different organization
- Export data to AWS S3 or Azure storage for archival
- Test restore procedures from Atlas snapshots regularly

### Q85: How long should I retain MongoDB backups?

- Minimum retention should match RPO requirements
- Consider regulatory compliance requirements for data retention
- Retain backups for at least 30 days for typical scenarios
- Archive older backups to cheaper storage after retention period
- Document backup retention policy for audit purposes
- Implement automated retention policy enforcement

### Q86: What is the proper backup procedure for sharded MongoDB cluster?

- Backup each shard separately as independent replica set
- Include config server data in backup for metadata
- Coordinate timing to capture consistent cluster state
- Backup mongos configuration for cluster recovery
- Document shard key information for cluster recreation
- Test shard backup and restore procedures separately

### Q87: How do I backup MongoDB to cloud storage like AWS S3?

- Use mongodump and pipe output to AWS S3 upload
- Configure AWS credentials and target S3 bucket
- Compress backup with gzip before uploading to reduce costs
- Implement multipart upload for large files
- Set S3 server-side encryption for data protection
- Verify upload completion and backup integrity

### Q88: How do I create automated daily backups for MongoDB?

- Write shell script calling mongodump with timestamp
- Schedule script in crontab for daily execution
- Specify backup directory with date suffix for easy archival
- Configure email notification on backup success or failure
- Implement monitoring to alert on missed backups
- Test recovery from automated backups monthly

### Q89: What are backup SLAs and how do I calculate them?

- RPO is maximum acceptable data loss between backups
- RTO is maximum acceptable time to restore from backup
- Backup frequency determines RPO capability
- Restore time depends on database size and backup method
- Document SLA requirements before selecting backup strategy
- Monitor actual SLAs against commitments regularly

### Q90: How do I handle backup failures and retries?

- Implement monitoring to detect backup job failures
- Configure automatic retry logic for failed backups
- Log backup execution details for troubleshooting
- Alert on repeated failures requiring manual investigation
- Verify backup integrity before considering recovery ready
- Document failure recovery procedures for operations team

---

## 8. MONITORING & DIAGNOSTICS

### Q91: What monitoring metrics are most critical for MongoDB?

- CPU usage indicates if mongod is CPU-bound workload
- Memory usage and cache hit ratio show working set efficiency
- Disk I/O operations per second determine storage throughput
- Replication lag on secondary members indicates sync issues
- Connection count shows concurrent client load
- Query performance percentiles identify slow operations

### Q92: How do I use mongostat for monitoring MongoDB?

- Run mongostat without parameters for continuous real-time statistics
- Specify --host for remote server monitoring
- Output includes operations per second, memory usage, and lock percentage
- Timestamps show when each measurement was taken
- Use --json output for scripted monitoring integrations
- Monitor for sustained high values indicating bottlenecks

### Q93: What is the profiler and how do I enable it?

- Database profiler logs slow query execution details
- Enable with db.setProfilingLevel() specifying level 0-2
- Level 1 logs queries exceeding slowms threshold (default 100ms)
- Level 2 logs all queries regardless of execution time
- Store profiling data in system.profile collection
- Review profiling data with find queries and aggregation

### Q94: How do I query the profiler output to find slow queries?

- Query system.profile collection with sorting by millis field descending
- Filter by namespace for specific database.collection
- Sort by exec stats keys like keysExamined and docsExamined
- Identify queries with high ratio of keys examined to documents returned
- Look for queries without appropriate index filter
- Correlate slow queries with query patterns in application

### Q95: What is serverStatus command and what does it show?

- db.serverStatus() returns comprehensive server statistics
- Shows memory usage, storage engine info, and replication status
- Displays operation counters for read, write, update, delete
- Returns timing information for slow operations
- Connection info shows current and available connections
- Monitor returned values to track performance over time

### Q96: How do I identify queries that are not using indexes?

- Use explain() with executionStats to see query execution plan
- Look for COLLSCAN stage indicating full collection scan
- Compare docsExamined to docsReturned ratio
- High ratio indicates inefficient query or missing index
- Use getIndexes() to verify expected indexes exist
- Create missing indexes on frequently queried fields

### Q97: How do I monitor replication status in replica sets?

- Use rs.status() to view current replica set state
- Check optime field shows how up-to-date each member is
- Monitor optime diffs for replication lag between members
- Verify all members in healthy state (1=primary, 2=secondary)
- Check network conditions if lag is high
- Alert on members with stale optime or unhealthy state

### Q98: What is the difference between optime and operational statistics?

- Optime is monotonically increasing timestamp of last applied operation
- Includes term and timestamp to track operation ordering
- Operational statistics show operations per second and types
- Optime used for consistency verification and replication ordering
- Operational stats used for performance trending and alerting
- Monitor both for complete operational visibility

### Q99: How do I set up alerting for MongoDB replication lag?

- Calculate replication lag from optime differences
- Alert when lag exceeds threshold (e.g., 60 seconds)
- Monitor secondary application time versus primary time
- Check network connectivity between primary and secondary
- Alert also on primary unavailability causing election
- Document alert response procedures

### Q100: How do I monitor disk usage and plan capacity?

- Use du command to check current database directory size
- Monitor growth rate to forecast when disk space exhausts
- Plan capacity based on retention policies and growth projections
- Alert when disk usage exceeds thresholds (e.g., 80%, 90%)
- Leave 10-15% free space for temporary operations
- Monitor both mongod data files and backup storage

---

## 9. INDEXING & QUERY OPTIMIZATION

### Q101: What types of indexes does MongoDB support?

- Single field indexes on individual fields
- Compound indexes on multiple fields in specified order
- Text indexes for text search across string fields
- Geospatial indexes for location-based queries
- Sparse indexes excluding documents without indexed field
- TTL indexes automatically removing documents after time period

### Q102: How do I create index on single field?

- Use db.collection.createIndex() with field specification
- Specify 1 for ascending order or -1 for descending
- Index creation blocks writes on collection by default
- Use background:true option for background index creation
- Background index creation takes longer but allows writes
- Specify unique:true for unique constraint on field

### Q103: What is compound index and when should I use it?

- Compound index indexes multiple fields together
- Order of fields matters for query efficiency
- First field used for filtering then subsequent fields
- Useful for queries filtering on multiple fields with common patterns
- Efficient for queries with sort on indexed fields
- Design compound indexes based on actual query patterns

### Q104: How do I create compound index with specific order?

- Run db.collection.createIndex({field1: 1, field2: -1, field3: 1})
- First field sorted ascending, second descending, third ascending
- Query must filter on first field for index to be used efficiently
- Can skip middle fields but must filter on first field
- Sort order specified in index affects query efficiency
- Test explain() output to verify index is used

### Q105: What is the impact of index selectivity?

- Selectivity is percentage of unique values in indexed field
- High selectivity means index can narrow results significantly
- Low selectivity index with many duplicates is less beneficial
- Query on low selectivity field may still full scan if better
- Consider selectivity when choosing which fields to index
- Combine low selectivity fields with high selectivity fields

### Q106: How do I remove unused or redundant indexes?

- Use db.collection.getIndexes() to list all indexes
- Query system.indexes to find index usage statistics
- Drop indexes with db.collection.dropIndex() specifying index name
- Removing redundant indexes improves write performance
- Verify index not used before dropping with explain()
- Document index removal reason for audit trail

### Q107: What is the cost of indexes on write performance?

- Each index requires additional write for insert, update, delete
- Multiple indexes multiply write overhead linearly
- Trade-off between read and write performance
- Monitor write throughput after index creation
- Remove indexes that improve reads but hurt write performance
- Balance number of indexes with write workload requirements

### Q108: How do I optimize slow aggregation pipeline queries?

- Place match stage first to filter early and reduce documents
- Use project to reduce fields processed by later stages
- Avoid sorting large result sets unless necessary
- Use allowDiskUse:true for large aggregations
- Monitor aggregation execution with explain()
- Create indexes on fields used in match stage

### Q109: What is the explain() output and how do I interpret it?

- executionStages shows query execution plan details
- COLLSCAN indicates full collection scan (inefficient)
- IXSCAN indicates index scan (efficient)
- docsExamined vs docsReturned ratio indicates efficiency
- executionTimeMillis shows query execution duration
- Study execution plan to identify optimization opportunities

### Q110: How do I force MongoDB to use specific index?

- Use hint() method in query to specify index name
- Example: db.collection.find({field:value}).hint({field:1})
- Useful for testing different indexes for same query
- Verify performance improvement before using in production
- Remove hint when index selection works optimally
- Document why hint was necessary if keeping in code

### Q111: What is index intersection and how does it work?

- Index intersection allows using multiple indexes for single query
- MongoDB selects most selective index then filters with others
- Useful when single compound index not available
- Multiple index scans may be less efficient than one larger index
- Monitor explain() output to identify index intersection scenarios
- Consider creating compound index for better performance

### Q112: How do I handle queries with OR conditions efficiently?

- OR queries may scan multiple indexes reducing efficiency
- Create compound index on OR fields if possible
- Example: query finding field1=value OR field2=value
- Consider denormalizing data to avoid OR queries
- Use regex or text search for complex OR patterns
- Monitor OR query performance in explain() output

### Q113: What are sparse indexes and when should I use them?

- Sparse index excludes documents without indexed field
- Saves index space when many documents missing field
- Reduces index size improving memory efficiency
- Query filters on sparse field may not use index
- Unique sparse index allows multiple null values
- Useful for optional fields in flexible schema

### Q114: How do I create TTL index for automatic document expiration?

- Run db.collection.createIndex({expiresAt: 1}, {expireAfterSeconds: 0})
- Field contains ISODate of when document should expire
- MongoDB background thread removes expired documents periodically
- Expiration check runs every 60 seconds by default
- Useful for sessions, cache, and temporary data
- Verify documents actually expire with find queries

### Q115: What is text index and how do I use it?

- Text index enables text search across string fields
- Create with db.collection.createIndex({field: "text"})
- Query using $text operator with search string
- Supports phrase search with quotes in search string
- Case-insensitive and diacritic-insensitive searches
- Weight fields with different text index weights

### Q116: How do I use explain() to identify missing indexes?

- Run db.collection.find(query).explain("executionStats")
- Look for COLLSCAN stage indicating full collection scan
- High docsExamined to docsReturned ratio indicates inefficiency
- Create index on fields in query filter or sort
- Re-run explain() to verify index is used
- Monitor index effectiveness over time as query patterns change

### Q117: What is cardinality and how does it affect index design?

- Cardinality is number of distinct values in indexed field
- High cardinality fields (many unique values) benefit from indexes
- Low cardinality fields (few unique values) have limited index benefit
- Combine high cardinality field first in compound indexes
- Low cardinality field alone may not benefit from indexing
- Analyze field distribution before deciding to index

### Q118: How do I handle index on array fields?

- Multikey indexes automatically created for array fields
- Index contains entry for each array element
- Affects index storage size for arrays with many elements
- Single array element query can use multikey index efficiently
- Multiple array fields in compound index have restrictions
- Monitor multikey index overhead versus query benefit

### Q119: What is partial index and how do I create it?

- Partial index includes only documents matching filter expression
- Reduces index size for conditional indexing scenarios
- Create with db.collection.createIndex({field:1},{partialFilterExpression:expr})
- Example: index only active documents in status field
- Save index space and memory for relevant documents
- Query filter must match partial index filter to use it

### Q120: How do I monitor index usage and statistics?

- Use db.collection.aggregate([{$indexStats:{}}]) for statistics
- View accesses, ops, host fields for index usage metrics
- Identify unused indexes by zero accesses value
- Monitor index size growth over time
- Track index operations per second for trending
- Remove unused indexes after confirmation

---

## 10. DISASTER RECOVERY SCENARIOS

### Q121: How do I recover from accidental database deletion?

- Stop application immediately to prevent writes to backup
- Identify timestamp of deletion from application logs
- Restore from most recent backup before deletion
- Use point-in-time recovery if oplog still contains deletion
- Verify restored data completeness and consistency
- Implement safeguards to prevent future accidental deletion

### Q122: What is the recovery procedure for data corruption in MongoDB?

- Stop application writes to prevent corruption spread
- Identify corrupted collections or databases
- Restore from known good backup before corruption
- Use point-in-time recovery if available and oplog not corrupted
- Verify data integrity after restore with consistency checks
- Investigate corruption cause (hardware, bug, or human error)

### Q123: How do I handle primary node complete failure in replica set?

- Replica set automatically elects new primary from secondaries
- No manual action needed for automatic failover
- Verify new primary elected within election timeout
- Check replication status shows healthy configuration
- Recover failed node separately when stable
- Implement alerting to notify on primary node failure

### Q124: What is the recovery procedure when majority of replica set members fail?

- Remaining minority members cannot elect new primary
- Writes are blocked until majority members recover
- Recover failed members as quickly as possible
- Can force reconfiguration if primary recovers with rs.reconfig()
- Consider forcing single member mode temporarily for recovery
- Implement hardware redundancy to prevent simultaneous failures

### Q125: How do I recover from network partition in replica set?

- Network partition causes replica set to split
- Minority members step down and cannot accept writes
- Majority members continue operation normally
- Write concern majority protects from data loss
- Repair network partition to heal split
- No data loss occurs if write concern majority enforced

### Q126: What is the procedure for recovering backup from test failure?

- Stop application to prevent writes conflicting with restore
- Drop corrupted database or collection
- Restore clean backup to same or different MongoDB instance
- Verify restored data completeness with sample queries
- Compare document counts with pre-failure measurements
- Resume application to point at restored database

### Q127: How do I do point-in-time recovery from oplog?

- Identify exact timestamp for recovery target
- Restore database from full backup before target timestamp
- Replay oplog entries from backup timestamp to target timestamp
- Use oplog entries from continuous oplog backup or replica set oplog
- Verify recovery consistency by checking application transactions
- Document timestamp and reason for recovery in audit log

### Q128: What is crash recovery and how is it automatic?

- MongoDB with journaling automatically recovers from crash
- Journal entries replayed on startup to restore consistency
- No data loss occurs if journaling enabled (WiredTiger default)
- Recovery time depends on journal size and entries count
- Verify mongod startup completes successfully
- Check replica set status shows recovered member healthy

### Q129: How do I recover MongoDB instance running out of disk space?

- Stop writing to MongoDB to prevent operational errors
- Add additional disk space if possible
- Delete unnecessary data or collections to free space
- Verify disk space after deletion
- Restart mongod if it crashed due to disk full
- Monitor disk space and alert before reaching capacity

### Q130: What is the recovery procedure for configsvr failure in sharded cluster?

- Configsvr stores metadata for entire sharded cluster
- Configsvr replica set failure prevents cluster operations
- Shards continue to operate but not accessible via mongos
- Recover configsvr replica set immediately as priority
- Restore configsvr from backup if all members fail
- Verify cluster metadata consistency after recovery

### Q131: How do I handle shard unavailability in sharded cluster?

- Queries for unavailable shard data fail
- Queries for available shard data succeed
- Secondary members continue to function if shard is replica set
- Identify root cause of shard unavailability
- Recover shard by fixing underlying issue
- Monitor balancing operations after shard recovery

### Q132: What is the procedure for recovering MongoDB Atlas cluster?

- Use automated backups to restore to point-in-time
- Restore to new cluster in same organization
- Test restored cluster before switching application traffic
- Atlas provides backup history for 35 days or longer
- Automatic backups continue during restore operation
- Configure PITR window for required recovery point

### Q133: How do I prevent data loss during planned maintenance?

- Enable write concern majority for data loss prevention
- Use replica sets with 3 or more members
- Perform maintenance on secondary members first
- Stepdown primary before maintenance starts
- Verify new primary elected during stepdown
- Test failover procedures before maintenance window

### Q134: What is backup validation testing and how often to perform?

- Monthly restore testing verifies backups are recoverable
- Restore to separate environment from backups
- Verify data integrity with consistency checks
- Test application against restored data
- Document restore success and time taken
- Refine recovery procedures based on test results

### Q135: How do I prepare for geographic disaster recovery?

- Replicate data to multiple geographic regions
- Configure secondary datacenter as hidden members
- Store backups in different region from primary
- Test failover to secondary datacenter regularly
- Configure DNS to redirect traffic after failover
- Document RTO and RPO for disaster recovery

### Q136: What is the recovery procedure for corrupted oplog?

- Oplog corruption causes replication to fail
- Perform initial sync on affected member if corruption detected
- Larger oplog replacement necessary if oplog too corrupted
- Primary will not replicate to member with corrupted oplog
- Delete and rebuild oplog on affected member
- Monitor member for replication lag after recovery

### Q137: How do I restore from backup with data encryption enabled?

- Encryption key needed to decrypt backup data during restore
- Restore to same encryption context for seamless recovery
- Different encryption key prevents restore
- Store encryption keys separately from backups
- Test restore with encryption in staging environment
- Document encryption key storage and rotation procedures

### Q138: What is the recovery procedure for full cluster loss?

- Restore all shards from backups simultaneously
- Verify configsvr recovery and consistency
- Restore each shard replica set separately
- Recreate mongos instances and connect to recovered configsvr
- Verify cluster metadata consistency across shards
- Monitor for chunk balancing and recovery completion

### Q139: How do I handle backup integrity issues discovered during restore?

- Stop restore operation immediately
- Identify specific data corruption in backup
- Investigate backup creation process for issues
- Use alternate backup if available for same timeframe
- Perform point-in-time recovery if oplog available
- Implement additional backup validation checks

### Q140: What is the procedure for testing disaster recovery plans?

- Schedule quarterly disaster recovery drills
- Simulate various failure scenarios
- Practice restore procedures from backups
- Measure actual RTO and compare to SLA
- Identify gaps in procedures and communication
- Update runbooks based on test results and lessons learned

---

## 11. TROUBLESHOOTING & OPERATIONS

### Q141: How do I diagnose high CPU usage in MongoDB?

- Check current running queries with db.currentOp()
- Identify queries with high millis indicating long runtime
- Review query execution plans with explain()
- Look for COLLSCAN or inefficient execution stages
- Create indexes on frequently queried fields
- Monitor CPU usage over time to identify patterns

### Q142: What causes high memory usage in MongoDB?

- Large working set exceeding available RAM
- Inefficient queries processing large result sets
- Aggregation pipelines without allowDiskUse enabled
- Too many concurrent connections
- Indexes consuming excessive memory
- Monitor memory usage with mongostat and serverStatus

### Q143: How do I reduce memory consumption in MongoDB?

- Increase RAM capacity if working set requires it
- Optimize queries to reduce result set size
- Create indexes on frequently queried fields
- Drop unused indexes reducing memory footprint
- Reduce connection pool size if excessive
- Archive old data to reduce active working set

### Q144: What is the cause of lock percentage remaining high?

- High lock percentage indicates contentious operations
- Write-heavy workloads cause higher lock percentages
- Multiple operations competing for database lock
- Optimize transactions to complete faster
- Reduce write operations or batch them
- Monitor lock statistics with db.serverStatus()

### Q145: How do I identify slow network between MongoDB nodes?

- Run ping command between replica set members
- Look for high latency (>10ms) indicating network issues
- Use iperf to measure actual network throughput
- Check network congestion on switches and routers
- Monitor replication lag for network indicators
- Address network issues before replication problems occur

### Q146: What causes connection failures to MongoDB?

- MongoDB process not running on target host
- Firewall blocking TCP port 27017 (default) or configured port
- Authentication credentials incorrect or user missing
- Host unreachable or network connectivity issue
- DNS resolution failing to resolve hostname
- Verify port open with telnet or nc command

### Q147: How do I handle too many connections error?

- Default maxConnections limit reached on server
- Increase maxConnections in configuration up to 1 million
- Reduce client connection pool size to limit total connections
- Implement connection pooling on application tier
- Monitor connections with db.serverStatus().connections
- Alert when connection count approaches limit

### Q148: What is connection pool and how do I configure it?

- Connection pool maintains persistent connections to MongoDB
- Application reuses pooled connections reducing overhead
- Default pool size varies by driver (typically 10-50)
- Configure maxPoolSize in connection string
- Larger pools use more memory but reduce connection churn
- Monitor pool efficiency with connection statistics

### Q149: How do I diagnose and fix authentication issues?

- Verify credentials are correct in connection string
- Check user exists in target database
- Confirm user has roles required for operation
- Enable authenticationMechanisms if not default
- Verify authentication database with --authenticationDatabase parameter
- Check server logs for authentication error details

### Q150: What is the procedure for rotating credentials in MongoDB?

- Create new user with same permissions as old user
- Update application connection strings to new credentials
- Verify all applications updated before removing old user
- Remove old user after confirming no connections active
- Monitor for any connection failures during transition
- Document credential rotation for audit compliance

### Q151: How do I handle write concern timeout errors?

- Write concern timeout occurs when write cannot replicate in time
- Increase wtimeout value in write concern settings
- Check replication lag and network latency
- Verify secondary members are healthy and replicating
- Identify and fix the underlying replication lag cause
- Consider using write concern majority instead of all

### Q152: What causes oplog overflow during high throughput?

- Write volume exceeds oplog retention time
- Secondaries cannot keep up with replication
- Increase oplog size with replSetResizeOplog command
- Reduce write throughput if possible
- Monitor oplog with db.oplog.rs.find().sort({ts:-1}).limit(1)
- Alert when oplog retention time approaches configured size

### Q153: How do I handle journal persistence errors?

- Journal write failures cause replication to stall
- Verify disk space available for journal writes
- Check disk performance and I/O errors
- Verify journal location is on fast storage device
- Restart mongod after fixing underlying disk issue
- Review disk I/O monitoring for saturation

### Q154: What is the recovery procedure for lost write operations?

- Write concern determines data loss risk
- Write concern majority prevents data loss on node failure
- Write concern unacknowledged risks silent data loss
- Always use write concern majority in production
- Point-in-time recovery recovers lost ops from oplog
- Implement automated data validation to detect loss

### Q155: How do I diagnose and fix query timeout issues?

- Query timeout error occurs when query exceeds configured timeout
- Increase maxTimeMS parameter in query if safe
- Optimize query to reduce execution time
- Create indexes to make query more efficient
- Monitor query execution time with explain()
- Consider running heavy queries during off-peak hours

### Q156: What causes stale reads from secondary members?

- Secondary replication lag causes older data read
- Use readPreference primary to always get latest data
- Monitor replication lag with rs.status()
- Verify secondary member has sufficient resources
- Check network connectivity between primary and secondary
- Consider using readPreference primaryPreferred for resilience

### Q157: How do I handle index build failures?

- Index build can fail due to disk space or memory issues
- Restart index build after fixing underlying resource issue
- Drop failed index and rebuild from scratch
- Monitor index build progress with db.currentOp()
- Use background index builds to avoid write blocking
- Allocate sufficient resources before building large indexes

### Q158: What is the procedure for cleaning up temporary collections?

- Temporary collections created by aggregation or sort operations
- MongoDB automatically cleans up if space reclaimed
- Manual cleanup using db.collection.drop()
- Check system temp space if cleanup delayed
- Monitor temp space usage during heavy aggregations
- Configure allowDiskUse to use disk for large operations

### Q159: How do I diagnose replication failure on secondary?

- Check replication status with rs.status()
- Look for errMsg field indicating error message
- Review server logs for replication error details
- Verify network connectivity to primary
- Check authentication credentials if authentication enabled
- Restart mongod service if replication stuck

### Q160: What causes zombie oplog holes during network failures?

- Oplog holes occur when entries skipped during crashes
- Cannot use oplog for point-in-time recovery with holes
- Restart replica set member cleanly to avoid holes
- Monitor oplog continuity in backup validation tests
- Use snapshots instead of oplog for critical recovery points
- Implement process to detect and alert on oplog discontinuities

---

## 12. MIGRATION & DATA TRANSFER

### Q161: How do I migrate data from MySQL to MongoDB?

- Design MongoDB schema based on application needs
- Use tools like Tapdata or custom scripts for migration
- Extract data from MySQL tables and transform
- Insert transformed data into MongoDB collections
- Verify record counts and sample data accuracy
- Run dual write period to ensure consistency

### Q162: What is the procedure for zero-downtime application migration?

- Set up dual write to both old and new MongoDB deployments
- Verify data consistency between systems
- Gradually shift read traffic to new MongoDB
- Complete write migration when comfortable
- Decommission old system after verification period
- Document migration procedure for future reference

### Q163: How do I migrate from standalone MongoDB to replica set?

- Add secondary nodes to standalone configuration
- Initiate replica set with rs.initiate()
- Standalone automatically becomes primary
- Monitor initial sync on secondary members
- Verify replica set health with rs.status()
- Update connection strings to point to replica set

### Q164: What is the procedure for upgrading MongoDB version?

- Test upgrade in staging environment first
- Backup current database before starting
- For replica sets, upgrade secondaries first one by one
- Stepdown primary and upgrade it last
- Verify cluster health after each member upgrade
- Monitor for errors in logs after upgrade completes

### Q165: How do I handle large collection migration between databases?

- Use aggregation pipeline with $out stage for migration
- Backup target database before migration
- Monitor migration progress and resource usage
- Verify document counts match before and after
- Drop source collection after successful migration
- Test application queries on migrated collection

### Q166: What is the procedure for sharded collection creation?

- Identify appropriate shard key for distribution
- Use sh.shardCollection() on existing collection
- Mongos distributes data across shards
- Monitor chunk distribution with sh.status()
- Verify chunk balancing progresses normally
- Test queries route correctly to appropriate shards

### Q167: How do I handle data skew after initial sharding?

- Data skew occurs when shard key values uneven
- Use reshardCollection() to redistribute data
- Choose different shard key if current one unsuitable
- Monitor chunk distribution after resharding
- Verify query performance after resharding
- Document shard key selection rationale

### Q168: What is the procedure for adding new shard to production cluster?

- Create replica set for new shard
- Use sh.addShard() to add to cluster
- Monitor balancing operation closely
- Verify chunk distribution with sh.status()
- Test query routing to new shard
- Adjust monitoring and backup procedures

### Q169: How do I migrate data from sharded to non-sharded deployment?

- Use mongodump to backup all shards
- Create single MongoDB instance or replica set
- Use mongorestore to load data
- Verify all documents restored successfully
- Update application connection strings
- Decommission sharded cluster after verification

### Q170: What is the procedure for moving collection between shards?

- Use moveChunk command to move chunks to target shard
- Run one at a time to avoid migration locks
- Monitor balancer status during movement
- Verify chunk distribution with sh.status()
- Test query performance after relocation
- Document reason for manual chunk movement

### Q171: How do I handle version incompatibility during migration?

- Check MongoDB version compatibility matrix
- Older clients may not work with newer servers
- Newer drivers may not work with older servers
- Plan driver upgrades alongside server upgrades
- Test driver and server combinations before production
- Document minimum version requirements

### Q172: What is the procedure for migrating from MongoDB Atlas to self-managed?

- Backup Atlas cluster with snapshots
- Download backup to local storage
- Restore to self-managed MongoDB cluster
- Verify all data restored correctly
- Perform parallel run with dual writes initially
- Cut over to self-managed after verification

### Q173: How do I migrate from self-managed to MongoDB Atlas?

- Export data from self-managed using mongodump
- Create Atlas cluster with same size
- Upload and restore data to Atlas
- Configure network access and authentication
- Test application connectivity to Atlas
- Migrate application traffic to Atlas cluster

### Q174: What is the proper procedure for data sync verification?

- Compare document counts between source and destination
- Query sample records to verify data accuracy
- Check index count and configuration
- Validate foreign key relationships if applicable
- Test application queries against both databases
- Document findings and resolve discrepancies

### Q175: How do I handle schema changes during migration?

- Define new schema before starting migration
- Use aggregation pipeline to transform data
- Test transformation with sample data first
- Apply transformation during bulk migration
- Verify new schema compliance in destination
- Archive old data before decommissioning

### Q176: What is the backup procedure before migration?

- Full backup of current production database
- Store backup in secure separate location
- Verify backup integrity before starting migration
- Test backup restore in staging environment
- Document backup details and retention period
- Keep backup available during testing phase

### Q177: How do I handle concurrent writes during migration?

- Dual write to both old and new systems during cutover
- Implement idempotent write logic
- Handle write conflicts between systems
- Verify data consistency periodically
- Use transaction wrapper for atomic operations
- Document conflict resolution procedures

### Q178: What is the procedure for parallel testing after migration?

- Run identical workload on both old and new systems
- Compare query results and performance metrics
- Identify performance gaps or issues
- Optimize new system to match or exceed old performance
- Collect metrics for final cutover decision
- Document testing results

### Q179: How do I handle application code changes during migration?

- Update connection strings to new MongoDB deployment
- Test connection in staging environment
- Verify all queries work against new database
- Handle any schema changes in application code
- Test application against both systems during cutover
- Document code changes and rollback procedures

### Q180: What is the final cutover procedure?

- Notify all users of maintenance window
- Stop writes to old system
- Verify all pending operations complete
- Final sync from old to new system
- Switch application connection strings to new system
- Monitor for errors and be ready to rollback

---

## 13. PERFORMANCE TUNING & OPTIMIZATION

### Q181: How do I identify the most expensive queries?

- Enable profiling with db.setProfilingLevel(1)
- Set slowms threshold to capture queries above duration
- Query system.profile collection for detailed execution stats
- Sort by millis field to identify longest queries
- Analyze query execution plan with explain()
- Create indexes to optimize top expensive queries

### Q182: What is query selectivity and its impact?

- Selectivity measures percentage of documents matching query
- High selectivity queries return small fraction of documents
- Low selectivity queries return large percentage of documents
- Design indexes to maximize query selectivity
- Combine field filters to increase selectivity
- Monitor selectivity improvement after index creation

### Q183: How do I optimize data model for MongoDB?

- Denormalize related data into single document when possible
- Use embedding for one-to-few relationships
- Use referencing for one-to-many or many-to-many relationships
- Avoid deeply nested structures limiting query flexibility
- Balance data duplication against query complexity
- Test query patterns before finalizing schema

### Q184: What is projection and how do I use it efficiently?

- Projection reduces data returned from queries
- Only return fields needed by application
- Projection reduces network transmission size
- Reduces memory usage in application code
- Use projection in find and aggregation queries
- Monitor projection effectiveness with explain()

### Q185: How do I optimize sort operations in MongoDB?

- Use index on sort fields for best performance
- Place sort before data processing to minimize sorting
- Compound indexes should have sort fields last
- Use allowDiskUse for sorts exceeding available memory
- Sort in index order to avoid additional sort step
- Consider data volume before using sort

### Q186: What is batch processing and why is it useful?

- Batch processing reduces network round trips
- insertMany and bulkWrite process multiple operations at once
- Larger batch sizes improve throughput for imports
- Balance batch size against memory usage
- Monitor batch processing performance metrics
- Use batch size between 100-1000 for optimal results

### Q187: How do I reduce network latency in MongoDB operations?

- Reduce round trip time by batching operations
- Use connection pooling to reuse connections
- Place application close to MongoDB geographically
- Use low-latency network between application and database
- Monitor ping latency to identify network issues
- Consider read-only replicas in application datacenters

### Q188: What is connection pool size optimization?

- Too small pool forces waiting for available connections
- Too large pool wastes memory and resources
- Default pool size often works for most applications
- Monitor active connections and adjust if queue delays observed
- Typical range 10-50 connections per application instance
- Calculate required pool size based on concurrent users

### Q189: How do I optimize bulk insert operations?

- Use insertMany for multiple documents instead of individual inserts
- Set ordered:false for better performance when order not important
- Batch documents into groups of 100-1000
- Monitor insert throughput with mongostat
- Use write concern acknowledged for performance
- Consider journal settings for insert-only scenarios

### Q190: What is write amplification and how to minimize it?

- Write amplification occurs when small writes cause large I/O
- Journal writes, data file writes, and index updates amplify writes
- Compression reduces write amplification
- Batch writes to minimize repeated operations
- Disable unnecessary indexes reducing write overhead
- Monitor disk I/O for write amplification signs

### Q191: How do I optimize memory usage in aggregation?

- Avoid large group stages creating many groups
- Use allowDiskUse to spill to disk for large results
- Limit result set early with match stage
- Project only required fields in project stage
- Monitor memory usage with db.serverStatus()
- Break large aggregations into multiple stages

### Q192: What is join performance and optimization?

- $lookup stage performs join-like operations
- Joining large collections can impact performance
- Ensure equality join fields are indexed
- Limit lookup results with pipeline array parameter
- Consider denormalization if join expensive
- Test join performance with explain() output

### Q193: How do I optimize text search queries?

- Create text index on text fields
- Text indexes require significant storage space
- Wildcards not supported in text search
- Phrase search supported with quoted strings
- Language support for stemming and stop words
- Consider full-text search engines for complex scenarios

### Q194: What is data locality and why does it matter?

- Data locality minimizes network traffic for queries
- Co-locate application and database in same datacenter
- Reduced latency improves query response time
- Critical for high-frequency trading and real-time applications
- Multi-region setups sacrifice locality for resilience
- Monitor network latency as performance indicator

### Q195: How do I benchmark MongoDB performance?

- Use standard workload tools like POCDriver
- Measure throughput and latency metrics
- Compare performance across different configurations
- Run benchmarks in isolated environment
- Repeat tests for consistency and variance analysis
- Document benchmark results for future comparison

### Q196: What is the impact of compression on performance?

- WiredTiger compression reduces storage size
- Compression uses CPU cycles during read/write
- Reduces network bandwidth for replication
- Useful when storage cost higher than CPU cost
- Monitor CPU usage increase from compression
- Adjust compression if CPU becomes bottleneck

### Q197: How do I optimize transactions for MongoDB?

- Multi-document transactions support atomicity
- Keep transactions as short as possible
- Minimize conflicting updates within transactions
- Use transaction timeout to detect deadlocks
- Consider single document updates if transaction not essential
- Test transaction performance impact

### Q198: What is conflict resolution in transactions?

- Write conflicts occur when transaction reads modified data
- Automatic retry mechanism handles some conflicts
- Application should implement retry logic
- Exponential backoff reduces retry storms
- Monitor transaction abort rates
- Optimize queries to reduce conflicts

### Q199: How do I handle hot data scenarios?

- Hot data is frequently accessed subset of total data
- Cache hot data in application memory
- Use read preference to distribute reads
- Consider separate replica for hot data reads
- Monitor access patterns to identify hot spots
- Implement caching strategy for most accessed data

### Q200: What is the procedure for performance baseline?

- Establish baseline metrics under normal load
- Measure latency, throughput, and resource usage
- Run baseline tests before production deployment
- Compare new changes against baseline
- Alert when metrics deviate from baseline
- Update baseline periodically as system evolves

---

## 14. ADDITIONAL OPERATIONAL SCENARIOS

### Q201: How do I handle MongoDB server startup failures?

- Check mongod process is running
- Review logs in /var/log/mongodb/mongod.log
- Verify data directory ownership and permissions
- Check available disk space on data partition
- Ensure port 27017 not in use by other process
- Manually start with verbose logging for details

### Q202: What is the procedure for emergency restart of MongoDB?

- Send SIGTERM signal for graceful shutdown
- Wait for shutdown completion (check process stopped)
- Restart mongod service normally
- Verify cluster health with rs.status()
- Monitor for replication catch-up on secondaries
- Document restart reason and timestamp

### Q203: How do I handle unresponsive MongoDB process?

- Verify process still running with ps command
- Check connectivity with mongosh connection attempt
- Send SIGTERM for graceful shutdown attempt
- Send SIGKILL if graceful shutdown fails
- Verify shutdown completion
- Restart MongoDB normally

### Q204: What causes swap memory usage in MongoDB?

- Insufficient RAM for working set
- MongoDB spilling to swap degrading performance
- Disable swap if possible or increase RAM
- Monitor memory usage continuously
- Alert when swap usage detected
- Resize collection or archive data if necessary

### Q205: How do I handle insufficient disk space errors?

- Stop application writes immediately
- Identify which directories filled
- Delete unnecessary files or collections
- Extend filesystem if possible
- Verify sufficient space before restart
- Implement disk usage monitoring

### Q206: What is the recovery procedure for corrupted data files?

- MongoDB cannot automatically repair data files
- Restore from clean backup as only recovery option
- Point-in-time recovery from oplog if available
- Verify backup integrity before restore
- Investigate corruption cause to prevent recurrence
- Document corruption incident for post-mortem

### Q207: How do I handle memory leak in MongoDB?

- Monitor memory usage over extended period
- Check if memory returns after garbage collection
- Review MongoDB version for known memory leak bugs
- Upgrade to patched version if available
- Periodically restart mongod as temporary workaround
- Report persistent memory leaks to MongoDB support

### Q208: What is the procedure for handling excessive lock contention?

- Monitor lock percentage over time
- Identify operations causing lock contention
- Optimize long-running transactions
- Break transactions into smaller pieces
- Add more hardware resources if needed
- Consider write scaling options

### Q209: How do I handle cascading replication failures?

- Primary failure triggers election of new primary
- Secondary failure reduces redundancy but cluster continues
- Multiple member failures may cause unavailability
- Recover failed members to restore redundancy
- Monitor member health continuously
- Alert on any member becoming unavailable

### Q210: What is checkpoint lag and why does it matter?

- Checkpoint lag is delay between writes and durability
- Journal captures writes immediately for recovery
- Actual data written to disk in checkpoint operation
- Longer lag increases data loss risk on crash
- Configure checkpointDelaySecs based on RPO requirement
- Monitor checkpoint progress with db.serverStatus()

# MongoDB Database Administration Complete FAQ Guide - Part 2
## Questions 211-500

---

## 15. TRANSACTIONS & ACID COMPLIANCE

### Q211: What are MongoDB transactions and when should I use them?

- Transactions group multiple read and write operations into atomic unit
- All operations succeed or entire transaction rolls back on failure
- Available for replica sets since MongoDB 4.0
- Available for sharded clusters since MongoDB 4.2
- Add overhead so use only when atomicity across documents required
- Single document operations inherently atomic without transaction overhead

### Q212: What is atomicity in MongoDB transactions?

- Atomicity guarantees all operations complete or none apply
- Either all writes succeed or all are rolled back
- Partial writes never occur leaving database in inconsistent state
- Prevents scenarios like money transferred but not credited
- Critical for financial transactions and inventory management
- Default ACID property since MongoDB 4.0

### Q213: What is the transaction timeout default value?

- Default transaction timeout is 60 seconds
- Configurable with transactionLifetimeLimitSeconds parameter
- Long running transactions automatically terminated after timeout
- Prevents resource exhaustion from long-running operations
- Set appropriate timeout based on longest expected transaction
- Monitor transaction duration with db.serverStatus()

### Q214: How do I start and commit a transaction?

- Use session.startTransaction() to begin transaction
- Execute read and write operations normally within transaction
- Call session.commitTransaction() to apply all changes
- Call session.abortTransaction() to rollback changes
- Committed data immediately visible to other sessions
- Aborted transactions appear never to have occurred

### Q215: What is snapshot isolation in transactions?

- Snapshot isolation provides consistent view of data throughout transaction
- Reads see data as it existed when transaction started
- Other transactions cannot affect reads within transaction
- Prevents dirty reads and non-repeatable reads
- Provides strong consistency guarantees
- Default isolation level for MongoDB transactions

### Q216: How do I handle transaction conflicts and retries?

- Write conflict occurs when transaction reads data another modifies
- Automatic retry mechanism handles some conflicts
- Application should implement exponential backoff retry logic
- Monitor transaction abort rates with serverStatus()
- Avoid long-running transactions to reduce conflicts
- Test conflict scenarios in staging environment

### Q217: What are the restrictions on multi-document transactions?

- Cannot write to capped collections in transactions
- Cannot access admin and config databases in transactions
- Cannot use explain() command within transaction
- Cannot create or drop collections in MongoDB versions before 4.4
- Cannot change shard keys in transactions
- Pre-create collections before transactions if needed

### Q218: How do I use sessions for consistent read operations?

- Sessions maintain causal ordering across operations
- Reads within session always see writes from earlier operations
- Multiple transactions use same session for consistency
- Create session with client.startSession()
- Pass session to each operation in read sequence
- Session ends when explicitly closed

### Q219: What is causal consistency in MongoDB?

- Causal consistency ensures operations ordered by causality
- Write followed by read within session always sees write
- Read from secondary sees primary writes from same session
- Requires write concern majority and read concern majority
- Enables building applications with strict consistency
- Overhead minimal for most applications

### Q220: How do I debug transaction failures?

- Check server logs for transaction error messages
- Use db.serverStatus().transactions for failure statistics
- Enable profiling to capture transaction details
- Review error codes to identify retry vs fatal errors
- Test transaction logic in staging environment first
- Implement comprehensive error handling in application

### Q221: What is write concern and its relationship to transactions?

- Write concern specifies acknowledgment level for writes
- Majority write concern prevents data loss on node failure
- Transaction durability depends on write concern level
- Use write concern majority for production transactions
- Acknowledge level impacts commit time and latency
- Configure write concern in connection string

### Q222: How do I optimize transaction performance?

- Keep transactions as short as possible under 1 second
- Batch operations to minimize round trips
- Avoid long-running queries within transactions
- Pre-fetch required documents before transaction start
- Optimize query predicates to reduce lock time
- Monitor transaction timing with db.currentOp()

### Q223: What happens during transaction rollback?

- All changes applied by transaction discarded completely
- Database returns to state before transaction started
- Intermediate states never visible to other operations
- Rollback immediate no cleanup phase required
- Rolled back data immediately available to other writers
- No recovery or cleanup operations needed

### Q224: How do I create multi-document transaction example?

- Start session and transaction
- Execute read from first collection
- Execute write to second collection
- Execute write to third collection
- Commit all changes atomically
- Rollback if any operation fails

### Q225: What is withTransaction helper and why use it?

- withTransaction automatically handles commit and retry logic
- Simplifies transaction code in application
- Implements recommended retry behavior automatically
- Available in Python, Node.js, and other drivers
- Reduces boilerplate error handling code
- Recommended for production transaction code

---

## 16. SESSIONS & CLIENT CONNECTIVITY

### Q226: What is MongoDB session and how do I create one?

- Session represents logical client connection to MongoDB
- Created with client.startSession() in driver
- Sessions maintain session state across multiple operations
- Multiple sessions share same underlying connection pool
- Sessions have unique sessionId used for consistency
- Must explicitly close session when complete

### Q227: How do I configure session timeout behavior?

- Idle sessions automatically closed after timeout period
- Default timeout is 30 minutes for inactive sessions
- Configure logicalSessionTimeoutMinutes in cluster
- Server monitors session activity and expires idle ones
- Application can explicitly end session earlier if needed
- Expired session throws error on next operation

### Q228: What is the maximum session duration?

- Sessions can persist until explicitly closed
- No hard maximum session lifetime by default
- Logical session timeout applies to idle sessions
- Server can terminate long-running sessions for resources
- Application should handle session expiration gracefully
- Reconnect automatically in most drivers

### Q229: How do I use serverSession for internal operations?

- ServerSession manages server-side session data
- Application does not directly interact with serverSession
- Driver manages serverSession lifecycle internally
- Tracks operations and memory usage per session
- Monitor session count with db.serverStatus()
- Alert if session count increases unexpectedly

### Q230: What are the implications of session count on performance?

- Each session requires server memory for state tracking
- High session count increases memory usage
- Monitor session count with db.serverStatus().sessions
- Implement connection pooling to reuse sessions
- Close idle sessions to reduce memory usage
- Alert when session count approaches system limits

---

## 17. CHANGE STREAMS & REAL-TIME OPERATIONS

### Q231: What are change streams and what do they enable?

- Change streams provide real-time notifications of data changes
- Application subscribes to changes without polling database
- Available on collections, databases, and entire deployments
- Receives insert, update, delete, and replace operations
- Uses aggregation framework for filtering and transformation
- Requires replica set or sharded cluster

### Q232: How do I implement change stream monitoring?

- Call watch() on collection without parameters
- Iterate over returned change stream cursor
- Each iteration returns change document
- Change document contains operationType and fullDocument
- Continue iteration until change stream closed
- Handle connection loss and reconnection

### Q233: What are resume tokens in change streams?

- Resume token uniquely identifies change in stream
- Allows resuming change stream from specific point
- Useful for recovery after connection loss
- Token encoded in resumeAfter parameter
- Enable 7-day resume capability with continuous oplog
- Store resume token to restart monitoring

### Q234: How do I filter change stream events?

- Use pipeline parameter in watch() for filtering
- Apply $match stage to select specific event types
- Filter on operationType for insert, update, delete, replace
- Filter on fullDocument fields for specific data
- Filter on ns for specific collections or databases
- Reduce stream volume with selective filtering

### Q235: How do I handle change stream connection failures?

- Driver automatically attempts reconnection
- Resume from resume token on reconnection
- Application should handle reconnection with backoff
- Store resume token for recovery across restarts
- Implement monitoring to detect connection failures
- Alert on repeated connection failures

### Q236: What is the performance impact of change streams?

- Change stream reading adds minimal database load
- Filtering in pipeline reduces data transmission
- Many concurrent change streams may impact performance
- Monitor database load with monitoring tools
- Consolidate multiple streams if possible
- Consider limiting number of concurrent streams

### Q237: How do I capture full document in change stream?

- Specify fullDocument parameter in watch()
- Value "whenAvailable" includes updated document
- Value "required" fails if full document unavailable
- Enables building complete audit trail
- Consider data size impact of including full document
- Alternative query new document separately

### Q238: What is showExpandedEvents option in change streams?

- showExpandedEvents captures create and drop operations
- Includes createIndexes, dropIndexes events
- Useful for capturing entire schema evolution
- Available in MongoDB 6.0 and later
- Not part of Stable API V1
- Increases change stream overhead capturing more events

### Q239: How do I use change streams for data synchronization?

- Subscribe to changes on primary system
- Apply captured changes to secondary system
- Ensures eventual consistency across systems
- Resume from token on network failures
- Test synchronization latency and completeness
- Implement conflict resolution for concurrent changes

### Q240: How do I implement change stream backpressure handling?

- Backpressure occurs when processing slower than changes arrive
- Implement queueing to buffer changes
- Process queue at sustainable rate
- Monitor queue depth for bottlenecks
- Increase processing threads or instances if needed
- Implement monitoring to detect backpressure

### Q241: How do I filter out specific operations from change stream?

- Use $match stage with operationType filter
- Exclude operations with negation operators
- Example exclude update operations for audit trail
- Combine multiple conditions with $and and $or
- Test filter behavior with test change stream
- Monitor filtered vs total operations

### Q242: What are the limitations of change streams?

- Change streams limited to database events
- Time series collections do not support change streams
- Capped collections limited change stream support
- Resume token retention depends on oplog size
- Cannot use all aggregation stages in pipeline
- Network latency affects change detection

---

## 18. TIME SERIES COLLECTIONS

### Q243: What are time series collections and when to use them?

- Time series collections optimized for time-stamped measurements
- Store data points efficiently from sensors and monitoring systems
- Automatically bucket related measurements together
- Reduce storage space and improve query performance
- Created with timeField and metaField specification
- Introduced in MongoDB 5.0

### Q244: How do I create a time series collection?

- Use db.createCollection() with timeseries option
- Specify timeField for timestamp field name
- Specify metaField for grouping identifier (e.g. sensor ID)
- Set granularity to seconds, minutes, or hours
- Configure expireAfterSeconds for TTL
- Verify collection type with db.getCollectionInfos()

### Q245: What is granularity setting in time series collections?

- Granularity hints how frequently measurements arrive
- Seconds for high frequency sub-minute data
- Minutes for 1-5 minute interval data
- Hours for hourly aggregated data
- Affects internal bucketing strategy
- Too fine-grained creates inefficient small buckets

### Q246: How does bucketing work in time series collections?

- MongoDB automatically groups measurements into buckets
- Documents with same metaField grouped together
- Buckets contain measurements from same time window
- Bucketing transparent to application
- Improves compression and query performance
- Buckets automatically closed after size limit

### Q247: What are the restrictions on time series collections?

- Deletes not supported in MongoDB 5.0 (added in 5.1)
- Limited updates to measurement fields only
- Cannot create/drop indexes like regular collections
- Cannot use change streams before MongoDB 6.0
- Cannot use some aggregation stages like $merge
- Schema defined at creation time

### Q248: How do I insert data into time series collections?

- Use insertOne() or insertMany() like regular collections
- Documents must contain timeField with timestamp
- Include metaField for grouping if defined
- Include measurement fields with sensor data
- insertMany() more efficient than individual inserts
- Batch inserts fill buckets more efficiently

### Q249: How do I query time series collections efficiently?

- Filter by timeField for date range queries
- Filter by metaField for specific sensor
- Combine both for efficient queries
- MongoDB 6.3+ automatically creates compound index
- Aggregation pipelines work on time series collections
- Verify explain() shows efficient execution plan

### Q250: How do I delete documents from time series collection?

- Delete operations supported since MongoDB 5.1
- Delete entire documents by filter
- Cannot delete individual measurements from bucket
- Cannot use update operators to remove fields
- Use deleteOne() or deleteMany() normally
- Verify deletions with subsequent queries

### Q251: How do I update documents in time series collections?

- Limited updates to measurement fields only
- Cannot modify timeField or metaField
- Cannot add new fields to bucket
- Use updateOne() or updateMany() for measurements
- Test update behavior before production use
- Consider data model if many updates needed

### Q252: What is the performance impact of time series collections?

- Significantly reduces storage space for time series data
- Improved query performance for time range queries
- Some limitations require workarounds
- Test performance with production-like data
- Compare storage and query performance
- Monitor resource usage after enabling time series

### Q253: How do I implement TTL on time series collections?

- Set expireAfterSeconds during collection creation
- MongoDB automatically deletes old documents
- Deletion runs periodically every 60 seconds
- Useful for short-term metrics retention
- Cannot modify expireAfterSeconds after creation
- Verify documents actually expire with queries

### Q254: How do I migrate regular collection to time series?

- Create new time series collection with appropriate schema
- Use aggregation pipeline with $out to transform data
- Include timeField and metaField in output
- Verify data completeness after migration
- Drop old collection when migration complete
- Test application queries work with time series

### Q255: How do I monitor time series collection storage?

- Use db.collection.stats() to check size
- Compare storage before and after conversion
- Monitor query performance metrics
- Check index sizes included in stats
- Track disk space savings
- Monitor query latency for improvements

---

## 19. ADVANCED INDEXING STRATEGIES

### Q256: What are wildcard indexes and when to use them?

- Wildcard indexes support queries on unknown fields
- Useful for flexible schema collections
- Index specified fields matching pattern
- Cannot support multiple fields in single query
- Sparse indexes excluding null values
- Use carefully as they consume storage space

### Q257: How do I create wildcard index on dynamic fields?

- Use db.collection.createIndex({fieldPattern: 1})
- Pattern uses wildcard specifier path
- Example db.collection.createIndex({"metadata.$**": 1})
- Indexes all fields under metadata recursively
- Supports queries on any field matching pattern
- Cannot use with compound or text indexes

### Q258: What are hidden indexes and their purpose?

- Hidden indexes ignored by query planner
- Useful for testing without dropping index
- Hide index to evaluate alternative plans
- Show index to restore to query planner
- Saves rebuild time compared to drop/recreate
- Monitor query performance with index hidden

### Q259: How do I hide and unhide indexes?

- Use db.collection.hideIndex() specifying index name
- Query planner excludes hidden index from plans
- Use db.collection.unhideIndex() to restore
- Verify index status with db.collection.getIndexes()
- No performance cost to hiding or unhiding
- Test queries before hiding important indexes

### Q260: What is index intersection and when does it occur?

- Index intersection uses multiple indexes for single query
- Query planner selects most selective index first
- Remaining filters applied to index results
- Less efficient than single compound index
- Monitor explain() output for intersection scenarios
- Create compound index if intersection common

### Q261: How do I create index for sorting queries?

- Create index with sort field at end of compound index
- Compound index fields ordered left to right
- Query filters on early fields, sorts on later fields
- ESR rule specifies optimal index field ordering
- Test with explain() to verify index used for sort
- Descending sort requires index in same order

### Q262: What is the ESR rule for compound indexes?

- Equality fields first in compound index
- Sort fields second in compound index
- Range fields last in compound index
- Optimal index design for common query patterns
- Example index on {status: 1, createdAt: 1, name: -1}
- Apply rule to maximize index effectiveness

### Q263: How do I handle queries with multiple sort orders?

- MongoDB cannot efficiently sort by field in one direction and another field opposite
- Create index matching one common sort order
- Accept slight inefficiency for less common sorts
- Consider query patterns before index design
- Test most common sort patterns with indexes
- Accept tradeoff between different query patterns

### Q264: How do I use explain() to optimize indexes?

- Run explain() with executionStats to see actual execution
- Look for COLLSCAN indicating inefficient full scan
- Check docsExamined vs docsReturned ratio
- Identify missing indexes for high-ratio queries
- Verify newly created indexes used after creation
- Re-run explain() to confirm optimization

### Q265: How do I handle index name collisions?

- Index names must be unique per collection
- Rename index before creating new one
- Use db.collection.dropIndex() to remove conflicts
- Automatic naming follows pattern fieldname_1 or fieldname_-1
- Specify custom index name with name parameter
- Document index naming convention

### Q266: What is index key limit and impact?

- Indexed field values must fit within limits
- Very long string values impact index size
- Compound indexes have combined key limit
- Monitor index size growth with stats()
- Consider if indexing necessary for large fields
- Alternative hash index for very large values

### Q267: How do I handle index creation on large collections?

- Background index creation blocks writes shorter period
- Foreground index creation blocks until complete
- Plan index creation during maintenance window
- Monitor mongod for resource usage during build
- Very large collections may take hours to index
- Consider replica set index building from secondary

### Q268: How do I rebuild indexes?

- Use db.collection.reIndex() to rebuild all indexes
- Temporary doubles disk space during rebuild
- All queries use collection scan during rebuild
- Useful for index fragmentation recovery
- Alternative drop and recreate specific index
- Plan rebuild during low-traffic periods

### Q269: How do I handle partial index expiration?

- Partial indexes with expiration may need tuning
- TTL monitor runs periodically not exactly at time
- Expect document deletion within 60 seconds of expiration
- Cannot rely on precise expiration timing
- Account for grace period in application logic
- Implement fallback cleanup logic if needed

### Q270: How do I optimize geospatial indexes?

- Use 2dsphere index for earth-like sphere queries
- Use 2d index only for flat plane geometries
- Create index on location field with coordinates
- Verify index used with explain() output
- Test query performance after index creation
- Monitor index size for large coordinate datasets

---

## 20. TEXT SEARCH & GEOSPATIAL QUERIES

### Q271: What is text index and how to implement full-text search?

- Text index enables full-text search on string fields
- Single text index per collection only
- Create with db.collection.createIndex({field: "text"})
- Query with db.collection.find({$text: {$search: "keywords"}})
- Supports phrase search with quotes
- Case insensitive and diacritic insensitive

### Q272: How do I create compound text index?

- Combine multiple fields in single text index
- Example db.collection.createIndex({title: "text", content: "text"})
- Specify weights for field importance
- Higher weight improves ranking for field matches
- Single compound text index per collection
- Query searches all indexed fields

### Q273: How do I handle text search with language support?

- Specify language with language parameter in index
- Support for 17 languages with stemming
- Default language is English
- Override default language per document
- Use language field in document for per-document language
- Verify language support in documentation

### Q274: How do I rank text search results?

- Text search returns textScore for result ranking
- Higher score indicates better match
- Sort by {score: {$meta: "textScore"}} to rank
- Phrase searches score higher than individual terms
- Weighted fields affect scoring calculation
- Test ranking with various search terms

### Q275: How do I create geospatial index for location queries?

- Use 2dsphere index for queries on earth-like sphere
- Create with db.collection.createIndex({location: "2dsphere"})
- Field must contain GeoJSON objects or legacy pairs
- Support for Point, LineString, and Polygon geometries
- Query nearby locations within distance
- Verify index used with explain()

### Q276: What are GeoJSON objects and how to use them?

- GeoJSON format stores geographic coordinates
- Point type represents single location with coordinates
- LineString connects sequence of points
- Polygon encloses area with coordinate rings
- MultiPoint, MultiLineString, MultiPolygon for multiple
- Store in location field for geospatial queries

### Q277: How do I query for nearby locations?

- Use $near operator to find closest locations
- Specify center point and max distance
- Returns results sorted by distance
- Example db.collection.find({location: {$near: {$geometry: point}}})
- Requires 2dsphere index for accuracy
- Optionally specify minDistance to exclude too-close results

### Q278: How do I find locations within polygon boundary?

- Use $geoWithin operator with Polygon geometry
- Query returns documents inside boundary
- Example uses $geometry with Polygon specification
- Efficient for finding locations in region
- Requires geospatial index
- Test with known locations before production

### Q279: How do I handle 2d legacy coordinate queries?

- 2d index supports flat plane coordinate queries
- Only use if coordinates not on earth
- Deprecated in favor of 2dsphere for real-world data
- Use with flat coordinate systems like game maps
- Cannot mix 2d and 2dsphere in same index
- Migrate to 2dsphere for geo applications

### Q280: How do I optimize geospatial index performance?

- Create 2dsphere index before queries
- Monitor index size with stats()
- Verify explain() shows index usage
- Consider sparse index if not all documents have location
- Test query performance with actual data
- Monitor for slow geospatial queries

---

## 21. WRITE OPERATIONS & BULK OPERATIONS

### Q281: What is write concern and its impact?

- Write concern specifies acknowledgment level
- Unacknowledged (0) returns immediately no confirmation
- Acknowledged (1) confirms write applied to single node
- Majority confirms applied to majority of replicas
- Majority provides durability guarantees
- Trade latency for durability with write concern

### Q282: How do I configure write concern in connection string?

- Use w parameter in connection string
- w=0 for unacknowledged writes
- w=1 for acknowledged on primary
- w="majority" for majority acknowledgment
- j=true for journal flush confirmation
- wtimeout parameter sets maximum wait time

### Q283: What are bulk write operations and advantages?

- insertMany, updateMany, deleteMany for bulk operations
- Reduces network round trips compared to individual operations
- Ordered operations stop on first error
- Unordered operations continue despite individual failures
- Significant throughput improvement for large batches
- Useful for data imports and cleanup tasks

### Q284: How do I use ordered vs unordered bulk operations?

- ordered: true (default) stops on first error
- ordered: false continues despite individual failures
- Unordered faster for independent updates
- Ordered safer for dependent operations
- Verify partial completion acceptable for unordered
- Monitor error details for failed operations

### Q285: How do I handle partial failures in bulk operations?

- Ordered bulk stops after first error
- Unordered bulk returns all successes and failures
- Check result for writeErrors array
- Verify matched and modified counts
- Retry failed operations if necessary
- Log failures for investigation

### Q286: What is the maximum batch size for bulk operations?

- Default batch size 100,000 documents
- Configurable but practical limits based on memory
- Larger batches use more memory on server
- Balance throughput against memory usage
- Monitor batch size impact on performance
- Adjust based on available server memory

### Q287: How do I implement upsert operations?

- Use updateOne() or updateMany() with upsert: true
- If no match found, insert new document
- Combine update with upsert creates if not exists
- Useful for idempotent update operations
- Specify unique identifier for upsert matching
- Verify upserted count in operation result

### Q288: How do I replace entire document?

- Use replaceOne() to replace matched document
- Replacement overwrites all fields except _id
- Specify replacement document completely
- Cannot use update operators with replaceOne
- Useful for data normalization
- Verify replacement with subsequent query

### Q289: How do I perform conditional writes?

- Use query conditions to select documents
- Apply update only if conditions match
- Retry if update fails due to document change
- Implement optimistic locking with version field
- Alternative pessimistic locking with exclusive read
- Test conditional write logic thoroughly

### Q290: How do I handle write failures and retries?

- Implement retry logic with exponential backoff
- Check error code to determine if retryable
- Some errors permanently fail without retry
- Monitor retry success rates
- Set maximum retry attempts
- Log retry behavior for debugging

---

## 22. DATA VALIDATION & SCHEMA ENFORCEMENT

### Q291: How do I enforce schema validation?

- Use JSON schema in collMod validator
- Define required fields and data types
- Validate on insert and update operations
- Set validationLevel for strict or moderate
- Reject non-conforming documents
- Allow legacy data with moderate validation

### Q292: What is JSON schema validator?

- Validator uses JSON schema specification
- Define structure, required fields, types
- Support for nested objects and arrays
- Enforce constraints like minLength, pattern
- Complex validation rules with $jsonSchema
- Catch data quality issues at write time

### Q293: How do I handle validation failures?

- Validation failure rejects write operation
- Application must handle validation error
- Modify data to conform to schema
- Alternative is relax validation level
- Test validation rules before production
- Document schema requirements for developers

### Q294: What is validation action and level?

- validationAction error rejects invalid documents
- validationAction warn logs invalid documents
- validationLevel strict validates all inserts and updates
- validationLevel moderate validates except existing documents
- Combination allows gradual migration to strict schema
- Test before changing validation level

### Q295: How do I update collection validation?

- Use db.runCommand() with collMod and validator
- Replace existing validator with new one
- Changes apply immediately to new documents
- Existing documents validated per validationLevel
- Test new validation with sample data
- Document validation changes

### Q296: How do I validate nested documents?

- JSON schema supports nested object validation
- Define properties for each level
- Validate array elements with items keyword
- Complex validation for hierarchical data
- Test nested validation with examples
- Monitor validation errors

### Q297: How do I enforce unique constraints?

- Create unique index on field
- Duplicate value rejected on insert/update
- Sparse unique index allows multiple nulls
- Compound unique index on multiple fields
- Test uniqueness before production use
- Monitor for duplicate value errors

### Q298: How do I handle validation for optional fields?

- JSON schema allows fields not in required list
- optional fields may be present or absent
- Define constraints only when field present
- Use oneOf for conditional validation
- Balance strictness against flexibility
- Test with and without optional fields

### Q299: How do I validate date and time fields?

- JSON schema does not directly validate dates
- Validate timestamp format with pattern
- Validate ISO8601 format with regex pattern
- Ensure timestamps within reasonable range
- Monitor for invalid date values
- Implement application-level date validation

### Q300: How do I migrate to schema validation?

- Start with moderate validation level
- Apply validator to existing collection
- Gradually tighten validation rules
- Monitor validation warnings
- Fix non-conforming documents gradually
- Switch to strict validation when ready

---

## 23. SYSTEM ADMINISTRATION & OPERATIONS

### Q301: How do I monitor system resources on MongoDB server?

- Monitor CPU usage with top or htop
- Monitor memory usage and swap
- Monitor disk I/O operations
- Monitor network throughput
- Use sysstat tools for historical data
- Set up alerting on resource thresholds

### Q302: What is ulimit and how to configure for MongoDB?

- ulimit controls system resource limits per process
- MongoDB needs file descriptor limit of 64000 or higher
- Set nofile limit with ulimit -n command
- Configure in /etc/security/limits.conf for persistence
- Verify MongoDB can open required file descriptors
- Set limits before starting MongoDB

### Q303: How do I enable MongoDB debug logging?

- Set logLevel in configuration to 0-5
- Level 0 is default, level 5 is most verbose
- Restart mongod to apply new log level
- Debug logging helps diagnose issues
- Verbose logging impacts performance
- Return to normal level after troubleshooting

### Q304: Where are MongoDB log files stored?

- Default location /var/log/mongodb/mongod.log
- Configured in systemLog.path in config
- Log rotation with logAppend in config
- Configure rotation period for long-term storage
- Archive old logs for audit compliance
- Monitor log file disk usage

### Q305: How do I analyze MongoDB log files?

- Use grep to find error messages
- Monitor for repeated errors indicating issues
- Track replication lag messages
- Watch for index build completion messages
- Search for authentication failures
- Use log parsing tools for analysis

### Q306: How do I set up MongoDB automatic log rotation?

- Configure systemLog.destination to file
- Enable logAppend for continuous appending
- Set logRotate in configuration for rotation
- Or use Linux logrotate for external rotation
- Verify logs rotate without losing messages
- Monitor rotation success

### Q307: How do I access MongoDB audit logs?

- Audit logs capture authentication and operations
- Enable with security.auditLog in config
- Specify destination for log file
- Filter with auditLog.filter for specific events
- Store audit logs separately for security
- Review audit logs for compliance

### Q308: How do I configure systemd service for MongoDB?

- MongoDB includes systemd service file
- Install in /etc/systemd/system/mongod.service
- Enable with systemctl enable mongod
- Start with systemctl start mongod
- Monitor with systemctl status mongod
- Use journalctl for system-level logging

### Q309: How do I perform graceful MongoDB shutdown?

- Send SIGTERM signal to mongod process
- Process completes current operations then exits
- Verify shutdown completion with process check
- Alternative stop command in mongosh
- Graceful shutdown prevents data corruption
- Use systemctl stop for cleaner shutdown

### Q310: How do I handle zombie MongoDB processes?

- Processes that fail to terminate properly
- Use ps to identify zombie processes
- Send SIGKILL if SIGTERM fails
- Restart mongod service cleanly
- Investigate root cause of failure
- Monitor process lifecycle

### Q311: How do I check MongoDB process resource usage?

- Use ps aux to see basic process info
- Use top for real-time monitoring
- Use ps aux --sort=rss for memory usage
- Monitor CPU percentage and memory
- Check file descriptor count with lsof
- Alert when resource usage exceeds thresholds

### Q312: How do I disable swap on Linux for MongoDB?

- Swap degrades MongoDB performance
- Use swapoff -a to disable all swap
- Remove swap from /etc/fstab for persistence
- Verify swap disabled with free command
- Allocate sufficient RAM to prevent need for swap
- Monitor for swap usage after changes

### Q313: How do I configure Linux kernel parameters for MongoDB?

- Set vm.swappiness to 1 to minimize swap
- Set net.core.somaxconn for connection queue
- Set net.ipv4.tcp_max_syn_backlog for TCP backlog
- Apply in /etc/sysctl.conf for persistence
- Use sysctl -p to apply changes immediately
- Document parameter changes for operations

### Q314: How do I handle MongoDB running out of inodes?

- Inodes track files and directories
- Running out prevents new file creation
- Check inode usage with df -i
- Archive old data to free inodes
- Extended filesystems have high inode count
- Monitor inode usage trending

### Q315: How do I secure MongoDB server access?

- Restrict network access with firewall
- Enable authentication and TLS
- Use strong passwords or LDAP
- Implement network segmentation
- Monitor for unauthorized access attempts
- Regularly review access logs

---

## 24. MONITORING TOOLS & METRICS

### Q316: What is mongostat and its key metrics?

- mongostat provides real-time MongoDB statistics
- Shows operations per second for reads and writes
- Displays memory and lock information
- Shows network in/out bytes per second
- Timestamp indicates when measurement taken
- Useful for quick performance assessment

### Q317: How do I use mongotop for monitoring?

- mongotop shows time spent reading/writing per collection
- Displays operations count and bytes for each collection
- Identifies hot collections with high activity
- Run with --locks for lock time measurement
- Specify --slow milliseconds for slow operations
- Monitor for collections needing optimization

### Q318: What is db.serverStatus() and how to interpret it?

- Returns comprehensive server statistics
- Shows memory usage, storage engine info
- Displays operation counters and timing
- Includes replication and connection status
- Monitor for anomalies in returned values
- Compare values across time to detect changes

### Q319: How do I use db.currentOp() for monitoring?

- Shows currently executing operations
- Filter with query parameter for specific operations
- Identify slow queries with high millis
- Monitor lock types held by operations
- Use idleSession filter to find idle operations
- Monitor operation duration to detect hangs

### Q320: What are key MongoDB performance indicators?

- Throughput measured in operations per second
- Latency measured in response time milliseconds
- Cache hit ratio for working set fit
- Replication lag in seconds for secondaries
- Lock percentage for contention level
- Network throughput for bandwidth saturation

### Q321: How do I set up alerting for MongoDB metrics?

- Configure monitoring tool with thresholds
- Alert on high CPU or memory usage
- Alert on replication lag exceeding limit
- Alert on slow query count threshold
- Alert on connection count approaching limit
- Test alert delivery before production

### Q322: What is the $indexStats aggregation stage?

- Returns statistics for indexes on collection
- Displays accesses, ops, host for each index
- Identifies unused indexes
- Monitors index usage trends
- Replaces system.indexes collection
- Use to optimize index configuration

### Q323: How do I monitor query execution with profiler?

- Enable profiling with db.setProfilingLevel()
- Level 1 logs slow queries exceeding threshold
- Level 2 logs all queries
- Query system.profile collection for details
- Analyze execution stats to identify optimization
- Disable profiling when not needed

### Q324: What is the explain() method and its output?

- Displays query execution plan
- Shows stages traversed during execution
- Reports docsExamined and docsReturned
- Indicates if index used
- Displays executionTimeMillis
- Use to optimize queries

### Q325: How do I identify slow queries with explain?

- Check executionStages.stage for COLLSCAN
- High ratio of docsExamined to docsReturned
- Execution time in seconds indicates slowness
- Look for multiple stages indicating inefficiency
- Create index if full scan detected
- Re-test with explain after index creation

---

## 25. TROUBLESHOOTING ADVANCED SCENARIOS

### Q326: How do I recover from corrupted WiredTiger data files?

- WiredTiger cannot automatically repair data files
- Restore from known good backup
- Alternatively perform initial sync from replica
- Investigate corruption cause
- Verify backup integrity before restore
- Document corruption incident

### Q327: How do I handle excessive disk I/O on MongoDB?

- Monitor disk operations with iostat
- Identify hot collections with mongotop
- Check query execution plans
- Add missing indexes to reduce scans
- Monitor write throughput
- Consider hardware upgrade if sustained high I/O

### Q328: What causes uneven CPU usage across cores?

- MongoDB primarily single-threaded for operations
- Some background threads use other cores
- High CPU on single core indicates hot query
- Profile queries to identify problematic ones
- Optimize queries or add indexes
- Monitor CPU distribution

### Q329: How do I debug network connectivity issues?

- Use netstat to verify listening ports
- Use ping to test connectivity
- Check firewall rules allow MongoDB port
- Verify DNS resolution if using hostnames
- Monitor network latency with ping
- Check network hardware for issues

### Q330: How do I identify and fix lock contention?

- Monitor lock percentage in mongostat
- Query db.serverStatus() for lock statistics
- Identify operations holding locks
- Optimize transaction duration
- Reduce write concurrency if possible
- Monitor after optimization

### Q331: How do I handle cache eviction storms?

- Cache eviction occurs when memory pressure high
- Indicates working set exceeds available memory
- Monitor eviction rate with db.serverStatus()
- Add memory if possible
- Reduce working set by archiving data
- Optimize queries to reduce data touched

### Q332: How do I recover from capped collection corruption?

- Capped collections cannot be repaired
- Drop and recreate collection
- Restore from backup if needed
- Re-insert data after recreation
- Verify data completeness
- Use regular collections instead if issues persist

### Q333: How do I debug authentication failures?

- Check credentials in connection string
- Verify user exists in target database
- Confirm user has required roles
- Check server logs for auth errors
- Enable debug logging for more details
- Test with known working credentials

### Q334: How do I handle connection pool exhaustion?

- Increase maxPoolSize in connection string
- Monitor active connections
- Implement connection pooling in application
- Close unused connections
- Check for connection leaks in code
- Set appropriate connection timeout

### Q335: How do I recover from index corruption?

- Drop corrupted index
- Recreate index from scratch
- Monitor build process for errors
- Verify index functionality after recreation
- Run repair against collection if needed
- Test queries using index

### Q336: How do I identify memory leaks in application driver?

- Monitor application memory growth over time
- Check for open connections not being closed
- Verify sessions closed after use
- Look for accumulating data structures
- Monitor MongoDB cursor usage
- Test with garbage collection forced

### Q337: How do I handle unresponsive replica set elections?

- Verify majority of members available
- Check network connectivity between members
- Monitor heartbeat communication
- Increase heartbeat timeout if network issues
- Manually trigger election if needed
- Investigate failed member

### Q338: What causes oplog holes?

- Oplog holes prevent point-in-time recovery
- Occur when member crashes during write
- Restart member cleanly to prevent holes
- Use filesystem snapshots for reliable recovery
- Implement oplog hole detection
- Monitor backup integrity

### Q339: How do I debug slow aggregation pipeline?

- Check each stage with explain()
- Use $match first to reduce documents
- Monitor stage duration with profiler
- Look for sorts on non-indexed fields
- Enable allowDiskUse for large aggregations
- Test in staging before production

### Q340: How do I recover from cluster metadata corruption?

- Config servers maintain cluster metadata
- Corruption prevents sharded cluster operation
- Restore config server from backup
- Verify metadata consistency across replicas
- Monitor cluster health after restoration
- Implement preventive backup strategy

---

## 26. SECURITY & COMPLIANCE

### Q341: How do I implement role-based access control (RBAC)?

- Define roles with specific privileges
- Assign roles to users for database access
- Use built-in roles for common scenarios
- Create custom roles for specific needs
- Test role permissions before production
- Document role assignment rationale

### Q342: What is field-level encryption in MongoDB?

- Encrypt sensitive fields at application level
- Client-side encryption before sending to MongoDB
- Server stores encrypted data without keys
- Only authorized clients can decrypt data
- Implement using MongoDB driver libraries
- Balance security against query limitations

### Q343: How do I handle encryption key rotation?

- Store keys in key management service
- Rotate keys without downtime
- Re-encrypt affected documents
- Maintain key versioning
- Monitor key usage
- Document rotation procedures

### Q344: How do I audit user activities?

- Enable audit log with security.auditLog
- Capture authentication and authorization events
- Store audit logs separately for security
- Analyze logs for suspicious activity
- Implement log retention policy
- Review logs regularly for compliance

### Q345: What are MongoDB compliance requirements?

- HIPAA for healthcare data
- PCI-DSS for payment card data
- GDPR for EU resident data
- SOC 2 for service organizations
- Implement features needed for compliance
- Regular compliance audits

### Q346: How do I handle data subject access requests (DSAR)?

- GDPR and CCPA require responding to data requests
- Identify all data for specific subject
- Extract in accessible format
- Implement process for handling requests
- Set timelines for response (typically 30 days)
- Document requests and responses

### Q347: How do I implement right to be forgotten?

- Identify and delete all data for subject
- Update indexes if data deleted
- Verify deletion with queries
- Remove from backups if kept
- Implement retention policies
- Document deletion procedures

### Q348: How do I secure sensitive data at rest?

- Use encryption at rest if available
- Implement file system encryption
- Restrict file access permissions
- Store encryption keys separately
- Monitor file access
- Verify encryption status

### Q349: How do I secure data in transit?

- Enable TLS encryption for connections
- Use strong cipher suites
- Implement certificate management
- Monitor for unencrypted traffic
- Validate server certificates
- Update certificates before expiration

### Q350: How do I implement secrets management?

- Store credentials in vault like HashiCorp Vault
- Use environment variables for secrets
- Never hardcode credentials in code
- Rotate secrets regularly
- Audit secret access
- Implement least privilege for secret access

---

## 27. SCALABILITY & OPTIMIZATION

### Q351: When should I implement sharding?

- Sharding needed when single server insufficient
- Data size exceeds server storage capacity
- Throughput exceeds server processing capacity
- Data growth rate unsustainable on single server
- Choose shard key before implementing
- Plan for shard rebalancing

### Q352: What are shard key anti-patterns?

- Monotonically increasing values cause hotspots
- Low cardinality fields create uneven distribution
- Non-existent fields cause null chunk
- Using _id as sole shard key
- Poor shard key difficult to change later
- Test shard key distribution before production

### Q353: How do I handle shard key exhaustion?

- Shard key value range exhausted over time
- Reshard with new shard key
- Choose shard key with higher cardinality
- Test new key distribution
- Monitor shard key values trending
- Plan for future growth

### Q354: How do I optimize write throughput?

- Use bulk operations to reduce round trips
- Batch inserts and updates
- Optimize indexes for write performance
- Reduce index count if write-heavy
- Use appropriate write concern
- Monitor write latency

### Q355: How do I optimize read throughput?

- Use read preference to distribute reads
- Read from secondaries to reduce primary load
- Implement caching in application layer
- Create appropriate indexes
- Reduce projection to needed fields only
- Monitor read latency

### Q356: How do I scale to multiple datacenters?

- Deploy replica set across datacenters
- Set priority values for primary location
- Configure read preference for locality
- Monitor replication lag between locations
- Test failover to secondary datacenter
- Plan network bandwidth requirements

### Q357: How do I implement connection pooling?

- Configure maxPoolSize in connection string
- Reuse connections to reduce overhead
- Monitor connection pool efficiency
- Set appropriate timeout values
- Test pool behavior under load
- Tune pool size for application

### Q358: How do I handle thundering herd problem?

- Multiple clients retry simultaneously after failure
- Implement exponential backoff with jitter
- Spread retry attempts over time
- Monitor retry patterns
- Test retry logic under failure
- Document retry strategy

### Q359: How do I implement circuit breaker pattern?

- Stop retrying after repeated failures
- Wait before attempting to reconnect
- Gradually increase wait time
- Implement half-open state to test recovery
- Monitor circuit state
- Alert when circuit broken

### Q360: How do I optimize memory usage on client?

- Limit cursor batch size
- Implement pagination for large result sets
- Close cursors when complete
- Monitor client process memory
- Use streaming for large result sets
- Profile memory usage patterns

---

## 28. ENTERPRISE FEATURES & INTEGRATION

### Q361: What is MongoDB Atlas and when to use it?

- MongoDB Atlas is fully managed cloud database service
- Eliminates operational overhead of self-managed MongoDB
- Provides automated backups and restore
- Handles patching and upgrades automatically
- Provides high availability with replica sets
- Alternative to self-managed deployment

### Q362: How do I migrate to MongoDB Atlas?

- Create Atlas cluster with same configuration
- Use mongodump/mongorestore or Database Migration Service
- Test connectivity from application
- Perform parallel run with dual writes
- Cut over to Atlas when confident
- Monitor Atlas cluster after migration

### Q363: What is MongoDB Ops Manager?

- Ops Manager is on-premises MongoDB management tool
- Provides monitoring and backup for self-managed MongoDB
- Includes automation for deployment
- Available in MongoDB Enterprise Advanced
- Alternative to Atlas for on-premises deployments
- Requires separate installation and licensing

### Q364: How do I implement MongoDB Connector for Spark?

- Connector enables Spark to read/write MongoDB data
- Use for distributed data processing
- Implements partitioning for parallel processing
- Convert MongoDB documents to Spark DataFrames
- Support for complex queries and aggregations
- Useful for analytics on MongoDB data

### Q365: What is MongoDB Charts?

- Charts provides visualization of MongoDB data
- Creates dashboards from collections
- Supports various chart types
- Refresh data automatically
- Share dashboards with users
- Analyze MongoDB data visually

### Q366: How do I integrate MongoDB with Apache Kafka?

- Kafka Source Connector captures MongoDB changes
- Publish changes to Kafka topics
- Enables real-time streaming architectures
- Use change streams for change capture
- Implement downstream consumers
- Monitor pipeline latency

### Q367: What is MongoDB Realm?

- Realm is backend-as-a-service platform
- Handles authentication and authorization
- Sync data between devices and MongoDB
- Serverless functions for business logic
- Offline-first synchronization
- Alternative to building custom backend

### Q368: How do I use MongoDB Stitch (now Realm)?

- Stitch is predecessor to Realm
- Serverless functions without infrastructure
- Trigger functions on database events
- Schedule periodic tasks
- Deploy functions to Edge Server for processing
- Monitor function execution

### Q369: What is MongoDB Search (now Atlas Search)?

- Full-text search with advanced capabilities
- More powerful than native text indexes
- Dynamic field mapping for schema flexibility
- Supports faceting, synonyms, and highlights
- Integrated into MongoDB Atlas
- Better for complex search scenarios

### Q370: How do I implement MongoDB with GraphQL?

- Connect GraphQL server to MongoDB
- Map GraphQL types to MongoDB collections
- Implement resolvers for database queries
- Support mutations for write operations
- Use MongoDB aggregation framework
- Monitor GraphQL query performance

---

## 29. DISASTER RECOVERY ADVANCED SCENARIOS

### Q371: How do I prepare for ransomware attack recovery?

- Maintain offline backups not accessible to network
- Implement immutable backups with retention lock
- Test restoration regularly
- Monitor for suspicious access patterns
- Implement access controls restricting deletion
- Document recovery procedures

### Q372: What is ransomware detection in MongoDB?

- Monitor unusual deletion patterns
- Alert on unexpected database size reduction
- Track collection drop operations
- Monitor for encryption of data files
- Implement audit logging of all operations
- Test detection alerting mechanisms

### Q373: How do I recover from ransomware attack?

- Isolate affected systems immediately
- Identify last clean backup
- Restore from clean backup to isolated environment
- Scan for malware before reconnection
- Verify data integrity after restore
- Implement preventive measures

### Q374: How do I prepare for earthquake/natural disaster?

- Geographic diversity across regions
- Backup storage in different geographic areas
- Regular disaster recovery drills
- Documented recovery procedures
- Communication plan for team
- Alternative site for operations

### Q375: How do I handle multi-region failure?

- Requires backups in multiple geographic regions
- Implement manual failover procedures
- Test failover before disaster
- Document RTO and RPO for business
- Maintain contact list for key personnel
- Implement communication mechanisms

### Q376: How do I backup MongoDB to cloud storage?

- Use mongodump and upload to S3 or Azure Blob
- Implement lifecycle policies for retention
- Encrypt backups in transit and at rest
- Verify backup integrity periodically
- Implement cost optimization with storage classes
- Monitor backup storage costs

### Q377: How do I implement cross-region backup replication?

- Backup to one region, replicate to others
- Implement automated replication
- Verify backup integrity after replication
- Test restore from replicated backup
- Monitor replication latency
- Implement retention policies

### Q378: How do I document disaster recovery plan?

- Document RTO and RPO requirements
- Create runbook for each disaster scenario
- Include contact information for key personnel
- Document recovery procedures step-by-step
- Include validation procedures
- Update documentation regularly

### Q379: How do I test disaster recovery plan?

- Schedule quarterly DR drills
- Simulate various failure scenarios
- Practice recovery from backups
- Measure actual RTO and compare to SLA
- Document lessons learned
- Update procedures based on findings

### Q380: How do I calculate MongoDB disaster recovery costs?

- Account for backup storage costs
- Factor in personnel time for testing
- Include alternate infrastructure costs
- Monitor actual costs versus budget
- Optimize storage to reduce costs
- Balance cost against recovery requirements

---

## 30. FINAL OPERATIONAL SCENARIOS

### Q381: How do I handle MongoDB version compatibility?

- Check compatibility matrix before upgrade
- Older clients may not work with newer servers
- Newer drivers may not work with older servers
- Plan driver upgrades alongside server upgrade
- Test compatibility in staging environment
- Document minimum version requirements

### Q382: How do I implement feature flags in MongoDB?

- Store feature flags in collection
- Query flags on startup or per-operation
- Enable/disable features without code changes
- Implement gradual rollout of features
- Monitor feature flag performance impact
- Remove flags after full rollout

### Q383: How do I handle application backward compatibility?

- Support multiple MongoDB versions if possible
- Implement conditional logic based on version
- Test with minimum supported version
- Document version-specific behavior
- Plan version upgrade timeline
- Coordinate with operations team

### Q384: How do I implement data analytics pipelines?

- Use aggregation framework for analytics
- Export data to data warehouse for analysis
- Use MongoDB Connector for Spark
- Schedule regular analytical exports
- Monitor pipeline performance
- Implement data quality checks

### Q385: How do I handle multi-tenancy in MongoDB?

- Database-per-tenant for complete isolation
- Collection-per-tenant with field filtering
- Store tenant identifier in each document
- Implement Row-Level Security (RLS)
- Monitor resource usage per tenant
- Implement data access controls

### Q386: How do I secure multi-tenant deployments?

- Restrict data access per tenant
- Implement authentication per tenant
- Enforce authorization at query level
- Monitor for cross-tenant data leaks
- Implement audit logging per tenant
- Test security with adversarial queries

### Q387: How do I handle data retention policies?

- Implement TTL indexes for automatic deletion
- Configure expireAfterSeconds for retention period
- Archive old data before deletion
- Monitor deletion to ensure policies followed
- Test retention policy behavior
- Document retention requirements

### Q388: How do I implement audit trails?

- Capture all write operations with timestamps
- Store who, what, when, where information
- Audit collection for immutable records
- Verify audit data not modified
- Implement retention policy for audit data
- Generate audit reports for compliance

### Q389: How do I handle data versioning?

- Store version number with each document
- Track changes across versions
- Implement rollback to previous version
- Query specific version if needed
- Archive old versions for audit
- Monitor storage impact of versioning

### Q390: How do I implement soft delete?

- Use deleted flag instead of hard delete
- Mark documents as deleted
- Filter deleted documents in queries
- Restore deleted documents if needed
- Periodically purge old deleted documents
- Monitor storage impact

### Q391: How do I handle MongoDB in Kubernetes?

- Use StatefulSets for MongoDB deployment
- Configure persistent volumes for data
- Implement readiness probes for health
- Configure proper resource limits
- Use init containers for setup
- Monitor MongoDB in Kubernetes environment

### Q392: How do I implement MongoDB observability?

- Collect metrics with Prometheus or similar
- Implement distributed tracing for queries
- Log all operations for audit
- Set up dashboards for visualization
- Create alerts for anomalies
- Implement SLO monitoring

### Q393: How do I handle MongoDB upgrades safely?

- Test upgrade in staging environment
- Backup database before upgrade
- Perform upgrade on secondary nodes first
- Stepdown primary before upgrade
- Verify cluster health after each upgrade
- Prepare rollback plan

### Q394: How do I implement MongoDB high availability?

- Deploy 3 or more nodes in replica set
- Configure across multiple fault domains
- Implement automatic failover
- Monitor replica set health
- Test failover procedures
- Implement load balancing for readers

### Q395: How do I troubleshoot MongoDB performance regression?

- Compare performance metrics to baseline
- Identify changes since last good performance
- Check index usage and creation/removal
- Review query patterns and execution plans
- Profile slow operations with profiler
- Test optimization hypotheses

### Q396: How do I handle MongoDB licensing compliance?

- Track number of MongoDB instances
- Monitor usage to ensure compliance
- Document MongoDB versions deployed
- Implement license key management
- Perform periodic license audits
- Plan for license renewals

### Q397: How do I implement MongoDB cost optimization?

- Monitor actual versus planned resource usage
- Archive old data to reduce active dataset
- Optimize index configuration
- Implement connection pooling
- Right-size infrastructure
- Use cloud reserved instances for stable workloads

### Q398: How do I handle MongoDB documentation?

- Document architecture and deployment
- Create runbooks for common procedures
- Document troubleshooting procedures
- Maintain configuration management
- Update documentation with changes
- Share documentation with operations team

### Q399: How do I implement MongoDB training?

- Provide new team member onboarding
- Conduct operational procedure training
- Train on troubleshooting techniques
- Implement knowledge base
- Conduct regular review sessions
- Maintain training documentation

### Q400: How do I plan for MongoDB organizational scaling?

- Define clear roles and responsibilities
- Implement on-call rotation
- Document escalation procedures
- Plan for team growth
- Implement knowledge sharing mechanisms
- Build institutional knowledge

---

## 31. CUTTING-EDGE OPERATIONAL SCENARIOS

### Q401: How do I implement MongoDB vector search?

- Vector search enables semantic search
- Store embeddings with documents
- Create vector index for similarity search
- Query using $search aggregation stage
- Support for various vector distances
- Useful for RAG and AI applications

### Q402: How do I optimize MongoDB for machine learning?

- Export training data efficiently
- Implement feature engineering pipelines
- Monitor model serving performance
- Store model predictions in MongoDB
- Implement model versioning
- Track model performance over time

### Q403: How do I handle MongoDB with generative AI?

- Store training data and embeddings
- Implement prompt optimization
- Cache frequently used queries
- Monitor token usage for cost control
- Implement rate limiting for API calls
- Track AI-generated content audit trail

### Q404: How do I implement MongoDB for time series analytics?

- Time series collections optimize storage
- Use aggregation for analytics
- Implement downsampling for historical data
- Create dashboards with Charts
- Implement real-time alerting
- Monitor analytics query performance

### Q405: How do I handle MongoDB with edge computing?

- Sync data to edge devices
- Use Realm for offline-first apps
- Implement edge caching
- Handle conflict resolution
- Monitor edge device health
- Implement bandwidth optimization

### Q406: How do I use MongoDB for IoT data?

- Time series collections for sensor data
- Implement data compression
- Monitor storage growth
- Implement data aggregation pipelines
- Create real-time dashboards
- Archive old sensor data

### Q407: How do I implement MongoDB for financial services?

- ACID transactions for money transfers
- Encryption for payment data
- Audit trails for compliance
- Real-time settlement tracking
- Risk analytics with aggregation
- Implement PCI-DSS compliance

### Q408: How do I handle MongoDB for healthcare?

- HIPAA compliance implementation
- Patient data encryption
- Access control implementation
- Audit logging of all access
- Data retention policies
- Backup and disaster recovery

### Q409: How do I implement MongoDB for e-commerce?

- Inventory management system
- Order processing with transactions
- Shopping cart optimization
- Product catalog indexing
- User behavior analytics
- Fraud detection implementation

### Q410: How do I use MongoDB for social media platforms?

- User feed generation with aggregation
- Real-time notifications with change streams
- User graph implementation
- Comment and engagement tracking
- Media storage reference
- Implement sharding by user_id

### Q411: How do I handle MongoDB for content management?

- Document versioning system
- Workflow tracking
- Media asset storage
- Published and draft states
- Metadata indexing
- Content search implementation

### Q412: How do I implement MongoDB for gaming applications?

- Player profile storage
- Game state management
- Leaderboard implementation
- Transaction support for in-app purchases
- Real-time multiplayer data
- Achievement tracking

### Q413: How do I use MongoDB for messaging platforms?

- Message storage and retrieval
- Conversation threading
- User presence tracking
- Real-time notifications
- Message encryption
- Archive old messages

### Q414: How do I handle MongoDB for logistics tracking?

- Real-time shipment tracking
- Location history storage
- Geospatial queries for routing
- Delivery confirmation
- Integration with IoT devices
- Analytics for optimization

### Q415: How do I implement MongoDB for advertising technology?

- User segment management
- Campaign tracking
- Ad performance analytics
- Real-time bidding data
- Audience targeting
- Fraud detection

### Q416: How do I use MongoDB for smart city applications?

- Sensor data aggregation
- Real-time traffic monitoring
- Energy consumption tracking
- Environmental data collection
- Incident reporting system
- Analytics for urban planning

### Q417: How do I handle MongoDB for autonomous vehicles?

- Real-time telemetry storage
- Map and route data
- Trip history and analytics
- Vehicle state management
- Sensor data fusion
- High-frequency data ingestion

### Q418: How do I implement MongoDB for chatbot platforms?

- Conversation history storage
- Intent and entity tracking
- Model training data
- Response performance analytics
- User preference learning
- Integration with AI services

### Q419: How do I use MongoDB for recommendation engines?

- User preference tracking
- Item popularity metrics
- Collaborative filtering data
- Real-time personalization
- A/B testing framework
- Performance analytics

### Q420: How do I implement MongoDB for mobile applications?

- Offline-first sync with Realm
- Light-weight data storage
- Background synchronization
- Conflict resolution
- Push notification tracking
- App usage analytics

### Q421: How do I handle MongoDB for API gateways?

- Rate limiting data storage
- API key management
- Request/response logging
- Traffic analytics
- Cache layer implementation
- Performance monitoring

### Q422: How do I use MongoDB for data lakes?

- Heterogeneous data storage
- Schema-on-read capability
- Data cataloging
- Access control implementation
- Data quality monitoring
- Integration with analytics tools

### Q423: How do I implement MongoDB for master data management?

- Single source of truth
- Data quality rules
- Change tracking
- Reference data management
- Data stewardship workflows
- Integration with systems

### Q424: How do I handle MongoDB for customer data platform?

- Unified customer profiles
- Behavior tracking
- Segment building
- Real-time personalization
- Privacy compliance
- Data governance

### Q425: How do I use MongoDB for business intelligence?

- Data warehouse alternative
- OLAP cube implementation
- Dimensional modeling
- Drill-down analytics
- Self-service BI
- Performance optimization

### Q426: How do I implement MongoDB serverless?

- MongoDB Atlas serverless for auto-scaling
- No infrastructure management
- Pay-per-request pricing
- Automatic backups
- Connection pooling included
- Monitor serverless metrics

### Q427: How do I handle MongoDB federated queries?

- Query across MongoDB and external sources
- Implement $lookup with SQL data
- Join MongoDB collections with relational data
- Real-time data integration
- Performance considerations
- Security and access control

### Q428: How do I use MongoDB with data virtualization?

- Abstract MongoDB behind virtualization layer
- Support heterogeneous data sources
- Unified query interface
- Implement row-level security
- Performance optimization
- Caching strategies

### Q429: How do I implement MongoDB for regulatory compliance?

- GDPR right to be forgotten
- Data residency requirements
- Audit trail maintenance
- Encryption implementation
- Access control enforcement
- Compliance reporting

### Q430: How do I handle MongoDB for supply chain management?

- End-to-end visibility
- Real-time tracking
- Supplier data management
- Inventory analytics
- Risk monitoring
- Integration with systems

---

## 32. FUTURE-READY MONGODB OPERATIONS

### Q431: How do I prepare MongoDB for future growth?

- Monitor usage trends
- Plan capacity for next 12-24 months
- Evaluate sharding requirements
- Test scaling procedures
- Document growth projections
- Budget for expansion

### Q432: How do I evaluate new MongoDB features?

- Test features in staging environment
- Evaluate impact on architecture
- Consider backward compatibility
- Plan migration strategy
- Monitor for known issues
- Document feature usage

### Q433: How do I handle MongoDB technical debt?

- Identify legacy patterns
- Plan modernization gradually
- Refactor one area at a time
- Implement new patterns for new features
- Monitor for technical debt growth
- Allocate time for debt reduction

### Q434: How do I implement MongoDB observability strategy?

- Define key performance indicators
- Implement metrics collection
- Create dashboards for visibility
- Set up alerting for anomalies
- Implement tracing for debugging
- Document observability approach

### Q435: How do I handle MongoDB security updates?

- Subscribe to security advisories
- Test patches in staging first
- Plan update windows
- Execute updates systematically
- Verify security fixes applied
- Monitor for vulnerabilities

### Q436: How do I implement MongoDB governance framework?

- Define data ownership
- Establish naming conventions
- Implement access policies
- Create data classification scheme
- Document decision processes
- Regular governance reviews

### Q437: How do I handle MongoDB cloud migration strategy?

- Evaluate cloud providers
- Compare costs of on-premises vs cloud
- Plan migration timeline
- Implement networking setup
- Test connectivity and performance
- Execute phased migration

### Q438: How do I use MongoDB for internal tools?

- Build admin dashboards
- Create deployment tools
- Implement monitoring dashboards
- Build automation tools
- Document tool usage
- Support tool users

### Q439: How do I implement MongoDB knowledge management?

- Document best practices
- Create architecture patterns
- Build decision framework
- Maintain FAQ for common issues
- Document lessons learned
- Share knowledge across teams

### Q440: How do I prepare for MongoDB certification?

- Study MongoDB documentation
- Practice with hands-on labs
- Take practice exams
- Focus on weak areas
- Join study groups
- Schedule certification exam

### Q441: How do I handle MongoDB community involvement?

- Join MongoDB community forums
- Attend MongoDB conferences
- Contribute to open source
- Share experiences in blog posts
- Participate in local meetups
- Mentor junior developers

### Q442: How do I evaluate MongoDB alternatives?

- Compare with PostgreSQL, DynamoDB, etc.
- Evaluate feature sets
- Compare licensing and costs
- Test performance with actual workload
- Consider organizational skills
- Document evaluation findings

### Q443: How do I implement MongoDB along with relational databases?

- Polyglot persistence approach
- Use MongoDB for document-like data
- Use relational for structured data
- Implement data consistency mechanisms
- Manage transactions across systems
- Document data ownership

### Q444: How do I handle MongoDB performance tuning methodology?

- Establish baseline metrics
- Change one variable at a time
- Measure impact of changes
- Document findings
- Implement changes systematically
- Monitor for regressions

### Q445: How do I implement MongoDB disaster recovery testing framework?

- Schedule regular DR tests
- Simulate various failure scenarios
- Measure actual RTO and RPO
- Document findings
- Update procedures based on learnings
- Track improvements over time

### Q446: How do I create MongoDB automation framework?

- Infrastructure as code for deployment
- Automated backup scheduling
- Monitoring setup automation
- Index creation automation
- User provisioning automation
- Automated testing framework

### Q447: How do I implement MongoDB CI/CD integration?

- Run tests against MongoDB in pipeline
- Automated backup before deployments
- Schema validation in pipeline
- Performance testing in pipeline
- Automated rollback procedures
- Monitoring in pipeline

### Q448: How do I handle MongoDB troubleshooting framework?

- Decision tree for common issues
- Reference for known problems
- Step-by-step diagnostic procedures
- Automated diagnostic scripts
- Escalation procedures
- Knowledge base integration

### Q449: How do I implement MongoDB cost management strategy?

- Monitor per-environment costs
- Implement cost allocation
- Optimize resource utilization
- Right-size instances
- Use reserved instances for stable workloads
- Regular cost reviews

### Q450: How do I create MongoDB team development plan?

- Identify skill gaps
- Plan training programs
- Allocate time for learning
- Cross-training programs
- Certification support
- Career development planning

### Q451: How do I handle MongoDB version lifecycle management?

- Track version release cycles
- Plan upgrade timeline
- Test upgrades systematically
- Maintain multiple versions briefly
- Plan deprecation of old versions
- Communicate timelines to teams

### Q452: How do I implement MongoDB operations maturity model?

- Define maturity levels
- Assess current state
- Plan progression path
- Measure improvements
- Regular assessment cycles
- Benchmark against industry

### Q453: How do I create MongoDB runbooks library?

- Organize runbooks by topic
- Version control runbooks
- Regular review and updates
- Include troubleshooting steps
- Include escalation procedures
- Test runbooks regularly

### Q454: How do I implement MongoDB change management process?

- Define change categories
- Implement approval workflow
- Document change procedures
- Test changes in staging
- Schedule maintenance windows
- Communicate changes to teams

### Q455: How do I handle MongoDB incident management?

- Define incident severity levels
- Establish escalation procedures
- Create incident response playbooks
- Post-incident review process
- Document lessons learned
- Prevent recurrence

### Q456: How do I implement MongoDB SLO/SLA framework?

- Define service level indicators
- Set realistic SLO targets
- Monitor SLI metrics continuously
- Calculate SLA compliance
- Report on SLO achievement
- Plan improvements

### Q457: How do I handle MongoDB stakeholder communication?

- Regular status reports
- Explain technical issues in business terms
- Communicate planned maintenance
- Incident notifications
- Performance trend reports
- Budget and planning discussions

### Q458: How do I implement MongoDB best practices framework?

- Document architecture patterns
- Create design guidelines
- Establish coding standards
- Define operational procedures
- Review and update regularly
- Share across organization

### Q459: How do I handle MongoDB innovation and experimentation?

- Allocate time for experimentation
- Test new features safely
- Evaluate before production use
- Share learnings with team
- Implement successful innovations
- Document experimental outcomes

### Q460: How do I create MongoDB technology roadmap?

- Assess current infrastructure
- Identify future needs
- Prioritize improvements
- Plan implementation phases
- Budget allocation
- Regular roadmap reviews

---

# MongoDB Database Administration Complete FAQ Guide - Final Part
## Questions 461-500 - Strategic Implementation & Production Excellence

---

## 33. PERFORMANCE ANTI-PATTERNS & MITIGATION

### Q461: What is unbounded array anti-pattern?

- Arrays in documents growing without size limits
- Each update rewrites entire array causing performance degradation
- MongoDB rewrites document whenever array changes
- Index on array fields becomes inefficient with large arrays
- Split array field to separate collection if exceeds reasonable size
- Monitor array sizes in production documents

### Q462: How do I fix bloated document anti-pattern?

- Bloated documents exceed practical size and processing time
- Excessive embedded documents increase memory usage
- Move frequently accessed subset to separate collection
- Keep rarely accessed data in main document or archive
- Reference moved documents with foreign key pattern
- Test performance before and after de-bloating

### Q463: What causes COLLSCAN anti-pattern?

- Full collection scan without index usage
- Result of missing index or poorly designed queries
- Occurs with wildcard regex patterns at start
- $nin and $not operators often cause scans
- Missing $match stage early in aggregation pipeline
- Monitor explain() output for COLLSCAN detection

### Q464: How do I fix large unbounded result sets?

- Queries returning entire collection cause memory issues
- Implement pagination to limit result set size
- Use batch processing instead of findAll()
- Apply projection to reduce data size
- Set reasonable limits in queries
- Monitor largest queries for optimization

### Q465: What is unnecessary transaction anti-pattern?

- Using transactions when single-document update sufficient
- Transactions add latency and resource overhead
- Single-document writes always atomic in MongoDB
- Only use transactions for multi-document changes
- Avoid transactions for read-only operations
- Monitor transaction overhead versus benefit

### Q466: How do I avoid inefficient regex patterns?

- Regex with leading wildcard cannot use index
- Example /.*term/ forces full collection scan
- Use prefix matching when possible for index usage
- Example /^term/ uses index efficiently
- Text search better for complex pattern matching
- Test regex query performance with explain()

### Q467: What is missing index anti-pattern?

- High cardinality fields without indexes force scans
- user_id, email commonly missing indexes
- Impact increases with collection size
- Monitor slow queries to identify missing indexes
- Profile collections to find index gaps
- Create compound indexes for common query patterns

### Q468: How do I handle deeply nested document anti-pattern?

- MongoDB supports up to 128 levels of nesting
- Deep nesting makes queries complex and slow
- Accessing deeply nested fields requires field path
- Consider flattening or moving nested data
- Limit nesting to 3-4 levels for performance
- Balance normalization against query complexity

### Q469: What is cartesian product anti-pattern in $lookup?

- Multiple $lookup stages on same document
- Creates multiplicative result set size
- Example two $lookup stages triple result set
- Each join multiplies rows significantly
- Avoid multiple unrelated lookups in pipeline
- Denormalize data if many lookups needed

### Q470: How do I avoid unbounded $graphLookup?

- $graphLookup can traverse entire collection tree
- Unbounded traversal exhausts memory and time
- Set depthLimit to prevent excessive traversal
- Implement restrictive match conditions
- Monitor $graphLookup execution time
- Consider separate data model if complex graphs needed

### Q471: What is inefficient sorting anti-pattern?

- Sorting large result sets without index
- MongoDB must load all matching documents into memory
- Sorting on field without index forces SORT stage
- Create index on sort field if not indexed
- Place sort field last in compound index
- Test sort performance with explain()

### Q472: How do I handle connection pool exhaustion anti-pattern?

- Connections not returned to pool cause exhaustion
- Applications hang waiting for available connection
- Set appropriate connection timeout
- Monitor pool usage metrics
- Implement connection cleanup logic
- Configure maxPoolSize appropriately

### Q473: What causes cache eviction storms?

- Working set exceeds available RAM
- Cache evicts pages rapidly causing thrashing
- Read performance degrades severely
- Monitor eviction rate with serverStatus()
- Add memory if budget allows
- Archive data to reduce working set

### Q474: How do I avoid write amplification anti-pattern?

- Small writes causing large internal rewrites
- Multiple indexes on high-write fields multiply overhead
- Compression trades CPU for storage
- Each index consumes storage and write overhead
- Remove unused indexes aggressively
- Monitor write performance impact of indexes

### Q475: What is unnecessary data duplication anti-pattern?

- Storing same data in multiple collections
- Synchronization becomes maintenance nightmare
- Storage usage increases unnecessarily
- Queries now touch multiple collections
- Use references for relationships instead
- Document data ownership clearly

---

## 34. PRODUCTION READINESS CHECKLIST

### Q476: What is production readiness assessment for MongoDB?

- Comprehensive review before going live
- Verify all critical components configured
- Test failure scenarios and recovery procedures
- Confirm backup and monitoring setup
- Validate performance under expected load
- Document all operational procedures

### Q477: How do I verify cluster configuration readiness?

- Minimum 3 nodes for replica set with distributed locations
- Each node on separate physical machine
- Network connectivity verified between all nodes
- Firewall rules allow MongoDB communication
- Synchronization working correctly all directions
- Automatic failover tested and confirmed

### Q478: What security checks needed before production?

- Enable authentication and authorization
- Configure TLS for client and inter-node communication
- Implement role-based access control
- Create audit logging for compliance
- Test user access restrictions
- Document security procedures

### Q479: How do I verify backup and restore procedures?

- Backups running on schedule successfully
- Monthly restore testing validates backups
- Restore time meets RTO requirements
- Recovery point meets RPO requirements
- Backup storage verified multiple locations
- Document backup retention policy

### Q480: What performance baselines needed pre-production?

- Load test with expected peak workload
- Measure query latency at scale
- Verify cache hit ratio meets targets
- Confirm throughput meets requirements
- Test under sustained load for stability
- Document all baseline measurements

### Q481: How do I validate monitoring setup?

- All critical metrics being collected
- Alerts configured for threshold breaches
- Alert delivery verified working
- Dashboards show relevant metrics
- Log aggregation working correctly
- Test alerting system with synthetic events

### Q482: What application integration testing needed?

- Test all application query patterns
- Verify transaction handling works correctly
- Test error handling and retry logic
- Load test with realistic application traffic
- Verify connection pooling working correctly
- Test failover impact on application

### Q483: How do I prepare operations team?

- Provide comprehensive training on procedures
- Create runbooks for common tasks
- Define escalation procedures clearly
- Establish on-call rotation and responsibilities
- Document troubleshooting procedures
- Conduct dry runs of failure scenarios

### Q484: What compliance checks for production?

- Data residency requirements met
- Encryption at rest and in transit enabled
- Audit logging captures all access
- Retention policies match requirements
- Access controls limit who can modify data
- Document compliance measures

### Q485: How do I handle capacity planning pre-production?

- Estimate data growth over planning horizon
- Project throughput growth from business plan
- Allocate headroom for unexpected spikes
- Plan upgrade timeline for capacity
- Implement monitoring for capacity metrics
- Review plan quarterly with business

---

## 35. ORGANIZATIONAL SCALING & TEAM DYNAMICS

### Q486: How do I structure MongoDB DBA team?

- Senior DBAs for architecture and strategy
- Mid-level DBAs for day-to-day operations
- Junior DBAs for routine maintenance tasks
- On-call rotation covering 24x7 support
- Dedicated on-call engineer during incidents
- Clear escalation path for critical issues

### Q487: How do I implement knowledge transfer?

- Pair programming for complex procedures
- Brown bag lunch sessions sharing knowledge
- Documentation wiki for procedures
- Video recordings of walkthroughs
- Regular team retrospectives
- Mentoring program for junior staff

### Q488: How do I handle on-call responsibilities?

- Rotation schedule published in advance
- Clear incident classification and severity
- Escalation procedures documented
- Response time expectations defined
- Post-incident review process
- Compensation for on-call duty

### Q489: How do I implement change management process?

- Changes categorized by risk level
- Standard changes can follow expedited path
- Non-standard changes require approval
- Testing required before production
- Rollback plan documented
- Communication to affected teams

### Q490: How do I handle skill development?

- Budget for MongoDB training courses
- Conference attendance for latest knowledge
- Certification support and resources
- Internal skills assessment quarterly
- Career path planning for staff
- Mentoring and coaching programs

### Q491: How do I measure team effectiveness?

- Incident response metrics tracking
- Uptime percentage against SLA
- Mean time to resolution (MTTR) for incidents
- Change success rate without rollback
- Deployment frequency and reliability
- Knowledge retention through documentation

### Q492: How do I implement cross-training?

- Each procedure documented with primary and backup owner
- Regular rotation of responsibilities
- Pairing engineers on different shifts
- Shadowing program for complex tasks
- Cross-team training sessions
- Measure coverage gaps regularly

### Q493: How do I handle incident post-mortems?

- Blameless culture focusing on systems
- Document timeline of incident
- Root cause analysis identifying underlying cause
- Action items to prevent recurrence
- Follow-up tracking until closure
- Share learnings across organization

### Q494: How do I implement career progression?

- Clear levels with defined responsibilities
- Skill requirements for each level
- Demonstrated mastery in role before promotion
- Salary ranges based on level
- Regular performance discussions
- Development plan for growth

### Q495: How do I measure operational excellence?

- MTTR trending downward over time
- Fewer incidents from same root causes
- Change success rate above 95%
- Customer satisfaction scores
- Team confidence in procedures
- Reduced escalations to senior staff

---

## 36. STRATEGIC MONGODB IMPLEMENTATION

### Q496: How do I evaluate MongoDB for new projects?

- Assessment of data model fit for MongoDB
- Comparison with relational database requirements
- Evaluation of team expertise with MongoDB
- Cost analysis for infrastructure
- Timeline for project delivery
- Risk assessment for chosen approach

### Q497: How do I build business case for MongoDB migration?

- Current state analysis of existing system
- Pain points addressed by MongoDB
- Cost comparison of current vs MongoDB
- Timeline and resource requirements
- Risk analysis and mitigation
- ROI calculation and payback period

### Q498: How do I plan MongoDB rollout strategy?

- Phase 1 non-critical systems for learning
- Phase 2 less critical production systems
- Phase 3 core business systems
- Phase 4 migration of remaining systems
- Each phase reduces risk and builds expertise
- Regular assessment between phases

### Q499: How do I ensure MongoDB long-term success?

- Continuous monitoring and optimization
- Regular training for operations team
- Knowledge documentation and sharing
- Architecture review and modernization
- Technology assessment for features
- Partnership with MongoDB for support

### Q500: How do I measure MongoDB deployment success?

- Application performance meets or exceeds expectations
- Operational overhead reduced versus old system
- Team confident in managing MongoDB
- Customer satisfaction on system stability
- Cost per transaction reduced
- Scalability supports future growth

---

## FINAL RECOMMENDATIONS FOR MONGODB SUCCESS

### Key Takeaways for MongoDB DBA Excellence:

**1. Technology Foundation**
- Understand MongoDB architecture deeply
- Master replication, sharding, and transactions
- Implement proper monitoring and alerting
- Automate routine operational tasks
- Stay current with MongoDB releases

**2. Operational Excellence**
- Create comprehensive documentation
- Implement strong change management
- Build robust disaster recovery procedures
- Train teams thoroughly
- Practice incident response regularly

**3. Security & Compliance**
- Enforce authentication and authorization
- Implement encryption at rest and transit
- Maintain detailed audit logs
- Comply with regulatory requirements
- Review security regularly

**4. Performance & Optimization**
- Establish baseline performance metrics
- Monitor continuously for degradation
- Create indexes strategically
- Avoid common anti-patterns
- Test changes in staging first

**5. Reliability & Availability**
- Design for high availability
- Test failover procedures
- Implement geographic redundancy
- Backup and test recovery
- Monitor replication health

**6. Cost Management**
- Right-size infrastructure for workload
- Monitor and optimize resource usage
- Plan capacity for growth
- Use reserved instances for baseline
- Archive old data appropriately

**7. Organizational Success**
- Build skilled operations team
- Invest in training and development
- Foster knowledge sharing culture
- Implement clear escalation procedures
- Measure success metrics regularly

### Critical Success Factors:

1. Executive alignment on MongoDB strategy
2. Sufficient budget for proper implementation
3. Skilled team with MongoDB experience
4. Cultural commitment to operational excellence
5. Continuous learning and improvement mindset
6. Strong partnership with MongoDB support
7. Regular review and adjustment of practices

### Common Mistakes to Avoid:

1. Under-sizing infrastructure for growth
2. Inadequate backup and disaster recovery
3. Insufficient monitoring and alerting
4. Poor index strategy causing slow queries
5. Inadequate team training and documentation
6. Deploying to production without testing
7. Ignoring capacity planning
8. Inadequate security implementation
9. Lack of version upgrade planning
10. Insufficient post-incident learning

### Best Practices Summary:

- Always implement in replica sets for production
- Use sharding when data or throughput requires it
- Implement ACID transactions carefully for atomicity
- Monitor everything important to business
- Backup and test recovery regularly
- Document all procedures thoroughly
- Train team comprehensively
- Review and optimize continuously
- Plan for growth and scalability
- Implement security from beginning

---

# Conclusion

## Final Thoughts

This comprehensive MongoDB Database Administration FAQ serves as an enterprise reference covering the complete lifecycle of MongoDB administration—from initial deployment and configuration to performance optimization, security, monitoring, scalability, backup, disaster recovery, and advanced production troubleshooting.

While every effort has been made to align the content with official MongoDB documentation and industry best practices, all procedures should be validated in a non-production environment before implementation in production systems.

A well-managed MongoDB environment depends not only on technical expertise but also on disciplined operational processes, continuous monitoring, regular testing, and effective collaboration across infrastructure, development, security, and business teams.

---

# Key Responsibilities of a MongoDB DBA

A successful MongoDB Database Administrator is responsible for ensuring the following core objectives:

### 1. Availability

* Maintain 24×7 database availability
* Design highly available architectures
* Minimize downtime during maintenance and failures

### 2. Performance

* Optimize queries and indexes
* Monitor resource utilization
* Eliminate bottlenecks proactively

### 3. Reliability

* Implement robust backup strategies
* Regularly test disaster recovery procedures
* Continuously monitor database health

### 4. Security

* Protect sensitive business data
* Enforce authentication and authorization
* Implement encryption and compliance standards

### 5. Scalability

* Support business growth
* Design scalable replica sets and sharded clusters
* Optimize storage and compute resources

### 6. Cost Optimization

* Right-size infrastructure
* Optimize storage utilization
* Balance performance with operational costs

---

# What This Guide Covers

This guide contains **500 practical MongoDB administration questions** covering real-world enterprise scenarios across the entire MongoDB ecosystem.

Major areas include:

* MongoDB Architecture
* Installation & Deployment
* Replica Sets
* Sharding
* Transactions (ACID)
* Sessions
* CRUD Operations
* Aggregation Framework
* Indexing Strategies
* Query Optimization
* Performance Tuning
* Monitoring & Alerting
* Backup & Restore
* Disaster Recovery
* Security & Authentication
* Authorization & RBAC
* Auditing
* Data Validation
* Schema Design
* Change Streams
* Time Series Collections
* Geospatial Data
* Text Search
* Bulk Operations
* Storage Engine Management
* Memory Optimization
* Logging & Diagnostics
* Capacity Planning
* Operational Best Practices
* Enterprise Features
* Cloud Deployments
* Kubernetes Operations
* Compliance
* Advanced Troubleshooting
* Future-Ready Operations
* Industry-Specific Production Scenarios

---

# Success Factors for MongoDB Administration

Successful MongoDB administration requires more than technical knowledge.

An experienced DBA should:
* Understand business requirements before designing technical solutions
* Automate repetitive operational tasks
* Continuously monitor production environments
* Test backup and recovery procedures regularly
* Document standard operating procedures (SOPs)
* Maintain security and compliance
* Build scalable and resilient architectures
* Continuously upgrade skills as MongoDB evolves

The best MongoDB administrators balance technical excellence with practical business objectives, ensuring systems remain secure, reliable, performant, and cost-effective.

---

# Recommendations

To maximize the value of this guide:

* Practice every concept in a lab environment.
* Validate operational procedures before production deployment.
* Maintain documented backup and disaster recovery plans.
* Perform regular health checks and capacity assessments.
* Keep MongoDB deployments updated with supported releases.
* Review security configurations periodically.
* Automate routine administrative tasks whenever possible.
* Stay current with new MongoDB features and release notes.

---

# Official Reference

For the latest features, supported versions, release notes, and operational best practices, always refer to the official MongoDB documentation:

**MongoDB Documentation**

[https://www.mongodb.com/docs/](https://www.mongodb.com/docs/)

---

# Document Information

| Item                 | Details                                                  |
| -------------------- | -------------------------------------------------------- |
| **Document Title**   | Enterprise MongoDB Database Administration FAQ           |
| **Document Version** | 1.0                                                      |
| **Total Questions**  | 500                                                      |
| **Major Sections**   | 36                                                       |
| **Coverage**         | Beginner to Enterprise-Level Administration              |
| **Source Reference** | Official MongoDB Documentation & Industry Best Practices |
| **Last Updated**     | 30th July 2026 - Thursday                                |
| **Document Status**  | Production Ready                                         |
| **Drafted by**       | Praveen Madupu                                           |

---

# Closing Statement

This guide has been designed as a practical reference for MongoDB Database Administrators, DevOps Engineers, Database Engineers, Cloud Engineers, and IT Operations teams responsible for managing mission-critical MongoDB deployments.

The knowledge presented here provides a strong foundation for building reliable, secure, scalable, and high-performing MongoDB environments. 

As MongoDB continues to evolve, ongoing learning, hands-on practice, and operational discipline remain essential for long-term success.

> **"Great MongoDB administrators don't just manage databases—they build resilient platforms that enable business success."**
