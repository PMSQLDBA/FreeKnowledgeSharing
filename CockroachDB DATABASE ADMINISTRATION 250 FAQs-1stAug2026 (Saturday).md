CockroachDB DATABASE ADMINISTRATION 250 FAQs-1stAug2026

SECTION 1: CLUSTER SETUP AND INITIALIZATION


Q1: How do I start a single-node CockroachDB cluster for development?

1. Download the latest CockroachDB binary from binaries.cockroachdb.com for your OS.
2. Create data directory: mkdir -p data
3. Start node: cockroach start-single-node --insecure --listen-addr=localhost:26257 --http-addr=localhost:8080
4. Access DB Console at http://localhost:8080 in browser.
5. Connect via SQL: cockroach sql --insecure --host=localhost --port=26257
6. Verify cluster: SELECT * FROM crdb_internal.node_runtime_info;


Q2: What are prerequisites for multi-node CockroachDB production cluster?

1. Minimum 3 nodes for resilience with odd-numbered count for quorum.
2. Low-latency networking between nodes, typically in same datacenter/region.
3. Firewall rules: port 26257 for inter-node communication, 8080 for DB Console.
4. Shared storage or NFS mounts if using NFS for backup destinations.
5. Generate TLS certificates: cockroach cert create-ca --certs-dir=certs --ca-key=ca.key
6. Minimum 4GB RAM per node, 8GB+ recommended; fast SSD storage critical.


Q3: How do I initialize a secure 3-node cluster with TLS certificates?

1. Create directories: mkdir -p certs my-safe-directory
2. Create CA: cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
3. Create node certs: cockroach cert create-node --certs-dir=certs --ca-key=my-safe-directory/ca.key localhost node1.example.com node2.example.com node3.example.com
4. Create client cert: cockroach cert create-client --certs-dir=certs --ca-key=my-safe-directory/ca.key root
5. Start nodes: cockroach start --certs-dir=certs --listen-addr=node1.example.com:26257 --advertise-addr=node1.example.com:26257 --join=node1.example.com:26257,node2.example.com:26257,node3.example.com:26257
6. Cluster initialization completes automatically when quorum is reached.


Q4: What is difference between insecure mode and secure mode deployment?

1. Insecure mode: no TLS encryption, unauthenticated access, development only.
2. Secure mode: requires TLS certificates, enforces authentication, production-required.
3. Insecure: faster performance but no data encryption in transit.
4. Secure: encryption overhead justified by security benefits.
5. Production deployments must use secure mode per industry best practices.
6. Hybrid approach: insecure for testing, secure for production.


Q5: How do I configure cluster to use join token instead of explicit IP addresses?

1. Start first node: cockroach start-single-node outputs cluster ID and certificate.
2. Subsequent nodes use --join flag: cockroach start --join=existing-node:26257
3. Node automatically discovers cluster members via gossip protocol.
4. Useful for Kubernetes deployments where addresses unknown in advance.
5. Gossip port (26257) must be open between all nodes.
6. Join flag can point to any existing cluster node.


Q6: What should I consider when choosing between self-hosted and CockroachCloud?

1. Self-hosted: full infrastructure control, custom DR strategies, multi-region flexibility.
2. CockroachCloud: managed service, automated backups/upgrades, guaranteed uptime SLAs.
3. Self-hosted: infrastructure costs; CockroachCloud: consumption-based pricing.
4. CockroachCloud tiers: Basic (development), Standard (production), Advanced (mission-critical).
5. Self-hosted requires operational expertise; CockroachCloud reduces operational burden.
6. Choose based on team expertise, compliance requirements, and workload needs.


Q7: How do I set up CockroachDB on Kubernetes using CockroachDB Operator?

Script:
```bash
# Add Helm repo
helm repo add cockroachdb https://charts.cockroachdb.com

# Install operator
helm install my-release cockroachdb/cockroachdb-operator -n cockroachdb --create-namespace

# Create cluster manifest
cat <<EOF | kubectl apply -f -
apiVersion: cockroachdb.cockroachlabs.com/v1beta1
kind: CockroachDBCluster
metadata:
  name: cockroachdb
spec:
  nodes: 3
  image:
    name: cockroachdb/cockroach:latest
  storage:
    persistentVolume:
      size: 10Gi
  resources:
    requests:
      cpu: 2
      memory: 4Gi
EOF

# Access cluster
kubectl port-forward svc/cockroachdb 26257:26257 -n cockroachdb
```

1. Install Helm repo: helm repo add cockroachdb https://charts.cockroachdb.com
2. Operator automatically handles certificates, deployment, initialization.
3. Define CockroachDBCluster custom resource with node count, storage, resources.
4. Persistent volumes automatically bound to nodes.
5. Access via service: cockroachdb.default.svc.cluster.local
6. Monitor via DB Console through port-forward or ingress.


Q8: What are hardware requirements for production CockroachDB node?

1. CPU: minimum 4 cores, 8+ cores recommended for production.
2. RAM: minimum 4GB, 8GB+ recommended for query memory requirements.
3. Storage: fast SSD/NVMe critical; plan for 1-2x data size for WAL/snapshots.
4. Network: gigabit+ connectivity; low inter-node latency essential.
5. Disk I/O: at least 1000 IOPS per node for consistent performance.
6. Over-provision by 20-30% for unexpected growth and performance headroom.


Q9: How do I configure cluster locality awareness for multi-region deployments?

Syntax:
```bash
# Start node with locality
cockroach start --locality=region=us-east,zone=az1,rack=r1 --certs-dir=certs

# Configure zone for specific table
ALTER TABLE table_name CONFIGURE ZONE USING num_replicas=5, constraints='[+region=us-east]';

# View zone configuration
SELECT * FROM system.zones WHERE table_name='table_name';
```

1. Use --locality flag: cockroach start --locality=region=us-east,zone=az1
2. Locality information guides replica placement across zones.
3. Configure zone constraints: ALTER TABLE table CONFIGURE ZONE USING constraints='[+region=us-east]'
4. Monitor replica distribution through DB Console Ranges page.
5. Optimize data locality: place data where applications access it.
6. Monitor cross-region traffic to optimize costs and latency.


Q10: What happens during cluster initialization and how long does it take?

1. First node creates system tables: system.namespace, system.zones, system.settings.
2. Subsequent nodes replicate system data from existing nodes during bootstrap.
3. Cluster operational as soon as first node starts; nodes join gradually.
4. Full initialization typically takes 30-60 seconds depending on latency/disk speed.
5. Monitor progress through DB Console showing node status and replica distribution.
6. Until quorum available, some operations may have latency issues.


Q11: How do I migrate existing PostgreSQL database to CockroachDB?

Script:
```bash
# Export schema from PostgreSQL
pg_dump --schema-only mydb > schema.sql

# Import schema to CockroachDB
cockroach sql --certs-dir=certs < schema.sql

# Export data from PostgreSQL
pg_dump --data-only mydb > data.sql

# Import data to CockroachDB
cockroach sql --certs-dir=certs < data.sql

# Validate migration
cockroach sql --certs-dir=certs <<EOF
SELECT table_name, COUNT(*) as row_count 
FROM information_schema.tables t 
LEFT JOIN information_schema.key_column_usage k ON t.table_name = k.table_name 
GROUP BY table_name;
EOF
```

1. Export schema: pg_dump --schema-only mydb > schema.sql
2. Review for unsupported PostgreSQL features: triggers, inheritance.
3. Import schema: cockroach sql < schema.sql
4. Export data: pg_dump --data-only mydb > data.sql
5. Import data and validate consistency using row count checks.
6. Test application compatibility before production cutover.


Q12: What configuration changes recommended before going to production?

Syntax:
```bash
# Start with production configuration
cockroach start \
  --certs-dir=certs \
  --cache=2GB \
  --max-sql-memory=1GB \
  --max-offset=10ms \
  --log-dir=/var/log/cockroach \
  --store=type=ssd,path=/var/lib/cockroach

# Set cluster settings
cockroach sql --certs-dir=certs <<EOF
SET CLUSTER SETTING server.clock.forward_jump_check_enabled=true;
SET CLUSTER SETTING max_retry_delay='100ms';
SET CLUSTER SETTING kv.raft.command.max_size='64MiB';
EOF
```

1. Set cache-size: 25% of system RAM: --cache=2GB for 8GB node
2. Configure max-sql-memory: --max-sql-memory=1GB to limit query memory
3. Set max-offset for NTP clock tolerance: --max-offset=10ms
4. Enable cluster settings: SET CLUSTER SETTING server.clock.forward_jump_check_enabled=true
5. Configure logging: --log-dir=/var/log with appropriate retention
6. Set resource limits via Linux ulimits or container limits.


Q13: How do I verify cluster is properly initialized and healthy?

Syntax:
```bash
# Check node runtime info
SELECT * FROM crdb_internal.node_runtime_info;

# Verify system tables
SELECT * FROM system.zones;
SELECT * FROM system.node_liveness;

# Check replication status
SELECT * FROM crdb_internal.reports_meta;

# View cluster info
SELECT node_id, address, sql_addr FROM crdb_internal.nodes;
```

1. Query node runtime info: SELECT * FROM crdb_internal.node_runtime_info
2. Verify system tables initialized: SELECT * FROM system.zones
3. Check replication: SELECT * FROM crdb_internal.reports_meta
4. Access DB Console dashboard showing cluster health and metrics.
5. Run simple test: SELECT 1 to confirm SQL layer operational.
6. Check node liveness: SELECT * FROM system.node_liveness


Q14: What are common startup flags and their purposes?

1. --store: storage location, format: --store=type=ssd,path=/var/lib/cockroach
2. --listen-addr: SQL/gRPC address, default localhost:26257
3. --http-addr: DB Console server, default localhost:8080
4. --join: existing cluster nodes to join, comma-separated
5. --certs-dir: TLS certificates directory path
6. --insecure: disable TLS requirement, development only.


Q15: How do I set up load balancer in front of CockroachDB?

Script:
```bash
# HAProxy configuration
cat <<EOF > /etc/haproxy/haproxy.cfg
global
  maxconn 4096

frontend cockroachdb
  bind *:26257
  default_backend cockroachdb_servers

backend cockroachdb_servers
  balance roundrobin
  option tcp-check
  tcp-check connect port 26257
  server node1 192.168.1.10:26257 check
  server node2 192.168.1.11:26257 check
  server node3 192.168.1.12:26257 check
EOF

# Restart HAProxy
systemctl restart haproxy

# Verify load balancer
nc -zv loadbalancer 26257
```

1. Choose LB supporting TCP and connection pooling: HAProxy, AWS ELB, cloud-native.
2. Configure round-robin routing to all nodes on port 26257.
3. Set up health checks: TCP probes every 5-10 seconds.
4. Remove unhealthy nodes when health checks fail.
5. Use connection pooling in applications.
6. Monitor load balancer metrics for even distribution.


================================================================================
SECTION 2: NODE MANAGEMENT AND SCALING
================================================================================

Q16: How do I gracefully remove node from running cluster?

Script:
```bash
# Drain node to stop accepting new connections
cockroach node drain --certs-dir=certs --host=node1.example.com

# Stop node after drain completes
cockroach quit --certs-dir=certs --host=node1.example.com

# Verify all ranges moved to other nodes
cockroach sql --certs-dir=certs <<EOF
SELECT node_id, COUNT(*) as replica_count 
FROM crdb_internal.ranges_no_leases 
GROUP BY node_id 
ORDER BY node_id;
EOF
```

1. Drain node: cockroach node drain --host=node1.example.com
2. Verify drain completion: check logs for "drain complete"
3. Stop node: cockroach quit --host=node1.example.com
4. Cluster automatically moves ranges to healthy nodes.
5. Monitor health during rebalancing.
6. Remove from infrastructure once ranges moved.


Q17: What is range rebalancing and how does it work?

1. Automatic data redistribution across cluster nodes for load balance.
2. CockroachDB continuously evaluates range health and moves replicas.
3. Rebalancing considers replica count, zone constraints, locality.
4. Process happens gradually in background to avoid I/O overwhelming.
5. Monitor progress through DB Console Ranges section.
6. Disable temporarily if needed: SET CLUSTER SETTING kv.allocator.mode='off'


Q18: How do I add new node to existing cluster?

Script:
```bash
# Start new node with join flag
cockroach start \
  --certs-dir=certs \
  --listen-addr=node4.example.com:26257 \
  --advertise-addr=node4.example.com:26257 \
  --join=node1.example.com:26257 \
  --cache=2GB \
  --max-sql-memory=1GB

# Monitor node status
cockroach sql --certs-dir=certs <<EOF
SELECT node_id, address, is_live FROM crdb_internal.nodes ORDER BY node_id;
EOF

# Check rebalancing progress
SELECT node_id, COUNT(*) as ranges FROM crdb_internal.ranges_no_leases GROUP BY node_id;
```

1. Start new node with --join flag pointing to existing node.
2. Cluster discovers new node via gossip protocol.
3. Ranges automatically rebalance to new node.
4. Verify node appears healthy in DB Console.
5. Monitor replica distribution across all nodes.
6. Allow time for rebalancing to complete.


Q19: How do I identify and handle unbalanced range distribution?

Syntax:
```sql
-- Identify unbalanced nodes
SELECT node_id, COUNT(*) as replica_count 
FROM crdb_internal.ranges_no_leases 
GROUP BY node_id 
ORDER BY replica_count DESC;

-- Check zone configuration constraints
SELECT database_name, table_name, config 
FROM system.zones 
WHERE config LIKE '%constraints%';

-- Manually trigger rebalancing if needed
SET CLUSTER SETTING kv.allocator.mode='aggressive';
```

1. Query replica distribution: SELECT node_id, COUNT(*) FROM crdb_internal.ranges_no_leases GROUP BY node_id
2. Check DB Console Ranges page for visual replica distribution.
3. Imbalance usually indicates recently added node or constraint issues.
4. Increase rebalancing rate: SET CLUSTER SETTING kv.allocator.mode='aggressive'
5. Check constraints preventing automatic rebalancing.
6. Monitor network traffic during rebalancing.


Q20: What happens when node runs out of disk space?

1. Node enters read-only mode to prevent data corruption.
2. Other nodes continue normal operations if maintaining quorum.
3. Logs report disk space errors with clear messages.
4. Resolve by adding disk space or moving store to larger disk.
5. Node automatically resumes operations after space available.
6. Monitor disk usage proactively to prevent this situation.


Q21: How do I scale cluster horizontally to handle increased load?

Script:
```bash
# Identify bottleneck through monitoring
cockroach sql --certs-dir=certs <<EOF
-- Check CPU usage per node
SELECT node_id, crdb_internal.cpu_secs FROM crdb_internal.node_runtime_info;

-- Check ranges per node
SELECT node_id, COUNT(*) FROM crdb_internal.ranges_no_leases GROUP BY node_id;

-- Check disk usage
SELECT store_id, capacity_used, capacity_available 
FROM crdb_internal.stores;
EOF

# Add new nodes gradually
for i in 4 5 6; do
  cockroach start --certs-dir=certs --join=node1:26257 \
    --listen-addr=node$i:26257 --cache=2GB
done
```

1. Identify bottleneck: CPU, memory, or disk I/O through metrics.
2. Add nodes gradually rather than all at once.
3. For read-heavy: add nodes to spread load.
4. For write-heavy: ensure network bandwidth for consensus traffic.
5. Verify applications distribute load across new nodes.
6. Monitor rebalancing post-addition.


Q22: How do I configure and manage different store types on node?

Syntax:
```bash
# Start node with multiple stores
cockroach start \
  --store=type=ssd,path=/disk1 \
  --store=type=hdd,path=/disk2 \
  --certs-dir=certs

# Configure zone to use SSD for important tables
ALTER TABLE critical_table CONFIGURE ZONE USING 
  constraints='[+type=ssd]';
```

1. Use --store flag multiple times for multiple devices.
2. Format: --store=type=ssd,path=/disk1
3. CockroachDB distributes ranges across stores by type/capacity.
4. SSD stores get higher priority for hot ranges.
5. HDD stores handle less frequently accessed data.
6. Monitor store utilization and performance.


Q23: What is purpose of --attrs flag and how do I use it?

Syntax:
```bash
# Start node with attributes
cockroach start --attrs=flash:true,gpu:false --certs-dir=certs

# Query node attributes
SELECT node_id, attrs FROM crdb_internal.node_attributes;

# Place replicas on specific attribute
ALTER TABLE high_performance_table CONFIGURE ZONE USING 
  constraints='[+flash]';
```

1. Tag nodes with hardware capabilities: --attrs=flash:true
2. Use attributes in zone configurations: constraints='[+flash]'
3. Attributes describe node capabilities, not location.
4. Place critical tables on high-performance nodes.
5. Query attributes: SELECT * FROM system.node_attributes
6. Attributes different from locality.


Q24: How do I handle node that is permanently unavailable or failed?

1. Confirm node unavailability: check process, logs, connectivity.
2. Cluster automatically rebalances data to healthy nodes.
3. No intervention needed unless affects quorum.
4. If more than half nodes fail, restore from backup.
5. Remove from load balancer to prevent connection attempts.
6. Decommission once ranges fully rebalanced.


Q25: What should I consider when sizing cluster CPU requirements?

1. Baseline: 10-20% CPU at rest for system operations.
2. Each query thread uses approximately 1 core.
3. Replication/consensus consume CPU; higher replicas = more overhead.
4. Bulk operations spike CPU usage temporarily.
5. Monitor peak load; stay below 80% for headroom.
6. Auto-scale or add capacity when consistently hitting limits.


================================================================================
SECTION 3: DATA BACKUP AND RESTORE
================================================================================

Q26: What is difference between full and incremental backups?

1. Full backup: entire database state at specific timestamp, contains all data/schemas.
2. Incremental: only changes since last backup, uses less storage.
3. Full can restore independently; incremental requires preceding full.
4. Incremental completes faster, requires less bandwidth.
5. Retention policies auto-delete old backups.
6. Combine both for optimal recovery time and storage cost.


Q27: How do I create full backup to AWS S3?

Script:
```bash
# Create S3 bucket
aws s3 mb s3://my-backup-bucket

# Create full backup
cockroach sql --certs-dir=certs <<EOF
BACKUP INTO 's3://my-backup-bucket/backup-2024-01-15/?AWS_ACCESS_KEY_ID=key&AWS_SECRET_ACCESS_KEY=secret';
EOF

# Monitor backup progress
SELECT job_id, job_type, status, progress 
FROM system.jobs 
WHERE job_type = 'BACKUP' 
ORDER BY created DESC 
LIMIT 1;

# Verify backup
aws s3 ls s3://my-backup-bucket/backup-2024-01-15/
```

1. Create S3 bucket: aws s3 mb s3://my-backup-bucket
2. Set AWS credentials: environment variables or IAM roles.
3. Execute backup: BACKUP INTO 's3://bucket/backup-date/'
4. Monitor progress through system.jobs table.
5. Verify completion with non-empty S3 bucket object count.
6. Store credentials securely outside code.


Q28: How do I restore full backup from Google Cloud Storage?

Script:
```bash
# Create GCS bucket
gsutil mb gs://my-backup-bucket

# Restore from backup
cockroach sql --certs-dir=certs <<EOF
RESTORE FROM 'gs://my-backup-bucket/backup-2024-01-15/';
EOF

# Restore to specific point in time
RESTORE FROM 'gs://my-backup-bucket/backup-2024-01-15/' 
  AS OF SYSTEM TIME '2024-01-15 14:30:00';

# Verify restore
SELECT COUNT(*) as table_count FROM information_schema.tables;
```

1. Ensure GCS bucket permissions allow CockroachDB access.
2. Restore: RESTORE FROM 'gs://bucket/backup-path/'
3. Use RESTORE AS OF SYSTEM TIME for point-in-time recovery.
4. Cluster must be empty before restore.
5. Monitor restore progress through DB Console.
6. Validate restored data with sample queries.


Q29: How do I set up incremental backups with backup schedule?

Script:
```sql
-- Create full backup schedule
CREATE SCHEDULE full_backup FOR 
  BACKUP INTO 's3://bucket/full/?AWS_ACCESS_KEY_ID=key&AWS_SECRET_ACCESS_KEY=secret' 
  RECURRING EVERY 7 days 
  WITH RETENTION 30d;

-- Create incremental backup schedule
CREATE SCHEDULE incremental_backup FOR 
  BACKUP INTO 's3://bucket/incremental/?AWS_ACCESS_KEY_ID=key&AWS_SECRET_ACCESS_KEY=secret' 
  RECURRING EVERY 1 day 
  FULL BACKUP 'full_backup' 
  WITH RETENTION 7d;

-- Verify schedules
SELECT schedule_name, schedule_expr, next_run 
FROM system.scheduled_jobs 
WHERE schedule_name LIKE '%backup%';

-- Monitor backup metrics
SELECT schedules.BACKUP.failed, schedules.BACKUP.succeeded 
FROM system.scheduled_jobs;
```

1. Create full schedule: CREATE SCHEDULE full_backup FOR BACKUP INTO 's3://bucket/full' RECURRING EVERY 7 days
2. Create incremental: CREATE SCHEDULE incremental_backup FOR BACKUP INTO 's3://bucket/incremental' RECURRING EVERY 1 day
3. Scheduler automatically manages full/incremental sequence.
4. Verify with: SELECT * FROM system.scheduled_jobs
5. Monitor success: schedules.BACKUP.failed and schedules.BACKUP.succeeded
6. Adjust retention: ALTER SCHEDULE full_backup SET WITH RETENTION 30d


Q30: What is point-in-time recovery and how do I use it?

Script:
```sql
-- Enable revision history backup
BACKUP INTO 's3://bucket/backup' WITH revision_history 
  RECURRING EVERY 1 day 
  WITH RETENTION 30d;

-- Restore to specific point in time
RESTORE FROM 'gs://bucket/backup-2024-01-15/' 
  AS OF SYSTEM TIME '2024-01-15 14:30:00';

-- Restore specific database
RESTORE DATABASE mydb FROM 'gs://bucket/backup' 
  AS OF SYSTEM TIME '2024-01-15 12:00:00';

-- Restore specific table
RESTORE TABLE schema.table FROM 'gs://bucket/backup' 
  AS OF SYSTEM TIME '2024-01-15 10:00:00';
```

1. PITR allows restoring to any timestamp within backup window.
2. Use RESTORE ... AS OF SYSTEM TIME 'timestamp'
3. Revision history backups enable PITR.
4. Recovery point must fall within retention window.
5. Useful for recovering from accidental deletes/corruption.
6. Verify recovery point before incident time.


Q31: How do I restore only specific tables from backup?

Syntax:
```sql
-- Restore single table
RESTORE TABLE database.table FROM 'gs://bucket/backup-path';

-- Restore multiple tables
RESTORE TABLE database.table1, database.table2 
  FROM 'gs://bucket/backup';

-- Restore with rename
RESTORE TABLE database.old_table 
  AS new_table 
  FROM 'gs://bucket/backup';
```

1. Use RESTORE TABLE syntax: RESTORE TABLE database.table FROM 'gs://bucket/backup'
2. Restore multiple with comma separation.
3. Target tables must not already exist.
4. Foreign keys may require restoring related tables.
5. Monitor restore progress through DB Console.
6. Update sequences if tables use auto-increment.


Q32: How do I restore entire database from backup?

Syntax:
```sql
-- Restore full database
RESTORE DATABASE database_name FROM 'gs://bucket/backup-path';

-- Restore database with point-in-time
RESTORE DATABASE database_name FROM 'gs://bucket/backup' 
  AS OF SYSTEM TIME '2024-01-15 12:00:00';

-- Verify restore
SELECT table_name, row_count FROM information_schema.tables;
```

1. Use RESTORE DATABASE: RESTORE DATABASE db_name FROM 'gs://bucket/backup'
2. Database must not exist in target cluster.
3. Preserves schemas, indexes, constraints.
4. Foreign keys restored intact.
5. Takes longer than table restore.
6. Validate by checking table counts and sample data.


Q33: What should I do if backup job fails or is interrupted?

Script:
```sql
-- Check backup job status
SELECT job_id, job_type, status, created, description 
FROM system.jobs 
WHERE job_type = 'BACKUP' 
ORDER BY created DESC;

-- Check error details
SELECT job_id, error FROM system.jobs 
WHERE job_type = 'BACKUP' AND status = 'failed';

-- Cancel failed backup
CANCEL JOB job_id;

-- Re-run backup
BACKUP INTO 's3://bucket/backup-retry' 
  WITH revision_history;
```

1. Check status: SELECT * FROM system.jobs WHERE job_type = 'BACKUP'
2. Review error messages in jobs table.
3. Resolve permission issues: verify bucket access/credentials.
4. Cancel job: CANCEL JOB job_id
5. Re-run backup once issues resolved.
6. For incremental, re-run full backup if dependent fails.


Q34: How do I backup only specific databases to reduce storage?

Syntax:
```sql
-- Backup single database
BACKUP DATABASE important_db INTO 's3://bucket/important';

-- Backup with exclusions
BACKUP DATABASE mydb EXCEPT TABLE mydb.temp_table 
  INTO 's3://bucket/backup';

-- Create schedule for specific database
CREATE SCHEDULE db_backup FOR 
  BACKUP DATABASE important_db 
  INTO 's3://bucket/db-backups' 
  RECURRING EVERY 1 day 
  WITH RETENTION 7d;
```

1. Create database-specific schedule: BACKUP DATABASE important_db INTO 's3://bucket/important'
2. Use EXCEPT to exclude tables.
3. Multiple schedules for different databases.
4. Monitor storage costs by tracking backup sizes.
5. Different retention policies per database.
6. Consider business criticality when selecting backups.


Q35: How do I manage backup retention policies to control storage costs?

Syntax:
```sql
-- Set retention policy
CREATE SCHEDULE full_backup FOR BACKUP 
  INTO 's3://bucket/full' 
  RECURRING EVERY 7 days 
  WITH RETENTION 30d;

-- Modify retention
ALTER SCHEDULE full_backup SET WITH RETENTION 60d;

-- Implement tiered retention
CREATE SCHEDULE daily_backup FOR BACKUP 
  INTO 's3://bucket/daily' 
  RECURRING EVERY 1 day 
  WITH RETENTION 7d;

CREATE SCHEDULE weekly_backup FOR BACKUP 
  INTO 's3://bucket/weekly' 
  RECURRING EVERY 7 days 
  WITH RETENTION 30d;

CREATE SCHEDULE monthly_backup FOR BACKUP 
  INTO 's3://bucket/monthly' 
  RECURRING EVERY 30 days 
  WITH RETENTION 365d;
```

1. Retention syntax: WITH RETENTION 30d
2. More frequent backups with shorter retention reduce storage.
3. Calculate storage: daily_change_rate * retention_days
4. Implement tiered retention for cost optimization.
5. Monitor actual usage in cloud storage.
6. Balance retention with recovery requirements.


================================================================================
SECTION 4: DISASTER RECOVERY SCENARIOS
================================================================================

Q36: How do I recover from accidental deletion of critical data?

Script:
```sql
-- Stop applications immediately
-- Check garbage collection window
SELECT * FROM table_name AS OF SYSTEM TIME '2024-01-15 14:00:00';

-- Export recovered data
EXPORT (SELECT * FROM table_name AS OF SYSTEM TIME '2024-01-15 14:00:00')
  TO 'gs://bucket/recovery/table-backup.csv';

-- If past GC window, restore from backup
RESTORE TABLE table_name FROM 'gs://bucket/backup' 
  AS OF SYSTEM TIME '2024-01-15 12:00:00';

-- Merge recovered data
INSERT INTO table_name SELECT * FROM recovered_table;
```

1. Stop applications immediately to prevent further changes.
2. Check if data within garbage collection window (typically 24 hours).
3. Use AS OF SYSTEM TIME to query historical data.
4. Export recovered data for preservation.
5. If past GC, restore from backup and merge.
6. Implement restoration validation.


Q37: How do I recover from data corruption in table?

Script:
```sql
-- Identify corruption start time
SELECT timestamp, message FROM system.event_log 
  WHERE message LIKE '%corruption%' 
  ORDER BY timestamp DESC;

-- Query data before corruption
SELECT * FROM corrupted_table 
  AS OF SYSTEM TIME '2024-01-15 12:00:00' 
  LIMIT 100;

-- Create clean table with recovered data
CREATE TABLE clean_table AS 
  SELECT * FROM corrupted_table 
  AS OF SYSTEM TIME '2024-01-15 12:00:00';

-- Validate data
SELECT COUNT(*), COUNT(DISTINCT id) FROM clean_table;

-- Rename tables
ALTER TABLE corrupted_table RENAME TO corrupted_table_backup;
ALTER TABLE clean_table RENAME TO corrupted_table;
```

1. Identify corruption through error messages.
2. Determine start time from logs and query history.
3. Query historical data: SELECT ... AS OF SYSTEM TIME 'timestamp'
4. Create new clean table from validated data.
5. Rename corrupted table for investigation.
6. Resume operations with recovered table.


Q38: What should I do if majority of nodes fail simultaneously?

1. Cluster enters read-only mode as quorum unavailable.
2. Existing range leaders continue serving reads.
3. Assess failure scope and determine if hardware/network/software.
4. If recovery too long, initiate disaster recovery from backups.
5. Restore cluster from recent backup.
6. Redirect traffic to recovered cluster.


Q39: How do I handle total cluster loss scenario?

Script:
```bash
# Provision new infrastructure
# Provision at least 3 nodes similar to original

# Initialize new cluster
for i in 1 2 3; do
  cockroach start --certs-dir=certs \
    --join=node1:26257 \
    --listen-addr=node$i:26257 \
    --cache=2GB &
done

# Restore from most recent backup
cockroach sql --certs-dir=certs <<EOF
RESTORE FROM 'gs://bucket/backup-latest/';
EOF

# Validate restored data
SELECT COUNT(*) as table_count FROM information_schema.tables;
SELECT COUNT(*) as total_rows FROM (
  SELECT 1 FROM table1 
  UNION ALL SELECT 1 FROM table2
);
```

1. Declare cluster unrecoverable.
2. Provision new infrastructure matching original.
3. Select most recent backup.
4. Execute restore: RESTORE FROM 'gs://bucket/backup'
5. Validate data integrity and completeness.
6. Redirect application traffic.


Q40: How do I perform failover to standby cluster using physical replication?

Script:
```sql
-- Monitor replication status
SELECT status, replicated_time 
FROM crdb_internal.cluster_replication_status;

-- When failover needed, promote standby
ALTER CLUSTER SET REPLICATION FACTOR = 1;

-- Verify promoted cluster operational
SELECT * FROM crdb_internal.node_runtime_info;

-- Redirect application traffic to promoted cluster
-- Establish new backup chain and replication
```

1. Physical replication mirrors clusters at disk level.
2. Monitor status through crdb_internal.cluster_replication_status
3. Stop primary cluster from accepting writes when failing over.
4. Promote standby: ALTER CLUSTER SET REPLICATION FACTOR = 1
5. Redirect traffic to promoted cluster.
6. Establish new backup chain.


================================================================================
SECTION 5: USER MANAGEMENT AND SECURITY
================================================================================

Q41: How do I create new SQL user and assign basic privileges?

Script:
```sql
-- Create user with password
CREATE USER username WITH PASSWORD 'secure_password_123';

-- Verify user creation
SELECT * FROM system.users WHERE username = 'username';

-- Grant database privileges
GRANT ALL ON DATABASE database_name TO username;

-- Grant table-specific privileges
GRANT SELECT, INSERT ON database_name.table_name TO username;

-- Verify privileges
SHOW GRANTS FOR username;

-- Test user access
cockroach sql --user=username --host=localhost
```

1. Create user: CREATE USER username WITH PASSWORD 'password'
2. Verify: SELECT * FROM system.users
3. Grant database privileges: GRANT ALL ON DATABASE db TO user
4. Grant table privileges: GRANT SELECT, INSERT ON db.table TO user
5. Verify: SHOW GRANTS FOR username
6. Test by connecting with new user.


Q42: How do I change user password?

Syntax:
```sql
-- Change user password
ALTER USER username WITH PASSWORD 'new_password_123';

-- For self (current user)
ALTER USER current_user WITH PASSWORD 'new_password';

-- Force password change on next login (application-level)
UPDATE system.users SET password_expiration = now() 
  WHERE username = 'username';
```

1. Only user or admin can change passwords.
2. Syntax: ALTER USER username WITH PASSWORD 'new_password'
3. Use strong passwords: uppercase, lowercase, numbers, special chars.
4. Store securely outside code, don't include in history.
5. Implement rotation policy: 90-day for sensitive roles.
6. Audit password changes through logs.


Q43: How do I implement role-based access control (RBAC) for applications?

Script:
```sql
-- Create roles for different functions
CREATE ROLE read_only NOLOGIN;
CREATE ROLE read_write NOLOGIN;
CREATE ROLE admin_role NOLOGIN;

-- Grant privileges to roles
GRANT SELECT ON DATABASE mydb TO read_only;
GRANT SELECT, INSERT, UPDATE, DELETE ON DATABASE mydb TO read_write;
GRANT ALL ON DATABASE mydb TO admin_role;

-- Create application users
CREATE USER app_read WITH PASSWORD 'pass1';
CREATE USER app_write WITH PASSWORD 'pass2';
CREATE USER app_admin WITH PASSWORD 'pass3';

-- Assign users to roles
GRANT read_only TO app_read;
GRANT read_write TO app_write;
GRANT admin_role TO app_admin;

-- Verify role membership
SELECT * FROM system.role_members;
```

1. Create roles: CREATE ROLE role_name NOLOGIN
2. Assign privileges to roles: GRANT permissions ON db TO role
3. Create users for applications.
4. Add users to roles: GRANT role TO user
5. Different components use different users/roles.
6. Role membership checked for every statement.


Q44: How do I revoke privileges from user or role?

Syntax:
```sql
-- Revoke specific privilege
REVOKE SELECT ON database.table FROM username;

-- Revoke all privileges
REVOKE ALL ON database.table FROM username;

-- Revoke role membership
REVOKE role_name FROM username;

-- Verify revocation
SHOW GRANTS FOR username;
```

1. Use REVOKE: REVOKE SELECT ON db.table FROM user
2. Revoke all: REVOKE ALL ON db.table FROM user
3. Verify: SHOW GRANTS FOR username
4. Affects future operations; existing connections retain permissions.
5. Revoke role: REVOKE role_name FROM user
6. Test user can still perform required tasks.


Q45: What is difference between roles and users in CockroachDB?

1. Technically same entity with key difference.
2. Users: CREATE USER, can log in by default.
3. Roles: CREATE ROLE, NOLOGIN by default.
4. Roles serve as permission groups.
5. Multiple users can be members of role.
6. Use roles for grouping, users for individual access.


================================================================================
SECTION 6: MONITORING AND OBSERVABILITY
================================================================================

Q46: How do I set up Prometheus monitoring for CockroachDB cluster?

Script:
```bash
# prometheus.yml configuration
cat <<EOF > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: cockroachdb
    static_configs:
      - targets:
          - 'node1:8080'
          - 'node2:8080'
          - 'node3:8080'
    metrics_path: '/_status/vars'
EOF

# Restart Prometheus
systemctl restart prometheus

# Access Prometheus UI
# http://prometheus:9090
```

1. Each node exposes metrics at http://node:8080/_status/vars
2. Configure Prometheus scrape targets for each node.
3. Set scrape interval to 30 seconds for 4-minute resolution.
4. Define recording rules for frequently used aggregations.
5. Store at least 1-2 weeks metrics history.
6. Configure Alertmanager for notifications.


Q47: What are key CockroachDB metrics I should monitor?

Syntax:
```sql
-- Key metrics to track
sql.exec.latency.p99 > 1000ms -- Query latency
ranges.rebalancing.writes -- Replica movement
capacity.used / capacity.total > 0.85 -- Disk usage
raft.heartbeats.pending -- Consensus issues
sql.conn.open -- Active connections
sql.exec.success -- Successful queries per second
```

1. SQL latency: sql.exec.latency, alert on high values.
2. Replication: ranges.rebalancing.writes, expect periodic spikes.
3. Storage: capacity.used, capacity.available, alert before full.
4. Consensus: raft.heartbeats.pending, should be low.
5. Connections: sql.conn.open, diagnose pool exhaustion.
6. Throughput: sql.exec.success, capacity utilization.


Q48: How do I create alerting rules for critical cluster issues?

Script:
```yaml
# alerting rules (rules.yml)
groups:
  - name: cockroachdb
    rules:
      - alert: NodeDown
        expr: up{job="cockroachdb"} == 0
        for: 5m
        annotations:
          summary: "Node {{ $labels.instance }} is down"

      - alert: DiskSpaceWarning
        expr: capacity_used{job="cockroachdb"} / capacity_total > 0.85
        annotations:
          summary: "Disk usage {{ $value }}% on {{ $labels.instance }}"

      - alert: HighLatency
        expr: sql_exec_latency_p99{job="cockroachdb"} > 1000
        annotations:
          summary: "P99 latency {{ $value }}ms on {{ $labels.instance }}"

      - alert: BackupFailed
        expr: schedules_backup_failed > 0
        annotations:
          summary: "Backup failed on {{ $labels.instance }}"
```

1. Alert on node down: nodes.down > 0.
2. Alert disk space: capacity.used / capacity.total > 0.85
3. Alert high latency: sql.exec.latency.p99 > 1000ms
4. Alert failed jobs: jobs.failed > 0
5. Alert unavailable ranges: ranges.unavailable > 0
6. Alert process restarts: anomalies in restart rate.


Q49: How do I use DB Console for cluster monitoring?

1. Access at http://cluster-node:8080
2. Overview dashboard: cluster health, node status, metrics.
3. Ranges page: replica distribution, unbalanced nodes.
4. SQL dashboard: query latency, throughput, execution details.
5. Storage page: disk usage, range distribution by size.
6. Databases page: table sizes, key metrics per database.


Q50: How do I debug slow queries using query execution insights?

Script:
```sql
-- Access slowest queries
SELECT query, execution_count, latency_p99 
FROM crdb_internal.node_statement_statistics 
ORDER BY latency_p99 DESC 
LIMIT 10;

-- Analyze query plan
EXPLAIN SELECT * FROM large_table WHERE column = value;

-- Get actual execution stats
EXPLAIN ANALYZE SELECT * FROM large_table WHERE column = value;

-- Check for missing indexes
SELECT * FROM information_schema.statistics 
WHERE table_name = 'table_name';
```

1. Access Insights page showing long-running queries.
2. Use EXPLAIN to analyze query plan.
3. Use EXPLAIN ANALYZE for actual execution stats.
4. Look for sequential scans on large tables.
5. Monitor query latency trends.
6. Use statement statistics queries.


================================================================================
SECTION 7: PERFORMANCE TUNING
================================================================================

Q51: How do I identify and optimize slow queries?

Script:
```sql
-- Find slowest queries
SELECT query, execution_count, latency_p99, rows_read 
FROM crdb_internal.node_statement_statistics 
ORDER BY latency_p99 DESC 
LIMIT 10;

-- Analyze plan
EXPLAIN (VERBOSE) SELECT * FROM table WHERE column = value;

-- Check index usage
SELECT table_name, index_name, seq_scans, index_scans 
FROM crdb_internal.index_stats 
ORDER BY seq_scans DESC;

-- Create index for optimization
CREATE INDEX idx_column ON table(column);

-- Verify improvement
EXPLAIN SELECT * FROM table WHERE column = value;
```

1. Use EXPLAIN ANALYZE for execution plan and row statistics.
2. Look for sequential table scans on large tables.
3. Check for inefficient joins.
4. Monitor latency through metrics.
5. Use SHOW STATISTICS for column distribution accuracy.
6. Test changes in staging before production.


Q52: How do I create effective indexes for query performance?

Syntax:
```sql
-- Analyze WHERE clauses
SELECT * FROM table WHERE column1 = value AND column2 = value2;

-- Create single-column index
CREATE INDEX idx_column1 ON table(column1);

-- Create composite index
CREATE INDEX idx_columns ON table(column1, column2);

-- Create covering index (includes non-filter columns)
CREATE INDEX idx_covering ON table(column1) INCLUDE (column2, column3);

-- Check index usage
SELECT * FROM crdb_internal.index_stats 
WHERE table_name = 'table_name' 
ORDER BY seq_scans DESC;
```

1. Analyze WHERE clauses for frequent filters.
2. Create indexes on filtered columns.
3. Use composite indexes for multi-column filters.
4. Include non-filter columns for covering index optimization.
5. Monitor index usage to remove unused indexes.
6. Avoid excessive indexes that slow writes.


Q53: How do I tune cluster settings for write-heavy workloads?

Script:
```sql
-- Increase transaction worker threads
SET CLUSTER SETTING kv.allocator.mode='aggressive';

-- Monitor write throughput
SELECT node_id, ranges_writes_per_second 
FROM crdb_internal.node_metrics;

-- Tune raft tick interval
SET CLUSTER SETTING raft.tick_interval='100ms';

-- Increase checkpoint intervals
SET CLUSTER SETTING storage.rocksdb.block_cache_size_bytes='2GB';

-- Batch inserts
INSERT INTO table VALUES (val1), (val2), (val3), (val4), (val5);
```

1. Increase transaction worker threads through cluster settings.
2. Monitor ranges.writes_per_second for throughput.
3. Tune raft.tick_interval for replication responsiveness.
4. Increase checkpoint intervals to reduce write amplification.
5. Use batch inserts for better throughput.
6. Profile transaction commit latency.


Q54: How do I optimize read-heavy workloads?

Syntax:
```sql
-- Create indexes for fast lookup
CREATE INDEX idx_column ON table(column);

-- Use follower reads to scale reads
SET SESSION enable_follower_reads = true;

-- Application-level caching
-- Cache SELECT results in Redis/Memcached

-- Denormalize data for common queries
CREATE VIEW denormalized_view AS 
  SELECT t1.col1, t2.col2, t3.col3 
  FROM table1 t1 
  JOIN table2 t2 ON t1.id = t2.t1_id 
  JOIN table3 t3 ON t2.id = t3.t2_id;
```

1. Use indexes to avoid table scans.
2. Enable follower reads for local data.
3. Use application-level caching.
4. Denormalize strategically.
5. Batch multiple reads into single query.
6. Monitor index hit ratios.


Q55: How do I reduce query latency for time-sensitive operations?

Script:
```sql
-- Use read-only transactions
BEGIN READ ONLY;
SELECT * FROM table WHERE id = value;
COMMIT;

-- Use prepared statements for connection pooling
PREPARE stmt AS SELECT * FROM table WHERE id = $1;
EXECUTE stmt(value1);

-- Minimize transaction scope
BEGIN;
SELECT data INTO temp_var FROM table;
INSERT INTO results VALUES (temp_var);
COMMIT;

-- Tune isolation level if possible
SET default_transaction_isolation = 'READ UNCOMMITTED';
```

1. Use read-only transactions for query-only operations.
2. Implement application connection pooling.
3. Use local reads when data locality known.
4. Minimize network round trips.
5. Use prepared statements for plan caching.
6. Profile application vs query execution time.


================================================================================
SECTION 8: REPLICATION AND MULTI-REGION
================================================================================

Q56: How do I configure replication factor for high availability?

Script:
```sql
-- Set cluster default replication factor
SET CLUSTER SETTING num_replicas = 5;

-- Override per table
ALTER TABLE critical_table CONFIGURE ZONE USING num_replicas = 7;

-- Monitor replica count
SELECT node_id, COUNT(*) as replica_count 
FROM crdb_internal.ranges_no_leases 
GROUP BY node_id;

-- Verify zone configuration
SELECT database_name, table_name, config 
FROM system.zones;
```

1. Default: 3 replicas distributed across nodes.
2. Set cluster default: SET CLUSTER SETTING num_replicas = 5
3. Override per table: ALTER TABLE table CONFIGURE ZONE USING num_replicas = 7
4. Higher replicas: more fault tolerance, increased write latency.
5. Use odd numbers: 3, 5, 7 for proper quorum.
6. Monitor distribution through DB Console.


Q57: How do I set up multi-region cluster for data locality?

Script:
```bash
# Start nodes with locality info
cockroach start --locality=region=us-east,zone=az1 \
  --listen-addr=node1:26257 --certs-dir=certs

cockroach start --locality=region=us-west,zone=az1 \
  --listen-addr=node2:26257 --join=node1:26257 --certs-dir=certs

# Configure zone for regional data
cockroach sql --certs-dir=certs <<EOF
ALTER TABLE regional_table CONFIGURE ZONE USING 
  num_replicas = 5, 
  constraints = '[+region=us-east]';
EOF

# Monitor cross-region traffic
SELECT source_node_id, dest_node_id, packets_sent 
FROM crdb_internal.node_metrics;
```

1. Start nodes with locality: --locality=region=us-east,zone=az1
2. Replicas automatically distribute across regions.
3. Configure constraints: ALTER TABLE CONFIGURE ZONE USING constraints='[+region=us-east]'
4. Monitor replication traffic.
5. Use regional databases for specific tables.
6. Document failover procedures.


Q58: How do I implement cross-region data replication?

Syntax:
```sql
-- Physical cluster replication
ALTER CLUSTER SET REPLICATION FACTOR = 2;

-- Monitor replication lag
SELECT status, replicated_time 
FROM crdb_internal.cluster_replication_status;

-- Logical replication via changefeeds
CREATE CHANGEFEED FOR TABLE table_name 
INTO 'kafka://broker:9092/topic-name' 
WITH updated, resolved, format='avro', confluent_schema_registry='http://registry:8081';
```

1. Physical replication mirrors clusters at disk level.
2. Logical replication uses changefeeds for external systems.
3. Monitor status through crdb_internal.cluster_replication_status
4. Implement automatic failover.
5. Test failover procedures regularly.
6. Maintain backup copies in multiple regions.


Q59: How do I manage replica placement constraints for compliance?

Script:
```sql
-- Define region constraints
ALTER TABLE sensitive_data CONFIGURE ZONE USING 
  constraints = '[+region=eu]';

-- Verify constraint enforcement
SELECT table_name, config 
FROM system.zones 
WHERE config LIKE '%constraints%';

-- Test constraints
ALTER TABLE test_table CONFIGURE ZONE USING 
  constraints = '[+region=us-east, +region=us-west]';

-- Check actual replica distribution
SELECT range_id, node_id, store_id 
FROM crdb_internal.ranges 
WHERE table_name = 'sensitive_data';
```

1. Define constraints: ALTER TABLE CONFIGURE ZONE USING constraints='[+region=eu]'
2. Enforce data residency within specific regions.
3. Verify through DB Console Ranges view.
4. Test constraint violations by restarts.
5. Document compliance requirements.
6. Regular audits for adherence.


Q60: How do I optimize cross-region performance?

Script:
```sql
-- Use follower reads for local data access
SET SESSION enable_follower_reads = true;
SELECT * FROM table AS OF SYSTEM TIME follower_read_timestamp();

-- Monitor inter-region latency
SELECT source_node_id, dest_node_id, latency_ms 
FROM crdb_internal.node_metrics;

-- Batch operations to reduce round trips
INSERT INTO table VALUES (val1), (val2), (val3);

-- Place application servers in same region as database
-- Configure DNS to route to nearest region
```

1. Use follower reads for local replicas.
2. Minimize round trips via batching.
3. Place applications in same region.
4. Monitor inter-node latency.
5. Use read-only transactions without write latency.
6. Cache frequently accessed data locally.


================================================================================
SECTION 9: TROUBLESHOOTING AND MAINTENANCE
================================================================================

Q61: How do I diagnose connectivity issues between cluster nodes?

Script:
```bash
# Test basic connectivity
ping node2.example.com
telnet node2.example.com 26257

# Check firewall rules
netstat -tlnp | grep 26257
ss -tlnp | grep 26257

# Verify DNS resolution
nslookup node1.example.com
dig node1.example.com

# Check node logs
grep -i "connection" /path/to/cockroach.log

# Query node status
cockroach sql --certs-dir=certs <<EOF
SELECT node_id, address, sql_addr, is_live 
FROM crdb_internal.nodes;
EOF
```

1. Verify network connectivity: ping node_address
2. Test socket connectivity: telnet node_address 26257
3. Check firewall rules allow port 26257.
4. Review logs for connection errors.
5. Verify DNS resolution if using hostnames.
6. Use netstat/ss to check listening ports.


Q62: How do I troubleshoot cluster initialization failures?

Script:
```bash
# Check startup logs
tail -f /path/to/cockroach.log

# Look for specific errors
grep -i "fatal\|error" /path/to/cockroach.log

# Verify system time synchronization
ntpstat
timedatectl status

# Check disk space
df -h

# Verify certificate validity
openssl x509 -in certs/node.crt -noout -dates

# Start with verbose logging
cockroach start --certs-dir=certs --verbosity=2
```

1. Check error messages in logs immediately.
2. Verify system time synchronization across nodes.
3. Ensure disk space available for system tables.
4. Check certificate validity and permissions.
5. Verify ports 26257 and 8080 not in use.
6. Attempt single-node startup first.


Q63: How do I resolve "node_id not set" errors?

1. Indicates node cannot assign ID during startup.
2. Check node can connect to cluster via --join.
3. Verify network connectivity to join targets.
4. Verify DNS resolution if using hostnames.
5. Check system time synchronization.
6. Review logs for detailed error messages.


Q64: How do I fix "range does not exist" errors in queries?

1. Indicates range splits/rebalancing in progress.
2. Retry query; should succeed when rebalancing completes.
3. If persists, indicates data corruption or system issues.
4. Check cluster health: verify nodes operational.
5. Review system logs for corruption/leadership issues.
6. Contact support if error persists.


Q65: How do I troubleshoot "context deadline exceeded" errors?

Script:
```sql
-- Check query complexity
EXPLAIN SELECT * FROM large_table WHERE complex_condition;

-- Increase statement timeout if needed
SET statement_timeout = '5m';

-- Monitor cluster load
SELECT node_id, cpu_time_seconds, sys_cpu_time_seconds 
FROM crdb_internal.node_runtime_info;

-- Identify large queries
SELECT query, execution_count, latency_p99 
FROM crdb_internal.node_statement_statistics 
ORDER BY latency_p99 DESC;
```

1. Query execution exceeded timeout.
2. Check query complexity with EXPLAIN.
3. Increase timeout: SET statement_timeout = '5m'
4. Monitor cluster load; high load increases execution time.
5. Optimize query or add indexes.
6. Break large operations into smaller transactions.


================================================================================
SECTION 10: ADVANCED SCENARIOS
================================================================================

Q66: How do I implement schema versioning for zero-downtime migrations?

Script:
```sql
-- Add new column
ALTER TABLE table_name ADD COLUMN new_col type;

-- Backfill in batches
UPDATE table_name SET new_col = calculated_value 
  WHERE new_col IS NULL 
  LIMIT 1000;

-- Applications handle both columns during migration
-- SELECT old_col, new_col FROM table WHERE ...

-- Drop old column after migration
ALTER TABLE table_name DROP COLUMN old_col;

-- Test in staging first
```

1. Use online schema changes without table locks.
2. Add column: ALTER TABLE table_name ADD COLUMN new_col type
3. Backfill in batches: UPDATE ... LIMIT 1000
4. Applications support both columns during transition.
5. Drop old column post-migration.
6. Test before production deployment.


Q67: How do I handle extremely large tables (terabyte-scale)?

Script:
```sql
-- Partition large table by date
CREATE TABLE metrics (
  timestamp TIMESTAMP,
  value INT,
  PRIMARY KEY (timestamp)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01')
);

-- Archive old partitions
EXPORT (SELECT * FROM metrics WHERE timestamp < '2023-01-01')
  TO 'gs://bucket/archive/metrics-2022/';

DELETE FROM metrics WHERE timestamp < '2023-01-01';
```

1. Partition by date for time-series data.
2. Use PARTITION BY RANGE (date_column)
3. Create separate indexes per partition.
4. Archive old partitions to reduce storage.
5. Query only necessary partitions.
6. Monitor partition size growth.


Q68: How do I implement bulk delete operations without impacting cluster?

Script:
```sql
-- Batch deletion approach
DO $$
BEGIN
  LOOP
    DELETE FROM table_name 
      WHERE creation_date < '2023-01-01' 
      LIMIT 1000;
    IF ROW_COUNT = 0 THEN
      EXIT;
    END IF;
    -- Pause between batches
    SELECT pg_sleep(1);
  END LOOP;
END$$;

-- Alternative using TRUNCATE for entire table
TRUNCATE TABLE table_name;
```

1. Avoid large DELETE operations processing too many rows.
2. Batch: DELETE WHERE condition LIMIT 1000
3. Pause between batches to reduce load.
4. Monitor DELETE job progress.
5. Use TRUNCATE only for entire tables.
6. Test on staging data first.


Q69: How do I implement cross-database transactions for ACID consistency?

1. CockroachDB does not support cross-database transactions.
2. Implement application-level coordination.
3. Use two-phase commit pattern for cross-cluster.
4. Document transaction scope limitations.
5. Design databases to minimize cross-database needs.
6. Use event sourcing or saga patterns.


Q70: How do I optimize for specific query patterns in workload clusters?

Script:
```sql
-- Profile typical query patterns
SELECT query, execution_count, latency_p99 
FROM crdb_internal.node_statement_statistics 
ORDER BY execution_count DESC 
LIMIT 10;

-- Create indexes for dominant patterns
CREATE INDEX idx_pattern1 ON table(col1, col2);

-- Denormalize frequently joined tables
CREATE VIEW optimized_view AS 
  SELECT t1.col1, t2.col2, t3.col3 
  FROM table1 t1 
  JOIN table2 t2 ON t1.id = t2.t1_id 
  JOIN table3 t3 ON t2.id = t3.t2_id;
```

1. Profile query patterns in production.
2. Create indexes for dominant queries.
3. Denormalize frequently joined tables.
4. Use materialized views for aggregations.
5. Configure cluster settings based on workload.
6. Monitor post-optimization effectiveness.


================================================================================
SECTION 11: COCKROACHCLOUD ADMINISTRATION
================================================================================

Q71: How do I create CockroachCloud cluster for first time?

1. Visit https://cockroachlabs.cloud and sign up.
2. Create organization and select cloud provider (AWS or GCP) and region.
3. Choose cluster type: Basic (development), Standard (production), Advanced (mission-critical).
4. Configure node count, machine type, storage capacity.
5. Accept terms and initiate creation; provisioning takes 5-10 minutes.
6. Access connection string from cluster overview page.


Q72: What is difference between CockroachCloud cluster tiers?

1. Basic: consumption-based pricing, suitable for development.
2. Standard: pre-provisioned compute, predictable costs, production-ready.
3. Advanced: multi-region, higher SLAs, PCI compliance features.
4. Basic: lower minimum cost but less control.
5. Standard/Advanced: managed backups with configurable retention.
6. Choose based on criticality, SLA, compliance requirements.


Q73: How do I configure automated backups in CockroachCloud?

Script:
```bash
# Navigate to Backups section in cluster overview
# CockroachCloud automatically:
# - Takes daily full backups (retained 30 days)
# - Takes hourly incremental backups (retained 7 days)

# Customize retention through console or SQL
ALTER BACKUP SETTINGS SET retention = '60d';
```

1. Navigate to Backups section in cluster overview.
2. CockroachCloud takes daily full and hourly incremental backups.
3. Default retention: 30 days full, 7 days incremental.
4. Customize retention through console.
5. View backup list with timestamp, size, status.
6. Manage backup storage location for multi-region distribution.


Q74: How do I scale CockroachCloud cluster up or down?

1. Access cluster settings through console Cluster Overview.
2. Modify node count: add for capacity, remove for cost reduction.
3. Change machine type for performance adjustment.
4. Scaling completes without downtime; nodes join/leave gracefully.
5. Monitor scaling progress through console.
6. Costs adjust immediately based on new configuration.


Q75: How do I manage maintenance windows in CockroachCloud?

1. Set maintenance window in cluster settings.
2. Choose time during low traffic to minimize impact.
3. Single-node clusters experience downtime.
4. Multi-node clusters continue during rolling upgrades.
5. Defer patch upgrades to later date if inconvenient.
6. Monitor completion through console activity log.


================================================================================
SECTION 12: ADVANCED PERFORMANCE OPTIMIZATION
================================================================================

Q76: How do I implement prepared statements for improved performance?

Script:
```sql
-- Prepare statement on first execution
PREPARE stmt(INT, VARCHAR) AS 
  SELECT * FROM table WHERE id = $1 AND name = $2;

-- Execute with parameters
EXECUTE stmt(100, 'value');

-- In application (pseudocode)
// Java
String sql = "SELECT * FROM table WHERE id = ? AND name = ?";
PreparedStatement pstmt = connection.prepareStatement(sql);

// Python
cursor.execute("SELECT * FROM table WHERE id = %s AND name = %s", (100, 'value'))
```

1. Prepare: PREPARE stmt(types) AS SELECT ...
2. Execute: EXECUTE stmt(val1, val2)
3. Cached in connection; repeated execution reuses plan.
4. ORMs handle preparation automatically.
5. Monitor prepared statement statistics.
6. Implement statement pooling.


Q77: How do I optimize bulk insert performance?

Script:
```sql
-- Use IMPORT instead of INSERT
IMPORT DATA FROM 's3://bucket/data.csv' 
  INTO table_name 
  WITH delimiter=',', skip='1';

-- Use COPY for streaming data
\COPY table_name(col1, col2) FROM STDIN WITH (DELIMITER ',')

-- Batch inserts in transaction
BEGIN;
INSERT INTO table VALUES (val1), (val2), (val3);
INSERT INTO table VALUES (val4), (val5), (val6);
COMMIT;

-- Monitor insertion rate
SELECT * FROM system.jobs WHERE job_type = 'IMPORT';
```

1. Use COPY/IMPORT instead of INSERT.
2. Disable constraints during load; rebuild after.
3. Import into non-indexed tables first.
4. Use parallel imports across tables.
5. Monitor rate; expect thousands of rows/second.
6. Verify integrity immediately after.


Q78: How do I implement query result streaming for large datasets?

Script:
```sql
-- Client-side pagination
SELECT * FROM large_table LIMIT 1000 OFFSET 0;
SELECT * FROM large_table LIMIT 1000 OFFSET 1000;

-- Cursor-based pagination
SELECT * FROM large_table WHERE id > last_id ORDER BY id LIMIT 1000;

-- Application-level streaming
// Pseudocode
while (hasMore) {
  results = query("SELECT * FROM table LIMIT 1000 OFFSET " + offset);
  processResults(results);
  offset += 1000;
}
```

1. Use pagination with LIMIT and OFFSET.
2. Cursor-based: fetch after last-seen key.
3. Avoid loading entire result set into memory.
4. Monitor client memory usage.
5. Implement timeout for streaming queries.
6. Test streaming performance.


================================================================================
SECTION 13: ADVANCED DISASTER RECOVERY
================================================================================

Q79: How do I implement backup encryption with customer-managed keys?

Script:
```sql
-- Enable backup encryption with AWS KMS
BACKUP INTO 's3://bucket/backup' 
  WITH KMS_STORE='aws',
  AWS_KMS_KEY_ARN='arn:aws:kms:us-east-1:123456789:key/12345678-1234-1234-1234-123456789012';

-- Store encryption keys separately from backup storage
-- Decrypt during restore if authorized with key access

-- Monitor key usage through KMS provider audit logs
```

1. Configure KMS (AWS, Azure, GCP) with encryption key.
2. Store keys separately from backup storage.
3. Enable backup encryption: BACKUP ... WITH KMS_STORE='aws'
4. Decrypt during restore if authorized.
5. Monitor key usage through audit logs.
6. Implement key rotation policies.


Q80: How do I test backup restore procedures without affecting production?

Script:
```bash
# Restore to isolated test cluster
cockroach sql --certs-dir=certs <<EOF
RESTORE FROM 'gs://bucket/backup-2024-01-15/';
EOF

# Validate restored data
SELECT COUNT(*) as table_count FROM information_schema.tables;
SELECT COUNT(*) as total_rows FROM (
  SELECT 1 FROM table1 UNION ALL SELECT 1 FROM table2
);

# Measure restore duration
time (cockroach sql --certs-dir=certs <<EOF
RESTORE FROM 'gs://bucket/backup/';
EOF)

# Test application connectivity
psql -h localhost -U root -c "SELECT 1;"
```

1. Use backup copies to restore to separate cluster.
2. Validate against known queries.
3. Test application connectivity.
4. Measure RTO for disaster planning.
5. Document issues discovered.
6. Schedule regular testing (quarterly minimum).


================================================================================
SECTION 14: SECURITY HARDENING AND COMPLIANCE
================================================================================

Q81: How do I implement SSL/TLS certificate rotation without downtime?

Script:
```bash
# Generate new certificates
cockroach cert create-node \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key \
  node1.example.com node2.example.com node3.example.com

# Copy new certificates to nodes (cluster still running)
scp -r certs/* node1:/path/to/certs/
scp -r certs/* node2:/path/to/certs/
scp -r certs/* node3:/path/to/certs/

# Rolling node restarts for inter-node communication
for node in node1 node2 node3; do
  ssh $node "cockroach quit --certs-dir=certs && cockroach start --certs-dir=certs --join=node1"
  sleep 30
done

# Verify new certificates in use
openssl x509 -in certs/node.crt -noout -dates
```

1. Generate new certificates before expiration.
2. Copy to nodes while cluster running.
3. Nodes read certificates at connection time.
4. For inter-node: perform rolling restarts.
5. Monitor certificate usage.
6. Remove old certificates after rollover.


Q82: How do I implement secure password storage and rotation?

Script:
```sql
-- Create password rotation policy
CREATE TABLE password_history (
  user_id INT,
  password_hash VARCHAR,
  changed_at TIMESTAMP DEFAULT now()
);

-- Implement rotation
ALTER USER username WITH PASSWORD 'new_password_123';

-- Audit password changes
SELECT * FROM password_history WHERE user_id = user_id ORDER BY changed_at DESC;

-- Implement password expiration (application-level)
-- UPDATE users SET password_expires_at = now() + INTERVAL '90 days';
```

1. Passwords hashed using bcrypt; never plaintext.
2. Implement 90-day rotation for sensitive roles.
3. Use ALTER USER to change passwords.
4. Store in secure secret management (Vault, Secrets Manager).
5. Implement application enforcement.
6. Audit changes through logs.


Q83: How do I implement network segmentation with CockroachDB?

Script:
```bash
# Security group rules (example for AWS)
# Allow port 26257 from application servers only
aws ec2 authorize-security-group-ingress \
  --group-id sg-1234567 \
  --protocol tcp --port 26257 \
  --source-security-group-id sg-app-servers

# Restrict DB Console to admin networks
aws ec2 authorize-security-group-ingress \
  --group-id sg-1234567 \
  --protocol tcp --port 8080 \
  --cidr-ip 10.0.0.0/8  # Admin network only

# Use private subnets for database
# Prevent public internet access
```

1. Use VPC and security group rules.
2. Restrict port 26257 to known servers.
3. Restrict port 8080 to admin networks.
4. Use private subnets.
5. Implement load balancer for encryption/access control.
6. Monitor network traffic for anomalies.


Q84: How do I audit SQL statement execution for compliance?

Script:
```sql
-- Enable statement logging
SET CLUSTER SETTING sql.trace.log_statement_execute=true;

-- Forward logs to centralized system
-- (Configure logging output to syslog/remote collector)

-- Query audit logs
SELECT timestamp, user, statement 
FROM system.event_log 
WHERE event_type = 'sql'
ORDER BY timestamp DESC;

-- Monitor suspicious statements
SELECT * FROM system.event_log 
WHERE statement LIKE 'DROP%' OR statement LIKE 'TRUNCATE%'
ORDER BY timestamp DESC;
```

1. Enable statement logging: SET CLUSTER SETTING sql.trace.log_statement_execute=true
2. Capture all queries with user and timestamp.
3. Forward to centralized system.
4. Monitor for suspicious statements (DROP, TRUNCATE).
5. Alert on failed auth/privilege escalation.
6. Archive for compliance review.


Q85: How do I implement fine-grained access control for sensitive data?

Script:
```sql
-- Create users with minimal privileges
CREATE USER analyst WITH PASSWORD 'password';

-- Create view limiting sensitive columns
CREATE VIEW public_data AS 
  SELECT id, name, category FROM customers 
  WHERE sensitive_flag = false;

-- Grant access only to view
GRANT SELECT ON public_data TO analyst;

-- Implement row-level security (application-level)
SELECT * FROM customers WHERE customer_segment = user_segment();

-- Audit access patterns
SELECT user_name, query, timestamp 
FROM system.event_log 
WHERE query LIKE '%sensitive%'
ORDER BY timestamp DESC;
```

1. Create users with minimal privileges.
2. Use view-based access for column security.
3. Implement row-level security application-side.
4. Audit data access patterns.
5. Alert on unusual access.
6. Mask sensitive columns in non-production.


================================================================================
SECTION 15: MIGRATION AND DATA INTEGRATION
================================================================================

Q86: How do I migrate from PostgreSQL to CockroachDB with zero downtime?

Script:
```bash
# Set up dual-write replication
# Write to both PostgreSQL and CockroachDB simultaneously

# 1. Export PostgreSQL schema
pg_dump --schema-only postgres_db > schema.sql

# 2. Import to CockroachDB
cockroach sql --certs-dir=certs < schema.sql

# 3. Implement ETL for data migration
psql postgres_db -c "
  COPY (SELECT * FROM table_name) TO STDOUT
" | cockroach sql --certs-dir=certs <<EOF
  COPY table_name FROM STDIN;
EOF

# 4. Validate row counts
psql postgres_db -c "SELECT COUNT(*) FROM table_name;"
cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM table_name;"

# 5. Switch application traffic to CockroachDB
# 6. Maintain PostgreSQL as fallback during transition
```

1. Set up dual-write: writes to both systems.
2. Export PostgreSQL schema.
3. Import schema to CockroachDB.
4. Implement ETL for data replication.
5. Validate consistency.
6. Switch traffic, maintain fallback.


Q87: How do I implement change data capture (CDC) for data integration?

Script:
```sql
-- Enable CDC on table
CREATE CHANGEFEED FOR TABLE table_name 
  INTO 'kafka://broker:9092/topic-name' 
  WITH updated, resolved='5s', format='avro', 
  confluent_schema_registry='http://registry:8081';

-- Monitor CDC lag
SELECT job_id, status, created, modified 
FROM system.jobs 
WHERE job_type = 'CHANGEFEED'
ORDER BY modified DESC;

-- Create changefeed to webhook
CREATE CHANGEFEED FOR TABLE table_name 
  INTO 'webhook-https://example.com/endpoint?ca_cert=$CA_CERT&client_cert=$CLIENT_CERT&client_key=$CLIENT_KEY';

-- Export to file-based storage
CREATE CHANGEFEED FOR TABLE table_name 
  INTO 's3://bucket/changefeed/';
```

1. Enable CDC: CREATE CHANGEFEED FOR TABLE ... INTO 'kafka://...'
2. Captures all modifications (insert/update/delete).
3. Stream to Kafka, Pub/Sub, webhooks.
4. Monitor CDC lag.
5. Implement idempotent consumers.
6. Test failover.


Q88: How do I integrate CockroachDB with data warehousing systems?

Script:
```bash
# Use CDC for continuous sync to warehouse
cockroach sql --certs-dir=certs <<EOF
CREATE CHANGEFEED FOR TABLE analytics_table 
  INTO 'kafka://broker:9092/warehouse-topic'
  WITH updated, format='json', resolved='10s';
EOF

# Use scheduled exports for periodic sync
CREATE SCHEDULE warehouse_sync FOR 
  EXPORT (SELECT * FROM table_name) 
  TO 's3://warehouse-bucket/exports/table_name-{date}'
  RECURRING EVERY 1 day;

# ETL tool integration (dbt, Airflow)
# dbt: CockroachDB as source, transform, load to warehouse
```

1. Use CDC/scheduled exports to sync data to warehouse.
2. Implement incremental updates using CDC.
3. Use ETL tools (dbt, Airflow) for orchestration.
4. Monitor ETL job completion and freshness.
5. Implement validation for integrity.
6. Document table-to-warehouse mapping.


================================================================================
SECTION 16: OPERATIONAL EXCELLENCE
================================================================================

Q89: How do I implement runbook automation for common procedures?

Script:
```bash
# Ansible playbook for graceful node drain
---
- hosts: cockroachdb_nodes
  tasks:
    - name: Drain node
      shell: cockroach node drain --certs-dir=certs --host={{ inventory_hostname }}
      register: drain_result

    - name: Stop cockroach
      systemd:
        name: cockroachdb
        state: stopped

    - name: Perform maintenance
      shell: |
        # Your maintenance tasks here
        apt-get update && apt-get upgrade -y

    - name: Start cockroach
      systemd:
        name: cockroachdb
        state: started

    - name: Verify node health
      shell: cockroach sql --certs-dir=certs -c "SELECT 1;"
```

1. Document procedures as code: scripts, playbooks (Ansible, Chef).
2. Implement idempotent operations.
3. Add error handling and rollback.
4. Test thoroughly before production.
5. Version control runbooks.
6. Update documentation with infrastructure changes.


Q90: What is complete production deployment checklist?

Script:
```bash
# Pre-deployment checklist script
#!/bin/bash

echo "=== CockroachDB Production Deployment Checklist ==="

# Network connectivity
echo "Testing network connectivity..."
ping -c 1 node1 && echo "✓ Node1 reachable" || echo "✗ Node1 unreachable"

# Security configuration
echo "Checking TLS configuration..."
openssl x509 -in certs/node.crt -noout -dates && echo "✓ Certificates valid" || echo "✗ Certificates invalid"

# Firewall rules
echo "Checking firewall rules..."
nc -zv node1 26257 && echo "✓ Port 26257 open" || echo "✗ Port 26257 closed"

# Backup setup
echo "Testing backup configuration..."
cockroach sql --certs-dir=certs -c "CREATE SCHEDULE test_backup FOR BACKUP INTO 's3://test-bucket' RECURRING EVERY 1 day WITH RETENTION 1d;"
cockroach sql --certs-dir=certs -c "DROP SCHEDULE test_backup;" && echo "✓ Backup functional" || echo "✗ Backup failed"

# Monitoring setup
echo "Checking monitoring..."
curl -s http://node1:8080/_status/vars | head -20 && echo "✓ Metrics available" || echo "✗ Metrics unavailable"

# Load testing
echo "Running load test..."
cockroach workload run tpcc --duration=1m

echo "=== Deployment Checklist Complete ==="
```

1. Verify network connectivity and latency.
2. Confirm TLS enabled, certificates valid, firewall rules.
3. Test backup schedule and functionality.
4. Configure monitoring and alerting.
5. Document disaster recovery procedures and test.
6. Perform load testing for capacity verification.


================================================================================
KEY PRODUCTION SETTINGS AND CONFIGURATIONS
================================================================================

Essential Production Configuration:

```bash
cockroach start \
  --certs-dir=/etc/cockroach/certs \
  --listen-addr=hostname:26257 \
  --advertise-addr=hostname:26257 \
  --http-addr=hostname:8080 \
  --join=node1:26257,node2:26257,node3:26257 \
  --cache=2GB \
  --max-sql-memory=1GB \
  --store=type=ssd,path=/var/lib/cockroach \
  --log-dir=/var/log/cockroach \
  --pid-file=/var/run/cockroach.pid
```

Critical Cluster Settings:

```sql
SET CLUSTER SETTING server.clock.forward_jump_check_enabled=true;
SET CLUSTER SETTING max_retry_delay='100ms';
SET CLUSTER SETTING kv.raft.command.max_size='64MiB';
SET CLUSTER SETTING num_replicas=3;
SET CLUSTER SETTING sql.log.all_statements.enabled=true;
SET CLUSTER SETTING server.consistency_check.enabled=true;
```

Backup Schedule Template:

```sql
CREATE SCHEDULE full_backup FOR 
  BACKUP INTO 's3://backup-bucket/cockroach/full-{date-time}?AWS_ACCESS_KEY_ID=XXX&AWS_SECRET_ACCESS_KEY=XXX' 
  RECURRING EVERY 7 days 
  WITH RETENTION 30d;

CREATE SCHEDULE incremental_backup FOR 
  BACKUP INTO 's3://backup-bucket/cockroach/incremental-{date-time}?AWS_ACCESS_KEY_ID=XXX&AWS_SECRET_ACCESS_KEY=XXX' 
  RECURRING EVERY 1 day 
  FULL BACKUP 'full_backup' 
  WITH RETENTION 7d;
```

Monitoring Query Template:

```sql
-- Overall cluster health
SELECT 
  (SELECT count(*) FROM crdb_internal.nodes WHERE is_live) as live_nodes,
  (SELECT count(*) FROM crdb_internal.ranges WHERE unavailable_replicas > 0) as unavailable_ranges,
  (SELECT max(latency_p99) FROM crdb_internal.node_statement_statistics) as max_latency_p99;

-- Per-node statistics
SELECT 
  node_id, 
  count(*) as replica_count,
  sum(bytes) as total_bytes
FROM crdb_internal.ranges_no_leases
GROUP BY node_id
ORDER BY node_id;
```

================================================================================

I apologize for the technical issues. Let me provide you with additional content directly:

## ADDITIONAL 160+ QUESTIONS (Q91-Q250)

================================================================================
SECTION 17: ADVANCED CLUSTERING PATTERNS
================================================================================

Q91: How do I implement CQRS pattern with CockroachDB?

1. Separate write model (commands) from read model (queries).
2. Write model executes transactions and persists to primary cluster.
3. Read model asynchronously updates read-optimized views.
4. Use CDC to stream changes from write model to read replicas.
5. Read model optimized for specific query patterns.
6. Implement eventual consistency handling in application logic.

Script:
```sql
-- Write model: transactional table
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  customer_id INT,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

-- Read model: denormalized view
CREATE MATERIALIZED VIEW order_summary AS
  SELECT customer_id, COUNT(*) as order_count, SUM(amount) as total_amount
  FROM orders
  GROUP BY customer_id;

-- CDC for real-time sync
CREATE CHANGEFEED FOR TABLE orders 
  INTO 'kafka://broker:9092/order-events'
  WITH format='json';
```

Q92: How do I implement event sourcing pattern with CockroachDB?

1. Store immutable events in event log table as single source of truth.
2. Derive application state by replaying events.
3. Implement snapshots to avoid replaying entire history.
4. Use CDC to stream events to external systems.
5. Implement event versioning for schema evolution.
6. Test event replay thoroughly.

Script:
```sql
CREATE TABLE events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id UUID NOT NULL,
  event_type VARCHAR NOT NULL,
  event_data JSONB NOT NULL,
  version INT NOT NULL,
  created_at TIMESTAMP DEFAULT now(),
  UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, created_at);

-- Event replay to rebuild state
SELECT event_type, event_data 
FROM events 
WHERE aggregate_id = $1 
ORDER BY version ASC;
```

Q93: How do I implement saga pattern for distributed transactions?

1. Saga orchestrates distributed transactions across services.
2. Each step is transaction; compensating transactions handle rollback.
3. Implement saga choreography: services respond to events.
4. Implement saga orchestration: central orchestrator coordinates.
5. Monitor execution; alert on compensation/rollback.
6. Test failure scenarios.

Script:
```sql
CREATE TABLE saga_steps (
  saga_id UUID PRIMARY KEY,
  step_number INT,
  status VARCHAR,
  service_name VARCHAR,
  compensating_step VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

-- Saga execution tracking
CREATE TABLE saga_execution (
  saga_id UUID PRIMARY KEY,
  status VARCHAR,
  started_at TIMESTAMP,
  completed_at TIMESTAMP
);
```

Q94: How do I implement bulkhead pattern for fault isolation?

1. Isolate critical workloads in separate resource pools.
2. Limit resource allocation per workload: memory, CPU, connections.
3. Prevent one workload exhausting cluster resources.
4. Monitor bulkhead utilization.
5. Implement automatic throttling when limit exceeded.
6. Test under failure scenarios.

Script:
```sql
-- Create separate connection pools per workload
-- Application-level configuration

-- Monitor resource usage
SELECT * FROM system.statement_statistics 
  WHERE user_name = 'app_critical'
  ORDER BY execution_count DESC;

-- Set query timeout per workload
ALTER ROLE app_critical SET statement_timeout = '30s';
ALTER ROLE app_batch SET statement_timeout = '5m';
```

Q95: How do I implement circuit breaker pattern for external calls?

1. Monitor external service health continuously.
2. Open state: reject requests due to service failure.
3. Half-open: allow limited requests to test recovery.
4. Closed: normal operation, requests flow.
5. Implement graceful degradation when open.
6. Alert on state changes.

Script:
```java
// Pseudocode - circuit breaker pattern
class CircuitBreaker {
  enum State { CLOSED, OPEN, HALF_OPEN }
  State state = CLOSED;
  int failureCount = 0;
  long lastFailureTime = 0;
  
  void call(Request req) {
    if (state == OPEN) {
      if (System.currentTimeMillis() - lastFailureTime > TIMEOUT) {
        state = HALF_OPEN;
      } else {
        throw new CircuitBreakerOpenException();
      }
    }
    
    try {
      executeQuery(req);
      onSuccess();
    } catch (Exception e) {
      onFailure();
    }
  }
  
  void onSuccess() {
    failureCount = 0;
    state = CLOSED;
  }
  
  void onFailure() {
    failureCount++;
    lastFailureTime = System.currentTimeMillis();
    if (failureCount >= THRESHOLD) {
      state = OPEN;
    }
  }
}
```

Q96: How do I implement strangler pattern for gradual migration?

1. Implement new system (CockroachDB) alongside legacy system.
2. Gradually redirect traffic to new system.
3. Maintain legacy system as fallback.
4. Implement bidirectional synchronization during transition.
5. Decommission legacy system once migration complete.
6. Test fallback scenarios.

Script:
```python
# Strangler pattern - gradual migration
def route_request(request):
    # Percentage-based routing
    import random
    
    migration_percentage = get_migration_percentage()  # 0-100
    
    if random.random() < (migration_percentage / 100):
        # Route to new CockroachDB
        return new_db.execute(request)
    else:
        # Route to legacy PostgreSQL
        return legacy_db.execute(request)

# Gradually increase migration_percentage over time
# Week 1: 10%, Week 2: 25%, Week 3: 50%, Week 4: 100%
```

Q97: How do I implement retry logic with exponential backoff?

1. Retry failed operations with exponential backoff: 1s, 2s, 4s, 8s.
2. Add jitter to prevent thundering herd.
3. Set maximum retry count.
4. Implement circuit breaker.
5. Log retry attempts.
6. Test under various failures.

Script:
```python
import time
import random

def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff with jitter
            base_delay = 2 ** attempt  # 1, 2, 4, 8, 16
            jitter = random.uniform(0, base_delay)
            wait_time = base_delay + jitter
            
            print(f"Attempt {attempt + 1} failed. Retrying in {wait_time:.2f}s")
            time.sleep(wait_time)

# Usage
retry_with_backoff(lambda: db.query("SELECT * FROM table"))
```

Q98: How do I implement graceful shutdown procedures?

1. Stop accepting new connections.
2. Wait for in-flight transactions to complete.
3. Close database connections cleanly.
4. Save in-memory state if needed.
5. Implement timeout to force shutdown.
6. Monitor shutdown completion.

Script:
```bash
#!/bin/bash
# Graceful shutdown script

echo "Starting graceful shutdown..."

# Step 1: Drain connections (stop accepting new)
cockroach node drain --certs-dir=certs --host=localhost

echo "Drain complete. Waiting for in-flight transactions..."
sleep 30

# Step 2: Stop cockroach gracefully
cockroach quit --certs-dir=certs --host=localhost

# Step 3: Verify stopped
if pgrep -f "cockroach" > /dev/null; then
    echo "Warning: Process still running after 10 seconds"
    # Force kill if necessary
    pkill -9 cockroach
fi

echo "Shutdown complete"
```

Q99: How do I implement health check endpoints for orchestration?

1. Implement HTTP /health endpoint.
2. Health check queries database for connectivity.
3. Return HTTP status: 200 healthy, 500 unhealthy.
4. Include detailed health info in response.
5. Use for load balancer routing decisions.
6. Monitor health check latency.

Script:
```go
// Go health check endpoint
func healthHandler(w http.ResponseWriter, r *http.Request) {
    // Check database connectivity
    err := db.Ping()
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "unhealthy",
            "error": err.Error(),
        })
        return
    }
    
    // Check replication status
    var replicas int
    db.QueryRow("SELECT count(*) FROM crdb_internal.nodes WHERE is_live").Scan(&replicas)
    
    health := map[string]interface{}{
        "status": "healthy",
        "database": "connected",
        "live_nodes": replicas,
        "timestamp": time.Now(),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(health)
}
```

Q100: How do I implement predictable latency SLOs?

1. Define SLO targets: P99 < 100ms, P50 < 10ms.
2. Monitor against targets continuously.
3. Implement circuit breakers when SLO violated.
4. Identify and fix violating queries.
5. Capacity plan for peak load.
6. Document trade-offs with cost/features.

Script:
```sql
-- Monitor SLO compliance
SELECT 
  percentile_cont(0.50) WITHIN GROUP (ORDER BY latency) as p50_latency,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY latency) as p95_latency,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY latency) as p99_latency
FROM query_latencies
WHERE timestamp > now() - interval '1 hour';

-- Alert if exceeding SLO
CREATE ALERT high_latency_alert 
  WHEN max(p99_latency) > 100000  -- 100ms in microseconds
  NOTIFY pagerduty;
```

================================================================================
SECTION 18: DATA CONSISTENCY AND VALIDATION
================================================================================

Q101: How do I implement application-driven consistency verification?

1. Periodically verify data consistency between primary and replicas.
2. Implement checksum queries.
3. Compare checksums across replicas.
4. Alert if checksums differ.
5. Use REPAIR procedures if corruption detected.
6. Implement as operational procedure.

Script:
```sql
-- Calculate data checksums
CREATE TABLE checksums (
  table_name VARCHAR,
  checksum VARCHAR,
  calculated_at TIMESTAMP DEFAULT now()
);

-- Generate checksums for all tables
INSERT INTO checksums (table_name, checksum)
SELECT 'customers', md5(STRING_AGG(md5(row_to_json(t::text)::text), '' ORDER BY id))
FROM customers t
UNION ALL
SELECT 'orders', md5(STRING_AGG(md5(row_to_json(t::text)::text), '' ORDER BY id))
FROM orders t;

-- Verify checksums haven't changed
SELECT table_name, 
  CASE WHEN checksum = LAG(checksum) OVER (PARTITION BY table_name ORDER BY calculated_at DESC)
    THEN 'CONSISTENT'
    ELSE 'DIVERGED'
  END as status
FROM checksums
ORDER BY calculated_at DESC;
```

Q102: How do I handle foreign key constraints with eventual consistency?

1. CockroachDB enforces FK constraints synchronously.
2. For distributed systems, implement in application.
3. Use soft foreign keys (documented without constraints).
4. Implement application-level consistency checks.
5. Use background jobs for reconciliation.
6. Document consistency model.

Script:
```sql
-- Soft foreign keys approach
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  customer_id UUID NOT NULL,  -- No FK constraint
  amount DECIMAL,
  created_at TIMESTAMP DEFAULT now()
);

-- Document the relationship in metadata
CREATE TABLE foreign_key_metadata (
  parent_table VARCHAR,
  parent_column VARCHAR,
  child_table VARCHAR,
  child_column VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

INSERT INTO foreign_key_metadata VALUES 
  ('customers', 'id', 'orders', 'customer_id', now());

-- Periodic reconciliation job
SELECT o.id, o.customer_id 
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;  -- Orphaned orders
```

Q103: How do I implement distributed locks for inter-cluster coordination?

1. CockroachDB supports row-level locking.
2. Use FOR UPDATE clause for explicit locking.
3. Create lock tables for coordination.
4. Acquire lock within transaction.
5. Implement timeout logic.
6. Monitor lock wait times.

Script:
```sql
-- Lock table for coordination
CREATE TABLE distributed_locks (
  resource_id VARCHAR PRIMARY KEY,
  locked_by VARCHAR,
  locked_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);

-- Acquire lock
BEGIN;
SELECT * FROM distributed_locks 
WHERE resource_id = 'resource_123' 
FOR UPDATE;

INSERT INTO distributed_locks (resource_id, locked_by, expires_at)
VALUES ('resource_123', 'service_a', now() + interval '30s')
ON CONFLICT (resource_id) DO UPDATE SET locked_by = 'service_a', locked_at = now();

-- Perform work
UPDATE important_data SET status = 'processing' WHERE resource_id = 'resource_123';

COMMIT;

-- Release lock
DELETE FROM distributed_locks WHERE resource_id = 'resource_123';
```

Q104: How do I implement row-level security policies?

1. CockroachDB lacks native RLS.
2. Implement application-level filtering.
3. Create security views showing only allowed rows.
4. Grant SELECT to views instead of tables.
5. Implement audit triggers.
6. Document RLS implementation.

Script:
```sql
-- Base table (restricted access)
CREATE TABLE employee_data (
  id INT PRIMARY KEY,
  name VARCHAR,
  salary DECIMAL,
  department VARCHAR,
  manager_id INT
);

-- Security view (row-level filtering)
CREATE VIEW employee_data_rls AS
SELECT id, name, salary, department, manager_id
FROM employee_data
WHERE department = current_setting('app.user_department')
  OR manager_id = current_setting('app.user_id')::INT;

-- Grant access to view only
GRANT SELECT ON employee_data_rls TO analyst_role;
REVOKE ALL ON employee_data FROM analyst_role;

-- Set user context
SET app.user_department = 'Sales';
SET app.user_id = '123';
```

Q105: How do I handle long-running transactions impacting cluster?

1. Long-running transactions block MVCC cleanup.
2. Monitor transaction duration via SHOW TRANSACTIONS.
3. Kill long transactions if necessary: CANCEL SESSION.
4. Set session timeout limits.
5. Implement application logic to break into smaller transactions.
6. Alert on exceeding threshold duration.

Script:
```sql
-- Monitor long-running transactions
SELECT session_id, user_name, statement, elapsed_time 
FROM crdb_internal.node_statements 
WHERE elapsed_time > interval '5 minutes'
ORDER BY elapsed_time DESC;

-- Kill specific transaction
CANCEL SESSION 'session_id';

-- Set timeout for session
SET statement_timeout = '5m';

-- Application-level transaction batching
for batch in batches_of_1000:
    BEGIN;
    for item in batch:
        INSERT INTO table VALUES (item);
    COMMIT;
    time.sleep(0.1)  # Pause between batches
```

Q106: How do I implement multi-tenant isolation in shared cluster?

1. Separate databases per tenant.
2. Implement authentication at application layer.
3. Create tenant-specific credentials with restricted privileges.
4. Use row-level filtering for data isolation.
5. Monitor isolation; verify no data leakage.
6. Implement backup/recovery per tenant.

Script:
```sql
-- Create per-tenant schema
CREATE DATABASE tenant_acme;
CREATE DATABASE tenant_widgets;

-- Create tenant-specific users
CREATE USER tenant_acme_user WITH PASSWORD 'secure_pass_1' IN DATABASE tenant_acme;
CREATE USER tenant_widgets_user WITH PASSWORD 'secure_pass_2' IN DATABASE tenant_widgets;

-- Grant minimal privileges
GRANT ALL ON DATABASE tenant_acme TO tenant_acme_user;
GRANT ALL ON DATABASE tenant_widgets TO tenant_widgets_user;

-- Implement row-level isolation
ALTER TABLE customers ADD COLUMN tenant_id UUID;
CREATE INDEX idx_tenant ON customers(tenant_id);

-- Application enforces tenant filtering
SELECT * FROM customers WHERE tenant_id = current_setting('app.tenant_id')::UUID;
```

Q107: How do I handle hybrid SQL and NoSQL patterns?

1. CockroachDB is SQL-only.
2. Use JSON columns for semi-structured data.
3. Use JSON functions for querying.
4. Design schema for both structured/flexible data.
5. Index JSON fields for efficiency.
6. Document hybrid schema design.

Script:
```sql
-- Hybrid schema with JSON support
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name VARCHAR,
  category VARCHAR,
  -- Structured fields
  price DECIMAL,
  inventory INT,
  -- Flexible JSON data
  attributes JSONB NOT NULL DEFAULT '{}',
  metadata JSONB
);

-- Index JSON fields
CREATE INDEX idx_attributes ON products USING GIN (attributes);

-- Query JSON data
SELECT id, name, attributes->>'color' as color, attributes->>'size' as size
FROM products
WHERE attributes->>'brand' = 'Nike'
  AND (attributes->>'color')::VARCHAR IN ('Red', 'Blue');

-- Insert hybrid data
INSERT INTO products (id, name, category, price, attributes)
VALUES (
  gen_random_uuid(),
  'Running Shoe',
  'Footwear',
  129.99,
  '{"brand": "Nike", "color": "Red", "size": "10", "eco_friendly": true}'
);
```

Q108: How do I implement feature flags tied to database state?

1. Store feature flags in database table.
2. Query flags on startup.
3. Implement caching to avoid repeated queries.
4. Use transactions for consistent flag state.
5. Monitor flag changes; alert on critical changes.
6. Document flag meanings and impact.

Script:
```sql
CREATE TABLE feature_flags (
  flag_name VARCHAR PRIMARY KEY,
  enabled BOOLEAN,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);

-- Application-level caching
class FeatureFlags:
    def __init__(self, db):
        self.db = db
        self.cache = {}
        self.cache_ttl = 300  # 5 minutes
        self.cache_time = 0
    
    def is_enabled(self, flag_name):
        if time.time() - self.cache_time > self.cache_ttl:
            self.refresh_cache()
        return self.cache.get(flag_name, False)
    
    def refresh_cache(self):
        result = self.db.query("SELECT flag_name, enabled FROM feature_flags")
        self.cache = {row[0]: row[1] for row in result}
        self.cache_time = time.time()

-- SQL trigger for audit
CREATE TABLE flag_audit (
  flag_name VARCHAR,
  old_value BOOLEAN,
  new_value BOOLEAN,
  changed_at TIMESTAMP DEFAULT now()
);

CREATE TRIGGER flag_change_audit
AFTER UPDATE ON feature_flags
FOR EACH ROW
INSERT INTO flag_audit VALUES (NEW.flag_name, OLD.enabled, NEW.enabled, now());
```

Q109: How do I implement time-series data optimization?

1. Time-series data is append-only with time-based queries.
2. Partition by time: PARTITION BY RANGE (timestamp).
3. Use consistent hash sharding for even distribution.
4. Archive old partitions to reduce active storage.
5. Create indexes on timestamp column.
6. Monitor partition growth; split large partitions.

Script:
```sql
CREATE TABLE metrics (
  timestamp TIMESTAMP NOT NULL,
  metric_name VARCHAR NOT NULL,
  value DECIMAL,
  tags JSONB,
  PRIMARY KEY (timestamp, metric_name)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01'),
  PARTITION p_2024_03 VALUES FROM ('2024-03-01') TO ('2024-04-01')
);

-- Archive old partitions
CREATE TABLE metrics_archive AS
SELECT * FROM metrics WHERE timestamp < '2023-01-01';

DELETE FROM metrics WHERE timestamp < '2023-01-01';

-- Query recent data efficiently
SELECT timestamp, value 
FROM metrics 
WHERE metric_name = 'cpu_usage' 
  AND timestamp > now() - interval '24 hours'
ORDER BY timestamp DESC;

-- Continuous insert performance
INSERT INTO metrics (timestamp, metric_name, value, tags)
SELECT now(), 'cpu_usage', 75.5, '{"host": "server1"}';
```

Q110: How do I implement cost allocation for multi-tenant cluster?

1. Track resource usage per tenant: queries, storage, replication.
2. Implement resource tagging.
3. Monitor cost trends; alert on unexpected increases.
4. Implement tenant-specific resource limits.
5. Report costs to tenants for billing.
6. Optimize queries to reduce per-tenant costs.

Script:
```sql
CREATE TABLE tenant_costs (
  tenant_id UUID,
  month DATE,
  query_count BIGINT,
  storage_bytes BIGINT,
  replication_bytes BIGINT,
  total_cost DECIMAL,
  calculated_at TIMESTAMP DEFAULT now()
);

-- Calculate monthly costs
INSERT INTO tenant_costs (tenant_id, month, query_count, storage_bytes, total_cost)
SELECT 
  tenant_id,
  date_trunc('month', now())::date,
  COUNT(*),
  SUM(bytes),
  (COUNT(*) * 0.0001) + (SUM(bytes) * 0.00001),  -- Example pricing
  now()
FROM tenant_queries
WHERE DATE_TRUNC('month', executed_at) = DATE_TRUNC('month', now())
GROUP BY tenant_id;

-- Implement tenant resource quotas
SET CLUSTER SETTING sql.defaults.max_execution_time_per_query = '30s';
CREATE ROLE tenant_acme_role WITH CONNECTION LIMIT 100;  -- Max 100 connections
```

================================================================================
SECTION 19: PRODUCTION TROUBLESHOOTING DEEP DIVE
================================================================================

Q111: How do I diagnose slow replication and identify lag causes?

1. Monitor replication metrics: ranges.rebalancing.writes, raft.process.append.entries.
2. High lag indicates network issues or slow followers.
3. Check network latency between nodes.
4. Monitor follower CPU and disk I/O for bottlenecks.
5. Identify and fix root cause.
6. Monitor post-fix for return to normal.

Script:
```sql
-- Monitor replication lag
SELECT 
  node_id,
  SUM(raft.process.append.entries_total) as total_appends,
  SUM(raft.heartbeats.pending) as pending_heartbeats
FROM crdb_internal.node_metrics
GROUP BY node_id
ORDER BY pending_heartbeats DESC;

-- Check for slow followers
SELECT 
  replica_node_id,
  COUNT(*) as slow_replica_count
FROM crdb_internal.ranges
WHERE includes_slow_replica = true
GROUP BY replica_node_id
ORDER BY slow_replica_count DESC;
```

Q112: How do I implement anomaly detection for cluster health?

1. Establish baseline metrics under normal operation.
2. Use statistical methods to detect outliers.
3. Alert when metrics deviate significantly.
4. Implement ML models for subtle anomalies.
5. Document anomaly signatures.
6. Implement feedback loop for improvement.

Script:
```python
import numpy as np
from scipy import stats

def detect_anomalies(metrics, threshold=2.5):
    """Detect anomalies using Z-score method"""
    mean = np.mean(metrics)
    std = np.std(metrics)
    
    z_scores = np.abs(stats.zscore(metrics))
    anomalies = z_scores > threshold
    
    return anomalies

# Usage
latencies = [45, 48, 52, 51, 49, 150, 48, 47, 50]  # 150 is anomaly
is_anomaly = detect_anomalies(latencies)
print(f"Anomalies detected at indices: {np.where(is_anomaly)[0]}")
```

Q113: How do I implement database activity monitoring (DAM)?

1. Capture all database activities: queries, logins, modifications.
2. Store in separate system to prevent tampering.
3. Monitor for policy violations and anomalies.
4. Alert on sensitive data access.
5. Generate compliance reports.
6. Maintain long-term records.

Script:
```sql
CREATE TABLE activity_log (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  timestamp TIMESTAMP DEFAULT now(),
  user_name VARCHAR,
  session_id VARCHAR,
  statement VARCHAR,
  rows_affected INT,
  execution_time_ms INT,
  source_ip VARCHAR,
  is_sensitive BOOLEAN
);

-- Create trigger for sensitive data access
CREATE TRIGGER sensitive_access_audit
AFTER SELECT ON customers
FOR EACH ROW
INSERT INTO activity_log (user_name, statement, is_sensitive)
VALUES (current_user, current_statement(), true);

-- Query sensitive access patterns
SELECT user_name, COUNT(*) as access_count, COUNT(DISTINCT timestamp::date) as days_accessed
FROM activity_log
WHERE is_sensitive = true
  AND timestamp > now() - interval '30 days'
GROUP BY user_name
ORDER BY access_count DESC;
```

Q114: How do I handle uneven disk usage across stores?

1. Query per-store metrics.
2. Identify rebalancing or placement issues.
3. Trigger rebalancing if significant imbalance.
4. Check zone configuration constraints.
5. Verify all stores healthy and accessible.
6. Monitor large ranges preventing split.

Script:
```sql
-- Monitor per-store usage
SELECT 
  store_id, 
  capacity_total,
  capacity_used,
  (capacity_used::FLOAT / capacity_total) * 100 as used_percent,
  capacity_available
FROM crdb_internal.stores
ORDER BY used_percent DESC;

-- Identify stores with imbalance (>15% difference)
WITH store_stats AS (
  SELECT 
    store_id,
    capacity_used,
    AVG(capacity_used) OVER () as avg_capacity
  FROM crdb_internal.stores
)
SELECT store_id, capacity_used, avg_capacity, 
  ABS(capacity_used - avg_capacity) as imbalance
FROM store_stats
WHERE ABS(capacity_used - avg_capacity) > avg_capacity * 0.15
ORDER BY imbalance DESC;
```

Q115: How do I implement comprehensive cluster self-healing?

1. Automatic recovery from individual failures.
2. Continuous health monitoring.
3. Automatic repairs: replace dead nodes, rebalance.
4. Alerts for manual intervention needs.
5. Document manual procedures for beyond-automation issues.
6. Test regularly with failure simulation.

Script:
```bash
#!/bin/bash
# Cluster self-healing script

echo "=== Cluster Self-Healing Check ==="

# Check node liveness
DEAD_NODES=$(cockroach sql --certs-dir=certs -c "
  SELECT COUNT(*) FROM crdb_internal.nodes WHERE NOT is_live;"
)

if [ "$DEAD_NODES" -gt 0 ]; then
  echo "WARNING: $DEAD_NODES nodes are dead"
  # Automatic trigger rebalancing
  cockroach sql --certs-dir=certs -c "SET CLUSTER SETTING kv.allocator.mode='aggressive';"
fi

# Check for unavailable ranges
UNAVAILABLE=$(cockroach sql --certs-dir=certs -c "
  SELECT COUNT(*) FROM crdb_internal.ranges WHERE unavailable_replicas > 0;"
)

if [ "$UNAVAILABLE" -gt 0 ]; then
  echo "WARNING: $UNAVAILABLE ranges are unavailable"
  # Trigger recovery
  cockroach sql --certs-dir=certs -c "SET CLUSTER SETTING server.consistency_check.enabled=true;"
fi

# Check disk space
FULL_STORES=$(cockroach sql --certs-dir=certs -c "
  SELECT COUNT(*) FROM crdb_internal.stores 
  WHERE (capacity_used::FLOAT / capacity_total) > 0.9;"
)

if [ "$FULL_STORES" -gt 0 ]; then
  echo "CRITICAL: $FULL_STORES stores near capacity"
  # Alert needs manual intervention
  send_alert "Cluster storage critical"
fi

echo "=== Self-Healing Check Complete ==="
```

================================================================================
SECTION 20: PRODUCTION INCIDENT RESPONSE
================================================================================

Q116: How do I handle database corruption with active users impacted?

1. Alert users immediately; prepare for interruption.
2. Implement read-only mode.
3. Identify unaffected data through point-in-time queries.
4. Create clean table from recovered data.
5. Switch applications to recovered table.
6. Communicate with users about recovery and data loss.

Script:
```sql
-- Step 1: Enable read-only mode
ALTER SYSTEM SET default_transaction_read_only = on;

-- Step 2: Query uncorrupted data
SELECT * FROM corrupted_table 
  AS OF SYSTEM TIME '2024-01-15 12:00:00' 
  LIMIT 100;

-- Step 3: Create recovered table
CREATE TABLE recovered_table AS 
SELECT * FROM corrupted_table 
  AS OF SYSTEM TIME '2024-01-15 12:00:00';

-- Step 4: Validate
SELECT COUNT(*) FROM recovered_table;

-- Step 5: Swap tables
ALTER TABLE corrupted_table RENAME TO corrupted_table_corrupted;
ALTER TABLE recovered_table RENAME TO corrupted_table;

-- Step 6: Resume operations
ALTER SYSTEM SET default_transaction_read_only = off;
```

Q117: How do I recover from application bug causing mass incorrect updates?

1. Identify affected records through timestamps.
2. Stop application immediately.
3. Recover good data from point-in-time.
4. Implement correction.
5. Test fix thoroughly in staging.
6. Deploy corrected application.

Script:
```sql
-- Step 1: Identify affected records
SELECT id, COUNT(*) as update_count
FROM audit_log
WHERE action = 'UPDATE' 
  AND table_name = 'orders'
  AND updated_at >= '2024-01-15 14:00:00'
GROUP BY id;

-- Step 2: Recover good data
CREATE TABLE orders_recovered AS
SELECT * FROM orders 
  AS OF SYSTEM TIME '2024-01-15 13:59:00'
WHERE id IN (
  SELECT DISTINCT id FROM audit_log
  WHERE action = 'UPDATE' AND updated_at >= '2024-01-15 14:00:00'
);

-- Step 3: Merge recovered data
BEGIN;
DELETE FROM orders WHERE id IN (SELECT id FROM orders_recovered);
INSERT INTO orders SELECT * FROM orders_recovered;
COMMIT;

-- Step 4: Validate
SELECT COUNT(*) FROM orders;
```

Q118: How do I recover from network partition in cluster?

1. Identify partition through network monitoring.
2. Only partition with quorum can write.
3. Other partition becomes read-only.
4. Repair network partition.
5. Cluster automatically synchronizes.
6. Verify data consistency post-repair.

Script:
```bash
# Network partition detection script
#!/bin/bash

# Check connectivity between nodes
for node in node1 node2 node3; do
  for peer in node1 node2 node3; do
    if [ "$node" != "$peer" ]; then
      if ! nc -zv $node $peer:26257 2>/dev/null; then
        echo "ALERT: Partition detected between $node and $peer"
        # Automatic alert
        send_alert "Network partition detected"
      fi
    fi
  done
done
```

Q119: How do I verify restore success without full comparison?

1. Validate row counts for verification.
2. Check data distribution statistics.
3. Compare checksums of large tables.
4. Query sample data and verify results.
5. Run application smoke tests.
6. Document validation procedures.

Script:
```sql
-- Validate restore completeness
SELECT table_name, COUNT(*) as row_count 
FROM information_schema.tables t
LEFT JOIN (SELECT * FROM information_schema.table_schema) ts ON t.table_name = ts.table_name
GROUP BY table_name
ORDER BY table_name;

-- Compare checksums
SELECT table_name, 
  md5(STRING_AGG(md5(id::text) ORDER BY id, '')) as table_checksum
FROM (
  SELECT 'customers' as table_name, id FROM customers
  UNION ALL
  SELECT 'orders', id FROM orders
) t
GROUP BY table_name;

-- Sample data validation
SELECT * FROM customers ORDER BY RANDOM() LIMIT 10;
SELECT * FROM orders ORDER BY RANDOM() LIMIT 10;
```

Q120: How do I implement zero-copy data migration between clusters?

1. Physical replication enables zero-copy migration.
2. Set up replication from old cluster to new.
3. Allow replication to catch up.
4. Promote new cluster to primary.
5. Redirect traffic to new cluster.
6. Decommission old cluster after stability.

Script:
```bash
# Physical cluster replication setup
cockroach sql --certs-dir=certs <<EOF
-- On source cluster: enable physical replication
ALTER CLUSTER SET replication_destination = 'target_cluster';

-- Monitor replication status
SELECT status, replicated_time 
FROM crdb_internal.cluster_replication_status;

-- On target cluster: wait for replication to catch up
-- Once caught up, promote
ALTER CLUSTER SET REPLICATION FACTOR = 1;

-- Redirect application traffic to target_cluster
EOF
```

================================================================================
COCKROACHDB ADMINISTRATION GUIDE - FINAL CONTINUATION
Questions Q121-Q250 with Advanced Scenarios, Compliance, and Operational Procedures

================================================================================
SECTION 21: COMPLIANCE AND DATA GOVERNANCE
================================================================================

Q121: How do I implement compliance with GDPR data residency requirements?

1. Use locality configuration to enforce data within specific regions: --locality=region=eu
2. Verify replica placement: ALTER TABLE CONFIGURE ZONE USING constraints='[+region=eu]'
3. Monitor cross-region movement; alert if data leaves allowed regions.
4. Implement backup location restrictions to comply with rules.
5. Document data residency architecture and verification procedures.
6. Perform quarterly compliance audits.

Script:
```sql
-- GDPR-compliant zone configuration
ALTER TABLE customer_data CONFIGURE ZONE USING 
  num_replicas = 3,
  constraints = '[+region=eu]',
  lease_preferences = '[[+region=eu]]';

-- Verify compliance
SELECT table_name, config 
FROM system.zones 
WHERE config LIKE '%eu%';

-- Monitor for data leaving EU
SELECT * FROM crdb_internal.ranges_no_leases 
WHERE table_name = 'customer_data' 
  AND zone_name NOT IN ('eu-west', 'eu-central');

-- Data deletion for GDPR right-to-be-forgotten
DELETE FROM customer_data WHERE customer_id = $1;
DELETE FROM orders WHERE customer_id = $1;
DELETE FROM audit_log WHERE customer_id = $1;
```

Q122: How do I implement encryption at rest and in transit?

1. In transit: TLS certificates for all connections (--certs-dir=certs).
2. At rest: CockroachCloud supports encryption via cloud provider.
3. Self-hosted: use CMEK through cloud provider KMS.
4. Enable during cluster initialization; cannot add later.
5. Verify encryption status through cluster configuration.
6. Implement key rotation policies.

Script:
```bash
# Enable TLS for cluster
cockroach start \
  --certs-dir=/etc/cockroach/certs \
  --listen-addr=localhost:26257

# Verify TLS enabled
openssl s_client -connect localhost:26257 -cert certs/client.crt -key certs/client.key -cacert certs/ca.crt

# For CockroachCloud with CMEK
# 1. Create AWS KMS key
aws kms create-key --description "CockroachDB CMEK"

# 2. Enable CMEK in cluster settings through console
# 3. Verify encryption in cluster info
```

Q123: How do I implement audit logging for regulatory compliance?

1. Enable statement logging: SET CLUSTER SETTING sql.trace.log_statement_execute=true
2. Capture all SQL statements with user and timestamp.
3. Forward logs to centralized security system.
4. Monitor for suspicious statements and access patterns.
5. Alert on critical operations (DROP, TRUNCATE).
6. Archive logs for compliance review and forensics.

Script:
```sql
-- Enable comprehensive audit logging
SET CLUSTER SETTING sql.trace.log_statement_execute=true;
SET CLUSTER SETTING server.auth_log.enabled=true;

-- Create audit log table
CREATE TABLE audit_log_external (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  timestamp TIMESTAMP DEFAULT now(),
  user_name VARCHAR,
  session_id VARCHAR,
  statement VARCHAR,
  execution_time_ms INT,
  rows_affected INT,
  source_ip VARCHAR,
  event_type VARCHAR  -- INSERT, UPDATE, DELETE, DROP, etc.
);

-- Trigger for all data modifications
CREATE TRIGGER audit_insert_trigger
AFTER INSERT ON sensitive_table
FOR EACH ROW
INSERT INTO audit_log_external (event_type, statement, rows_affected)
VALUES ('INSERT', 'INSERT INTO sensitive_table', 1);

-- Query audit log for compliance
SELECT user_name, event_type, COUNT(*) as count
FROM audit_log_external
WHERE timestamp > now() - interval '30 days'
GROUP BY user_name, event_type
ORDER BY count DESC;
```

Q124: How do I implement data classification and sensitivity labeling?

1. Create metadata table for data classification.
2. Tag tables/columns with sensitivity levels.
3. Implement access controls based on classification.
4. Enforce encryption for highly sensitive data.
5. Monitor access to sensitive data.
6. Document classification policy.

Script:
```sql
-- Data classification metadata
CREATE TABLE data_classification (
  table_name VARCHAR,
  column_name VARCHAR,
  sensitivity_level VARCHAR,  -- PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
  pii_flag BOOLEAN,
  retention_days INT,
  created_at TIMESTAMP DEFAULT now()
);

INSERT INTO data_classification VALUES
  ('customers', 'id', 'PUBLIC', false, 2555, now()),
  ('customers', 'email', 'CONFIDENTIAL', true, 2555, now()),
  ('customers', 'ssn', 'RESTRICTED', true, 2555, now()),
  ('orders', 'customer_id', 'INTERNAL', true, 2555, now());

-- Access control based on classification
CREATE ROLE restricted_access NOLOGIN;
GRANT SELECT ON customers TO restricted_access;
REVOKE SELECT (ssn) ON customers FROM restricted_access;

-- Implement data masking for non-production
SELECT id, email, 'XXXX-XXXX-' || RIGHT(ssn, 4) as ssn_masked
FROM customers;
```

Q125: How do I implement data retention and purge policies?

1. Define retention schedules based on data type.
2. Use TTL or scheduled jobs to expire old data.
3. Archive data to cold storage before deletion.
4. Implement soft deletes for audit trail preservation.
5. Document retention policies and obtain approval.
6. Implement automated purge procedures.

Script:
```sql
-- Retention policy configuration
CREATE TABLE retention_policies (
  table_name VARCHAR,
  retention_days INT,
  policy_type VARCHAR,  -- HARD_DELETE, SOFT_DELETE, ARCHIVE
  archive_location VARCHAR,
  last_purged_date DATE,
  created_at TIMESTAMP DEFAULT now()
);

INSERT INTO retention_policies VALUES
  ('transactions', 2555, 'HARD_DELETE', NULL, now()::date, now()),
  ('audit_log', 3650, 'ARCHIVE', 's3://archive-bucket/', now()::date, now()),
  ('deleted_users', 730, 'SOFT_DELETE', NULL, now()::date, now());

-- Scheduled purge job
CREATE SCHEDULE purge_old_transactions FOR 
  DELETE FROM transactions 
  WHERE created_at < now() - interval '7 years'
  RECURRING EVERY 1 day 
  LIMIT 10000;

-- Soft delete implementation
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP DEFAULT NULL;
CREATE INDEX idx_not_deleted ON users(id) WHERE deleted_at IS NULL;

-- Query only active users
SELECT * FROM users WHERE deleted_at IS NULL;
```

Q126: How do I handle right-to-be-forgotten (RTBF) requests?

1. Identify all personal data for individual across tables.
2. Execute coordinated deletion across all tables.
3. Verify complete deletion including backups.
4. Audit deletion with timestamp and operator.
5. Document deletion for compliance proof.
6. Implement systematic RTBF procedures.

Script:
```sql
-- Right-to-be-forgotten implementation
BEGIN;

-- Delete personal data across all tables
DELETE FROM customers WHERE customer_id = $1;
DELETE FROM orders WHERE customer_id = $1;
DELETE FROM addresses WHERE customer_id = $1;
DELETE FROM phone_numbers WHERE customer_id = $1;
DELETE FROM payment_methods WHERE customer_id = $1;

-- Log deletion for compliance
INSERT INTO rtbf_audit_log (customer_id, deleted_at, deleted_by, tables_affected)
VALUES ($1, now(), current_user, 5);

-- Verify deletion
SELECT COUNT(*) FROM customers WHERE customer_id = $1;
-- Should return 0

COMMIT;

-- Note: Backups containing deleted data must be excluded from retention
-- or newer backups created after RTBF request honored
```

Q127: How do I implement PCI DSS compliance for payment data?

1. Encrypt payment card data at rest and in transit.
2. Restrict access to payment data (encryption keys separate from data).
3. Implement network segmentation (payment data isolated).
4. Maintain audit logs for all payment data access.
5. Implement regular security testing and scans.
6. Document PCI compliance procedures.

Script:
```sql
-- PCI-compliant payment data storage
CREATE TABLE payments (
  payment_id UUID PRIMARY KEY,
  customer_id UUID,
  amount DECIMAL,
  encrypted_card_data BYTEA,  -- Encrypted with KMS
  card_token VARCHAR,  -- Tokenized instead of full PAN
  merchant_reference VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

-- Restrict access to payment data
CREATE ROLE payment_processor NOLOGIN;
GRANT SELECT ON payments TO payment_processor;

-- Sensitive columns: restrict to specific users
CREATE ROLE payment_auditor NOLOGIN;
GRANT SELECT (payment_id, customer_id, amount, created_at) ON payments TO payment_auditor;
REVOKE SELECT (encrypted_card_data) ON payments FROM payment_auditor;

-- Audit all payment access
CREATE TABLE payment_access_log (
  access_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  payment_id UUID,
  accessed_at TIMESTAMP DEFAULT now()
);

-- Create trigger to log access
CREATE TRIGGER log_payment_access
AFTER SELECT ON payments
FOR EACH ROW
INSERT INTO payment_access_log (user_name, payment_id)
VALUES (current_user, NEW.payment_id);
```

Q128: How do I implement HIPAA compliance for healthcare data?

1. Encrypt all PHI (Protected Health Information) at rest and in transit.
2. Implement granular access controls with role-based restrictions.
3. Maintain comprehensive audit logs for all PHI access.
4. Implement data integrity controls (checksums, digital signatures).
5. Enforce minimum necessary principle (access only required data).
6. Regular compliance audits and penetration testing.

Script:
```sql
-- HIPAA-compliant patient data schema
CREATE TABLE patients (
  patient_id UUID PRIMARY KEY,
  encrypted_ssn BYTEA,  -- Encrypted
  encrypted_name BYTEA,  -- Encrypted
  encrypted_medical_record BYTEA,  -- Encrypted
  access_log_id UUID,
  created_at TIMESTAMP DEFAULT now()
);

-- Restrict access by role
CREATE ROLE nurse NOLOGIN;
CREATE ROLE doctor NOLOGIN;
CREATE ROLE patient_records_admin NOLOGIN;

-- Doctors see all medical information
GRANT SELECT ON patients TO doctor;

-- Nurses see limited information
GRANT SELECT (patient_id, encrypted_medical_record) ON patients TO nurse;

-- Comprehensive audit
CREATE TABLE hipaa_access_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  patient_id UUID,
  accessed_columns VARCHAR[],
  purpose VARCHAR,
  accessed_at TIMESTAMP DEFAULT now()
);

-- Integrity verification
ALTER TABLE patients ADD COLUMN data_integrity_hash VARCHAR;
UPDATE patients SET data_integrity_hash = 
  md5(patient_id::text || encrypted_ssn::text || encrypted_name::text);
```

Q129: How do I implement SOC 2 compliance procedures?

1. Implement comprehensive security controls (access control, encryption).
2. Maintain audit logs for all system access and changes.
3. Implement change management procedures with approval workflows.
4. Perform regular risk assessments and vulnerability scanning.
5. Implement incident response procedures with documentation.
6. Maintain security training records for personnel.

Script:
```sql
-- SOC 2 change management tracking
CREATE TABLE change_log (
  change_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  change_type VARCHAR,  -- CONFIG, SCHEMA, SECURITY, BACKUP
  description VARCHAR,
  changed_by VARCHAR,
  approved_by VARCHAR,
  approval_date TIMESTAMP,
  change_date TIMESTAMP,
  rollback_procedure VARCHAR,
  status VARCHAR,  -- PENDING, APPROVED, IMPLEMENTED, ROLLED_BACK
  created_at TIMESTAMP DEFAULT now()
);

-- Risk assessment tracking
CREATE TABLE risk_assessment (
  assessment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  risk_category VARCHAR,
  risk_description VARCHAR,
  likelihood VARCHAR,  -- LOW, MEDIUM, HIGH
  impact VARCHAR,  -- LOW, MEDIUM, HIGH
  mitigation_plan VARCHAR,
  owner VARCHAR,
  due_date DATE,
  completed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT now()
);

-- Vulnerability scanning results
CREATE TABLE vulnerability_scans (
  scan_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_date DATE,
  vulnerability_id VARCHAR,
  severity VARCHAR,  -- LOW, MEDIUM, HIGH, CRITICAL
  description VARCHAR,
  remediation VARCHAR,
  remediated_date DATE,
  created_at TIMESTAMP DEFAULT now()
);

-- Incident tracking
CREATE TABLE security_incidents (
  incident_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  incident_type VARCHAR,
  discovery_date TIMESTAMP,
  description VARCHAR,
  impact_assessment VARCHAR,
  resolution VARCHAR,
  resolved_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT now()
);
```

Q130: How do I implement ISO 27001 information security controls?

1. Implement asset management: track all data assets and resources.
2. Access control: principle of least privilege throughout system.
3. Cryptography: encryption for sensitive data at rest and transit.
4. Physical security: secure data center with access controls.
5. Personnel security: background checks, confidentiality agreements.
6. Regular audits: compliance verification and improvement.

Script:
```sql
-- Asset management
CREATE TABLE asset_inventory (
  asset_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_name VARCHAR,
  asset_type VARCHAR,  -- DATABASE, SERVER, APPLICATION
  classification VARCHAR,  -- PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
  owner VARCHAR,
  custodian VARCHAR,
  backup_status VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

-- Access control matrix
CREATE TABLE access_matrix (
  access_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  asset_id UUID,
  permission VARCHAR,  -- READ, WRITE, EXECUTE, DELETE, ADMIN
  justification VARCHAR,
  approved_by VARCHAR,
  approval_date DATE,
  expiration_date DATE,
  created_at TIMESTAMP DEFAULT now()
);

-- Cryptography register
CREATE TABLE cryptography_register (
  crypto_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  algorithm VARCHAR,
  key_length INT,
  usage VARCHAR,  -- DATA_ENCRYPTION, KEY_ENCRYPTION, AUTHENTICATION
  key_generation_date DATE,
  key_expiration_date DATE,
  rotation_schedule VARCHAR,
  key_holder VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);
```

================================================================================
SECTION 22: ADVANCED OBSERVABILITY AND METRICS
================================================================================

Q131: How do I implement distributed tracing for request tracking?

1. Enable session tracing: SET SESSION TRACE = ON
2. Trace captures all operations, network round trips, timing.
3. Export traces to distributed tracing system (Jaeger, Zipkin).
4. Correlate traces with application logs using request IDs.
5. Analyze trace latency breakdown to identify bottlenecks.
6. Implement sampling for high-volume tracing.

Script:
```sql
-- Enable distributed tracing
SET SESSION TRACE = ON;

-- Execute query to trace
SELECT * FROM orders WHERE customer_id = $1;

-- View trace output
SELECT * FROM [SHOW TRACE FOR SESSION];

-- Export trace to external system
SELECT 
  timestamp,
  span_id,
  operation,
  duration_micros
FROM [SHOW TRACE FOR SESSION]
ORDER BY timestamp;

-- Application-level trace correlation
-- Include trace-id header in all requests
-- Log trace-id with every database operation
```

Script (Application level - pseudocode):
```python
import uuid
from opentelemetry import trace, metrics
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Set up Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)
tracer = trace.get_tracer(__name__)

# Create spans for database operations
def query_database(query):
    trace_id = str(uuid.uuid4())
    with tracer.start_as_current_span("db_query") as span:
        span.set_attribute("trace_id", trace_id)
        span.set_attribute("query", query)
        # Execute query with trace context
        return db.execute(query)
```

Q132: How do I implement real-time alerting for SLA violations?

1. Define SLO thresholds in monitoring configuration.
2. Continuously compare actual metrics against targets.
3. Alert when SLO compliance at risk.
4. Implement tiered alerting (warning, critical).
5. Route alerts to appropriate responder.
6. Track alert frequency and accuracy.

Script:
```yaml
# Prometheus alerting rules
groups:
  - name: slo_violations
    rules:
      - alert: P99LatencySLOViolation
        expr: sql_exec_latency_p99 > 100000  # 100ms in microseconds
        for: 5m
        annotations:
          severity: warning
          summary: "P99 latency {{ $value }}ms exceeds SLO of 100ms"

      - alert: ErrorRateSLOViolation
        expr: sql_exec_failed / sql_exec_total > 0.01  # 1% error rate
        for: 5m
        annotations:
          severity: critical
          summary: "Error rate {{ $value | humanizePercentage }} exceeds SLO"

      - alert: AvailabilitySLOViolation
        expr: (node_down / node_total) > 0.05
        annotations:
          severity: critical
          summary: "Availability below SLO: {{ $value | humanizePercentage }} nodes down"
```

Q133: How do I implement cost monitoring and budgeting?

1. Track resource usage by dimension (tenant, service, region).
2. Calculate costs based on usage metrics.
3. Set monthly/quarterly budgets per cost center.
4. Alert when approaching budget limits.
5. Generate chargeback reports for cost allocation.
6. Implement cost optimization recommendations.

Script:
```sql
-- Cost tracking by tenant
CREATE TABLE tenant_cost_tracking (
  tenant_id UUID,
  month DATE,
  query_count BIGINT,
  storage_bytes BIGINT,
  data_transfer_bytes BIGINT,
  query_cost DECIMAL,
  storage_cost DECIMAL,
  transfer_cost DECIMAL,
  total_cost DECIMAL,
  budget_limit DECIMAL,
  created_at TIMESTAMP DEFAULT now()
);

-- Calculate monthly costs
INSERT INTO tenant_cost_tracking (tenant_id, month, query_count, storage_bytes, total_cost, budget_limit)
SELECT 
  tenant_id,
  date_trunc('month', now())::date,
  COUNT(*),
  SUM(bytes),
  (COUNT(*) * 0.0001) + (SUM(bytes) * 0.00001),
  10000.00,  -- $10k budget
  now()
FROM tenant_usage
WHERE date_trunc('month', created_at) = date_trunc('month', now())
GROUP BY tenant_id;

-- Alert on budget exceeded
SELECT tenant_id, total_cost, budget_limit,
  CASE WHEN total_cost > budget_limit THEN 'EXCEEDED'
       WHEN total_cost > budget_limit * 0.9 THEN 'WARNING'
       ELSE 'OK'
  END as status
FROM tenant_cost_tracking
WHERE month = date_trunc('month', now())::date;
```

Q134: How do I implement custom metrics collection?

1. Use stored procedures to compute application-specific metrics.
2. Store metrics in separate tables for analysis.
3. Implement metrics collection jobs scheduled periodically.
4. Export metrics to external monitoring system.
5. Monitor collection overhead.
6. Document custom metrics and interpretation.

Script:
```sql
-- Custom business metrics
CREATE TABLE business_metrics (
  metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  metric_name VARCHAR,
  metric_value DECIMAL,
  dimensions JSONB,  -- Tags like {tenant_id, region, service}
  measured_at TIMESTAMP DEFAULT now()
);

-- Stored procedure for metrics collection
CREATE PROCEDURE collect_business_metrics() AS $$
BEGIN
  -- Revenue metrics
  INSERT INTO business_metrics (metric_name, metric_value, dimensions)
  SELECT 
    'daily_revenue',
    SUM(amount),
    '{"date": "' || CURRENT_DATE || '"}'::jsonb
  FROM orders
  WHERE DATE(created_at) = CURRENT_DATE;

  -- Customer metrics
  INSERT INTO business_metrics (metric_name, metric_value, dimensions)
  SELECT 
    'active_customers',
    COUNT(DISTINCT customer_id),
    '{"period": "7d"}'::jsonb
  FROM orders
  WHERE created_at > now() - interval '7 days';

  -- Performance metrics
  INSERT INTO business_metrics (metric_name, metric_value, dimensions)
  SELECT 
    'order_processing_time_seconds',
    AVG(EXTRACT(EPOCH FROM (shipped_at - created_at))),
    '{"status": "shipped"}'::jsonb
  FROM orders
  WHERE shipped_at IS NOT NULL;
END
$$ LANGUAGE plpgsql;

-- Schedule metrics collection
CREATE SCHEDULE collect_metrics FOR CALL collect_business_metrics() 
  RECURRING EVERY 1 hour;
```

Q135: How do I implement observability for multi-cluster deployments?

1. Aggregate metrics from all clusters to central monitoring.
2. Correlate logs across clusters using request IDs.
3. Implement distributed tracing across cluster boundaries.
4. Monitor inter-cluster replication status.
5. Alert on cross-cluster communication issues.
6. Implement dashboards for multi-cluster overview.

Script:
```sql
-- Multi-cluster metrics aggregation
CREATE TABLE cluster_metrics (
  cluster_id VARCHAR,
  metric_name VARCHAR,
  metric_value DECIMAL,
  measured_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT now()
);

-- Replication status monitoring
SELECT 
  cluster_id,
  status,
  replicated_time,
  NOW() - replicated_time as replication_lag
FROM cluster_replication_status
ORDER BY replication_lag DESC;

-- Cross-cluster connectivity test
SELECT 
  source_cluster,
  destination_cluster,
  avg_latency_ms,
  packet_loss_percent
FROM cluster_connectivity_metrics
ORDER BY avg_latency_ms DESC;
```

================================================================================
SECTION 23: ADVANCED OPERATIONAL PROCEDURES
================================================================================

Q136: How do I implement rolling zero-downtime updates?

1. Plan update sequence for odd-numbered nodes.
2. Update one node at a time to maintain quorum.
3. Verify node health after each update.
4. Monitor cluster performance during update.
5. Rollback procedure if issues detected.
6. Document update completion.

Script:
```bash
#!/bin/bash
# Rolling update script for CockroachDB

NODES=("node1" "node2" "node3")
BACKUP_DIR="/backups/pre-update"

# Backup current state
echo "Creating pre-update backup..."
cockroach sql --certs-dir=certs -c "
  BACKUP INTO 's3://backup-bucket/pre-update-backup';"

# Update nodes one at a time
for node in "${NODES[@]}"; do
  echo "Updating $node..."
  
  # Drain node
  cockroach node drain --host=$node --certs-dir=certs
  
  # Stop service
  ssh $node "systemctl stop cockroachdb"
  
  # Update binary
  ssh $node "
    cd /usr/local/bin
    curl -O https://binaries.cockroachdb.com/cockroach-v24.2.0.linux-amd64.tgz
    tar xzf cockroach-v24.2.0.linux-amd64.tgz
    cp cockroach-v24.2.0/cockroach ./
  "
  
  # Start service
  ssh $node "systemctl start cockroachdb"
  
  # Verify node health
  sleep 30
  cockroach sql --certs-dir=certs -c "
    SELECT node_id, is_live FROM crdb_internal.nodes WHERE address = '$node';"
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Node $node failed to start. Rolling back..."
    # Rollback logic
    exit 1
  fi
done

echo "Update complete. Finalizing..."
cockroach sql --certs-dir=certs -c "
  SET CLUSTER SETTING version = '24.2';"
```

Q137: How do I implement canary deployments for schema changes?

1. Deploy schema change to canary cluster first.
2. Run extensive testing on canary.
3. Monitor canary for issues before broader rollout.
4. Deploy to staging environment.
5. Final deployment to production with rollback plan.
6. Monitor post-deployment metrics closely.

Script:
```sql
-- Canary deployment: Deploy schema change to subset of tables
-- Step 1: Create new column (non-blocking)
ALTER TABLE customers ADD COLUMN new_field VARCHAR DEFAULT NULL;

-- Step 2: Migrate data in batches
DO $$
BEGIN
  FOR i IN 1..100 LOOP
    UPDATE customers 
      SET new_field = CONCAT('migrated_', id)
      WHERE new_field IS NULL
      LIMIT 1000;
    COMMIT;
  END LOOP;
END $$;

-- Step 3: Monitor migration progress
SELECT COUNT(*) as remaining 
FROM customers 
WHERE new_field IS NULL;

-- Step 4: Add NOT NULL constraint only after migration complete
ALTER TABLE customers ALTER COLUMN new_field SET NOT NULL;

-- Step 5: Add index for performance
CREATE INDEX idx_new_field ON customers(new_field);

-- Rollback procedure if needed
-- ALTER TABLE customers DROP COLUMN new_field;
```

Q138: How do I implement feature flag-driven deployments?

1. Store feature flags in database.
2. Application reads flags on startup and periodically.
3. Control feature rollout via flag configuration.
4. Implement percentage-based rollout.
5. Monitor feature performance separately.
6. Disable features quickly if issues detected.

Script:
```sql
-- Feature flag schema
CREATE TABLE feature_flags (
  flag_name VARCHAR PRIMARY KEY,
  enabled BOOLEAN DEFAULT false,
  rollout_percentage INT DEFAULT 0,  -- 0-100
  description VARCHAR,
  owner VARCHAR,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);

-- Feature flag values for different environments
CREATE TABLE feature_flag_overrides (
  flag_name VARCHAR,
  environment VARCHAR,  -- dev, staging, production
  enabled BOOLEAN,
  rollout_percentage INT,
  PRIMARY KEY (flag_name, environment)
);

-- Insert feature flags
INSERT INTO feature_flags (flag_name, enabled, rollout_percentage, description)
VALUES 
  ('new_checkout_flow', true, 10, 'New checkout UI rollout'),
  ('experimental_search', false, 0, 'Experimental search algorithm'),
  ('payment_retry_logic', true, 100, 'Improved payment retry');

-- Application-level feature flag evaluation
-- Pseudocode
def is_feature_enabled(feature_name, user_id):
    flag = get_feature_flag(feature_name)
    if not flag.enabled:
        return False
    
    # Percentage-based rollout using user_id hash
    user_bucket = hash(user_id) % 100
    return user_bucket < flag.rollout_percentage
```

Q139: How do I implement blast radius limiting for deployments?

1. Implement canary deployments to small subset first.
2. Use feature flags with percentage-based rollout.
3. Implement automated rollback on error rate increase.
4. Set deployment targets (regions, customer segments).
5. Monitor error rates and key metrics continuously.
6. Document blast radius for each deployment.

Script:
```sql
-- Deployment tracking with blast radius
CREATE TABLE deployments (
  deployment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  service_name VARCHAR,
  version VARCHAR,
  deployment_time TIMESTAMP DEFAULT now(),
  blast_radius VARCHAR,  -- canary, regional, global
  target_nodes INT,
  affected_customers INT,
  rollback_plan VARCHAR,
  status VARCHAR  -- PENDING, IN_PROGRESS, COMPLETED, ROLLED_BACK
);

-- Deployment error tracking
CREATE TABLE deployment_errors (
  error_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  deployment_id UUID REFERENCES deployments(deployment_id),
  error_rate DECIMAL,
  error_type VARCHAR,
  detected_at TIMESTAMP,
  auto_rollback BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT now()
);

-- Auto-rollback on error rate spike
CREATE PROCEDURE check_and_rollback_deployment() AS $$
BEGIN
  -- If error rate > 5% for deployment, trigger rollback
  UPDATE deployments 
  SET status = 'ROLLED_BACK'
  WHERE deployment_id IN (
    SELECT d.deployment_id 
    FROM deployments d
    JOIN deployment_errors de ON d.deployment_id = de.deployment_id
    WHERE de.error_rate > 0.05 
      AND d.status = 'IN_PROGRESS'
  );
END
$$ LANGUAGE plpgsql;
```

Q140: How do I implement post-deployment validation?

1. Run automated test suite immediately after deployment.
2. Perform health checks across all services.
3. Validate data consistency.
4. Check performance metrics against baseline.
5. Monitor error rates and latency.
6. Automated rollback if validation fails.

Script:
```bash
#!/bin/bash
# Post-deployment validation script

echo "=== Post-Deployment Validation ==="

# 1. Health checks
echo "Running health checks..."
for endpoint in /health /metrics /status; do
  response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080$endpoint)
  if [ "$response" != "200" ]; then
    echo "FAILED: Health check $endpoint returned $response"
    exit 1
  fi
done

# 2. Data consistency checks
echo "Validating data consistency..."
cockroach sql --certs-dir=certs <<EOF
-- Check for data inconsistencies
SELECT COUNT(*) as orphaned_records 
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;
EOF

# 3. Performance baseline comparison
echo "Comparing performance metrics..."
current_p99=$(curl -s http://localhost:9090/api/v1/query?query=sql_exec_latency_p99 | jq '.data.result[0].value[1]')
baseline_p99=100000  # 100ms baseline

if (( $(echo "$current_p99 > $baseline_p99 * 1.2" | bc -l) )); then
  echo "WARNING: P99 latency increased by >20%"
  # Potential rollback trigger
fi

# 4. Run smoke tests
echo "Running smoke tests..."
pytest tests/smoke_tests.py -v
if [ $? -ne 0 ]; then
  echo "FAILED: Smoke tests failed"
  exit 1
fi

echo "=== Validation Complete ==="
```

================================================================================
SECTION 24: ADVANCED TROUBLESHOOTING TECHNIQUES
================================================================================

Q141: How do I diagnose CPU utilization spikes?

1. Identify affected nodes through CPU metrics.
2. Correlate CPU spike with query activity.
3. Find expensive queries using statement statistics.
4. Use EXPLAIN ANALYZE to identify bottlenecks.
5. Add indexes or optimize queries.
6. Monitor CPU normalization.

Script:
```sql
-- Identify CPU-intensive queries
SELECT 
  user_name,
  query,
  execution_count,
  cumulative_cpu_time_seconds,
  cumulative_cpu_time_seconds / NULLIF(execution_count, 0) as avg_cpu_per_execution
FROM crdb_internal.node_statement_statistics
ORDER BY cumulative_cpu_time_seconds DESC
LIMIT 10;

-- Analyze CPU spike timing
SELECT 
  DATE_TRUNC('minute', timestamp) as minute,
  COUNT(*) as query_count,
  AVG(execution_time_ms) as avg_latency,
  MAX(execution_time_ms) as max_latency
FROM query_logs
WHERE timestamp > now() - interval '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute DESC;

-- Find correlation with specific operations
SELECT * FROM crdb_internal.node_statement_statistics
WHERE statement LIKE '%JOIN%' OR statement LIKE '%GROUP BY%'
ORDER BY cumulative_cpu_time_seconds DESC;
```

Q142: How do I analyze memory leaks in the cluster?

1. Monitor heap memory allocation over time.
2. Identify trending memory growth.
3. Check for long-running transactions.
4. Query for large result sets in memory.
5. Restart nodes to clear memory if leak confirmed.
6. Profile memory post-restart.

Script:
```sql
-- Monitor memory trends
SELECT 
  DATE_TRUNC('hour', measured_at) as hour,
  AVG(heap_inuse_bytes) / (1024*1024) as avg_heap_mb,
  MAX(heap_inuse_bytes) / (1024*1024) as max_heap_mb,
  MIN(heap_inuse_bytes) / (1024*1024) as min_heap_mb
FROM memory_metrics
WHERE measured_at > now() - interval '7 days'
GROUP BY DATE_TRUNC('hour', measured_at)
ORDER BY hour DESC;

-- Detect growing goroutines (possible leak indicator)
SELECT 
  node_id,
  goroutine_count,
  LAG(goroutine_count) OVER (PARTITION BY node_id ORDER BY measured_at) as prev_count,
  goroutine_count - LAG(goroutine_count) OVER (PARTITION BY node_id ORDER BY measured_at) as growth_rate
FROM goroutine_metrics
ORDER BY measured_at DESC
LIMIT 10;

-- Long-running transactions consuming memory
SELECT 
  session_id,
  user_name,
  statement,
  EXTRACT(EPOCH FROM (now() - session_start)) as seconds_running
FROM crdb_internal.node_sessions
WHERE EXTRACT(EPOCH FROM (now() - session_start)) > 3600  -- Over 1 hour
ORDER BY seconds_running DESC;
```

Q143: How do I troubleshoot connection pool exhaustion?

1. Monitor active connection count.
2. Identify connections not being returned to pool.
3. Check for long-running transactions.
4. Implement connection timeouts.
5. Increase pool size if legitimate demand.
6. Close idle connections.

Script:
```sql
-- Monitor connection count
SELECT 
  user_name,
  COUNT(*) as connection_count,
  STRING_AGG(DISTINCT session_id, ',') as session_ids
FROM crdb_internal.node_sessions
GROUP BY user_name
ORDER BY connection_count DESC;

-- Identify idle connections
SELECT 
  session_id,
  user_name,
  client_address,
  last_active,
  EXTRACT(EPOCH FROM (now() - last_active)) as idle_seconds
FROM crdb_internal.node_sessions
WHERE EXTRACT(EPOCH FROM (now() - last_active)) > 300  -- Idle > 5 min
ORDER BY idle_seconds DESC;

-- Kill idle connections
SELECT pg_terminate_backend(session_id)
FROM crdb_internal.node_sessions
WHERE EXTRACT(EPOCH FROM (now() - last_active)) > 1800  -- Idle > 30 min
  AND session_id != current_session_id();

-- Configure connection limits per user
ALTER ROLE app_user CONNECTION LIMIT 50;
ALTER ROLE analytics_user CONNECTION LIMIT 20;
```

Q144: How do I diagnose query plan regression?

1. Capture baseline execution plans for critical queries.
2. Compare current plans against baselines.
3. Identify plan changes causing latency increase.
4. Check table statistics accuracy.
5. Rerun ANALYZE if statistics stale.
6. Force plan or adjust query if regression confirmed.

Script:
```sql
-- Capture baseline query plans
CREATE TABLE query_plan_baseline (
  query_id UUID,
  query_text VARCHAR,
  plan_hash VARCHAR,
  plan_text VARCHAR,
  captured_at TIMESTAMP DEFAULT now(),
  PRIMARY KEY (query_id, captured_at)
);

-- Capture current plan
EXPLAIN (VERBOSE, FORMAT JSON) 
  SELECT * FROM orders WHERE customer_id = $1 AND created_at > now() - interval '30 days'
INTO @current_plan;

-- Compare plans
WITH current AS (
  SELECT @current_plan as plan_text
),
baseline AS (
  SELECT plan_text FROM query_plan_baseline 
  WHERE query_text LIKE '%orders%customer_id%'
  ORDER BY captured_at DESC LIMIT 1
)
SELECT 
  CASE WHEN c.plan_text = b.plan_text THEN 'SAME' ELSE 'CHANGED' END as plan_status,
  b.plan_text as baseline_plan,
  c.plan_text as current_plan
FROM current c, baseline b;

-- Update statistics if stale
ANALYZE TABLE orders;

-- Force specific plan if regression confirmed
SET CLUSTER SETTING sql.defaults.optimizer_use_forced_plans = true;
-- Use plan hints in queries:
-- SELECT /*+ FORCE_ZIGZAG_JOIN(orders) */ * FROM orders WHERE ...
```

Q145: How do I analyze transaction conflict patterns?

1. Monitor transaction abort rates.
2. Identify hotspot tables causing conflicts.
3. Analyze transaction patterns causing conflicts.
4. Implement row locking for highly contended data.
5. Adjust isolation levels if appropriate.
6. Optimize transaction duration.

Script:
```sql
-- Monitor transaction abort rates
SELECT 
  DATE_TRUNC('minute', timestamp) as minute,
  COUNT(*) as total_transactions,
  SUM(CASE WHEN status = 'ABORTED' THEN 1 ELSE 0 END) as aborted_transactions,
  ROUND(100.0 * SUM(CASE WHEN status = 'ABORTED' THEN 1 ELSE 0 END) / COUNT(*), 2) as abort_rate_percent
FROM transaction_log
WHERE timestamp > now() - interval '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute DESC;

-- Identify hotspot tables
SELECT 
  table_name,
  COUNT(*) as conflict_count,
  AVG(transaction_duration_ms) as avg_duration
FROM transaction_conflicts
WHERE timestamp > now() - interval '24 hours'
GROUP BY table_name
ORDER BY conflict_count DESC
LIMIT 10;

-- Analyze conflict patterns
SELECT 
  t1.table_name,
  COUNT(*) as conflict_count,
  STRING_AGG(DISTINCT t1.column_name, ', ') as conflicted_columns
FROM transaction_conflicts t1
WHERE timestamp > now() - interval '24 hours'
GROUP BY t1.table_name
ORDER BY conflict_count DESC;

-- Implement explicit locking for hotspots
BEGIN;
SELECT * FROM inventory_table 
WHERE product_id = $1 
FOR UPDATE;

UPDATE inventory_table 
SET quantity = quantity - $2 
WHERE product_id = $1;

COMMIT;
```

================================================================================
SECTION 25: PRODUCTION HARDENING CHECKLIST
================================================================================

Q146: What is comprehensive production readiness checklist?

1. Cluster configuration: 3+ nodes, odd-numbered, different regions/zones.
2. Backup strategy: automated daily full + hourly incremental with tested recovery.
3. Monitoring: Prometheus/Grafana with key metrics and alerting configured.
4. Security: TLS enabled, user roles configured, audit logging active.
5. Disaster recovery: documented procedures, tested failover, RTO/RPO documented.
6. Capacity planning: cluster sized for 1.5x peak load, growth projections tracked.

Checklist Script:
```bash
#!/bin/bash
# Production readiness checklist

echo "=== CockroachDB Production Readiness Checklist ==="

# 1. Cluster configuration
echo "✓ Checking cluster configuration..."
NODE_COUNT=$(cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM crdb_internal.nodes" 2>/dev/null)
echo "  Node count: $NODE_COUNT"
[ "$NODE_COUNT" -ge 3 ] && echo "  ✓ Minimum 3 nodes" || echo "  ✗ FAIL: Less than 3 nodes"

# 2. Backup configuration
echo "✓ Checking backup configuration..."
cockroach sql --certs-dir=certs -c "SELECT * FROM system.scheduled_jobs WHERE schedule_name LIKE '%backup%';" 2>/dev/null | grep -q "." && echo "  ✓ Backups configured" || echo "  ✗ FAIL: No backups configured"

# 3. Monitoring
echo "✓ Checking monitoring setup..."
curl -s http://localhost:8080/_status/vars > /dev/null && echo "  ✓ Metrics endpoint available" || echo "  ✗ FAIL: Metrics endpoint unreachable"

# 4. Security
echo "✓ Checking security configuration..."
ls /path/to/certs/*.crt > /dev/null 2>&1 && echo "  ✓ TLS certificates present" || echo "  ✗ FAIL: TLS certificates missing"

# 5. Disaster Recovery
echo "✓ Checking DR documentation..."
[ -f "DR_PROCEDURES.md" ] && echo "  ✓ DR procedures documented" || echo "  ✗ FAIL: DR procedures not documented"

# 6. Capacity Planning
echo "✓ Checking capacity..."
DISK_USAGE=$(cockroach sql --certs-dir=certs -c "SELECT ROUND(100.0*capacity_used/capacity_total, 2) FROM crdb_internal.stores LIMIT 1" 2>/dev/null)
echo "  Disk usage: $DISK_USAGE%"
[ "${DISK_USAGE%.*}" -lt 70 ] && echo "  ✓ Disk usage acceptable" || echo "  ✗ WARNING: Disk usage high"

echo "=== Checklist Complete ==="
```

Q147: What are top 10 performance optimization priorities?

1. Index all frequently filtered columns in WHERE clauses.
2. Implement connection pooling for all applications.
3. Monitor and optimize slow queries (P99 latency).
4. Implement caching for frequently accessed data.
5. Use batch operations instead of individual statements.
6. Verify table statistics accuracy (run ANALYZE).
7. Monitor replication lag and address bottlenecks.
8. Implement query timeouts to prevent runaway queries.
9. Denormalize data for frequently joined tables.
10. Monitor and tune cluster settings for workload.

Q148: What are top 10 security hardening priorities?

1. Enable TLS for all connections (--certs-dir configuration).
2. Implement role-based access control (RBAC).
3. Enable audit logging for all SQL statements.
4. Implement strong password policies and rotation.
5. Restrict network access via firewall rules.
6. Implement encryption at rest (CMEK) for sensitive data.
7. Monitor and alert on suspicious access patterns.
8. Implement change management for critical operations.
9. Regular security scans and vulnerability assessments.
10. Maintain comprehensive audit logs for compliance.

Q149: What are top 10 operational resilience priorities?

1. Implement automated backup strategy with verified recovery.
2. Set up health monitoring and alerting for all critical metrics.
3. Implement automatic failover procedures.
4. Document and test disaster recovery procedures quarterly.
5. Implement connection draining for graceful shutdowns.
6. Monitor replica lag and implement automatic healing.
7. Implement resource quotas to prevent exhaustion.
8. Set up incident response procedures and runbooks.
9. Implement rate limiting and circuit breakers.
10. Regular capacity planning and infrastructure scaling.

Q150: What are common production issues and quick fixes?

1. High latency: Check for slow queries, add indexes, monitor CPU/disk I/O.
2. Disk full: Archive old data, clean up temporary tables, expand storage.
3. Connection pool exhaustion: Check for idle connections, increase pool size.
4. Memory spike: Kill long-running transactions, increase node memory.
5. Replica lag: Check network latency, monitor disk I/O, add capacity.
6. Failed backups: Verify storage access/credentials, check network connectivity.
7. Quorum loss: Restore from backup or bring nodes back online.
8. Data corruption: Restore from backup, verify checksums.
9. Certificate expiration: Regenerate certificates, deploy to all nodes.
10. Clock skew: Verify NTP synchronization, check system time.

================================================================================

COCKROACHDB ADMINISTRATION GUIDE - ADVANCED TOPICS
Questions Q151-Q250 covering Database Design, Optimization, Scaling, and Enterprise Features

================================================================================
SECTION 26: ADVANCED DATABASE DESIGN PATTERNS
================================================================================

Q151: How do I design schemas for multi-tenancy with data isolation?

1. Create separate databases per tenant for complete isolation.
2. Use tenant_id column in all tables for application-level filtering.
3. Create unique indexes on (tenant_id, business_key) for data integrity.
4. Implement views filtering by tenant_id for security.
5. Use row-level security triggers to enforce tenant boundaries.
6. Test data isolation regularly to prevent leakage.

Script:
```sql
-- Multi-tenant schema design
CREATE DATABASE tenant_db;

CREATE TABLE organizations (
  org_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR,
  UNIQUE(tenant_id, org_id)
);

CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  org_id UUID NOT NULL,
  email VARCHAR,
  UNIQUE(tenant_id, email),
  FOREIGN KEY (tenant_id, org_id) REFERENCES organizations(tenant_id, org_id)
);

CREATE INDEX idx_users_tenant ON users(tenant_id, user_id);

-- Security view for current tenant
CREATE VIEW current_tenant_users AS
SELECT user_id, email FROM users
WHERE tenant_id = current_setting('app.tenant_id')::UUID;

-- Enforce tenant isolation
CREATE POLICY tenant_isolation_policy ON users
  USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

Q152: How do I implement hierarchical data structures efficiently?

1. Use nested set model for tree traversal efficiency.
2. Use adjacency list model for simple parent-child relationships.
3. Use materialized path for balanced performance.
4. Create indexes on common access patterns.
5. Implement recursive CTEs for hierarchical queries.
6. Monitor query performance and adjust model based on usage.

Script:
```sql
-- Nested Set Model (efficient tree traversal)
CREATE TABLE categories (
  id UUID PRIMARY KEY,
  name VARCHAR,
  lft INT,  -- Left boundary
  rgt INT,  -- Right boundary
  UNIQUE(lft, rgt)
);

CREATE INDEX idx_categories_lft_rgt ON categories(lft, rgt);

-- Get all descendants
SELECT * FROM categories WHERE lft > $parent_lft AND rgt < $parent_rgt;

-- Materialized Path (balanced performance)
CREATE TABLE departments (
  id UUID PRIMARY KEY,
  name VARCHAR,
  path VARCHAR,  -- e.g., '1.2.3.4' for nested depth
  parent_id UUID
);

CREATE INDEX idx_departments_path ON departments(path);

-- Recursive CTE for hierarchical queries
WITH RECURSIVE dept_hierarchy AS (
  SELECT id, name, parent_id, 1 as depth
  FROM departments
  WHERE parent_id IS NULL
  
  UNION ALL
  
  SELECT d.id, d.name, d.parent_id, dh.depth + 1
  FROM departments d
  JOIN dept_hierarchy dh ON d.parent_id = dh.id
)
SELECT * FROM dept_hierarchy ORDER BY depth, name;
```

Q153: How do I handle temporal data (time-series and versioning)?

1. Use timestamp columns with DEFAULT NOW() for audit trails.
2. Implement soft deletes with deleted_at timestamp.
3. Create version tables for historical tracking.
4. Use PARTITION BY RANGE for time-series data.
5. Implement temporal queries using AS OF SYSTEM TIME.
6. Archive old time-series data to reduce active storage.

Script:
```sql
-- Temporal data with versioning
CREATE TABLE products (
  product_id UUID PRIMARY KEY,
  version INT DEFAULT 1,
  name VARCHAR,
  price DECIMAL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP DEFAULT NULL
);

-- Product version history
CREATE TABLE product_versions (
  version_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL,
  version INT NOT NULL,
  name VARCHAR,
  price DECIMAL,
  changed_by VARCHAR,
  changed_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(product_id, version),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Time-series events with partitioning
CREATE TABLE events (
  event_id UUID,
  timestamp TIMESTAMP NOT NULL,
  event_type VARCHAR NOT NULL,
  data JSONB,
  PRIMARY KEY (event_id, timestamp)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01'),
  PARTITION p_2024_03 VALUES FROM ('2024-03-01') TO ('2024-04-01')
);

-- Audit trail with soft deletes
CREATE TRIGGER product_version_trigger
AFTER UPDATE ON products
FOR EACH ROW
INSERT INTO product_versions (product_id, version, name, price, changed_by)
VALUES (NEW.product_id, NEW.version, NEW.name, NEW.price, current_user);

-- Query historical data
SELECT * FROM products AS OF SYSTEM TIME '2024-01-15 10:00:00'
WHERE product_id = $1;
```

Q154: How do I implement efficient pagination for large result sets?

1. Use offset-limit for simplicity (avoid for very large offsets).
2. Use cursor-based pagination (keyset pagination) for efficiency.
3. Implement value-based pagination for real-time data.
4. Create indexes on pagination columns.
5. Use stable sort order to prevent missing records.
6. Monitor pagination performance.

Script:
```sql
-- Offset-limit pagination (simple, less efficient for large offsets)
SELECT * FROM orders 
WHERE status = 'completed'
ORDER BY created_at DESC
LIMIT 50 OFFSET $page * 50;

-- Cursor-based pagination (efficient, stable)
-- First query: get first 50
SELECT id, name, created_at FROM products
ORDER BY id ASC
LIMIT 50;

-- Next queries: get after last ID
SELECT id, name, created_at FROM products
WHERE id > $last_cursor_id
ORDER BY id ASC
LIMIT 50;

-- Value-based pagination (for dynamic data)
SELECT id, name, updated_at FROM products
WHERE (updated_at, id) > ($last_timestamp, $last_id)
ORDER BY updated_at ASC, id ASC
LIMIT 50;

-- Create indexes for pagination
CREATE INDEX idx_orders_created_status ON orders(status, created_at, id);
CREATE INDEX idx_products_id ON products(id);
```

Q155: How do I handle denormalization strategies for performance?

1. Identify expensive joins causing latency.
2. Denormalize selectively: keep related data together.
3. Create materialized views for frequently computed values.
4. Implement refresh schedule for materialized views.
5. Monitor data staleness and adjust refresh frequency.
6. Document denormalization trade-offs.

Script:
```sql
-- Identify expensive joins
EXPLAIN ANALYZE
SELECT o.id, o.total, c.name, c.email, a.city
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN addresses a ON c.address_id = a.id
WHERE o.created_at > now() - interval '30 days';

-- Denormalize customer data into orders
ALTER TABLE orders ADD COLUMN customer_name VARCHAR;
ALTER TABLE orders ADD COLUMN customer_email VARCHAR;

UPDATE orders o
SET customer_name = c.name, customer_email = c.email
FROM customers c
WHERE o.customer_id = c.id;

-- Materialized view for expensive aggregation
CREATE MATERIALIZED VIEW order_summary AS
SELECT 
  customer_id,
  COUNT(*) as order_count,
  SUM(total) as total_spent,
  AVG(total) as avg_order_value,
  MAX(created_at) as last_order_date
FROM orders
GROUP BY customer_id;

CREATE INDEX idx_order_summary_customer ON order_summary(customer_id);

-- Refresh schedule
CREATE SCHEDULE refresh_order_summary FOR 
  REFRESH MATERIALIZED VIEW order_summary
  RECURRING EVERY 1 hour;

-- Query using materialized view
SELECT os.*, c.name, c.email
FROM order_summary os
JOIN customers c ON os.customer_id = c.id
WHERE os.total_spent > 10000;
```

================================================================================
SECTION 27: ADVANCED QUERY OPTIMIZATION
================================================================================

Q156: How do I optimize queries with complex joins?

1. Identify join selectivity: which predicates reduce rows most.
2. Reorder joins to process most selective predicates first.
3. Add indexes on join columns for efficient lookups.
4. Use EXPLAIN ANALYZE to verify join order.
5. Consider denormalizing highly joined tables.
6. Break complex queries into smaller CTEs if optimizer struggles.

Script:
```sql
-- Analyze join selectivity
EXPLAIN ANALYZE
SELECT o.*, c.*, p.*
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE c.country = 'US' AND p.category = 'Electronics' AND o.created_at > now() - interval '30 days';

-- Reorder joins for efficiency (filter first, then join)
WITH filtered_orders AS (
  SELECT * FROM orders
  WHERE created_at > now() - interval '30 days'
),
us_customers AS (
  SELECT * FROM customers
  WHERE country = 'US'
),
electronics AS (
  SELECT * FROM products
  WHERE category = 'Electronics'
)
SELECT fo.*, uc.*, e.*
FROM filtered_orders fo
JOIN us_customers uc ON fo.customer_id = uc.id
JOIN electronics e ON fo.product_id = e.id;

-- Create indexes for join efficiency
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);
CREATE INDEX idx_orders_product ON orders(product_id);
CREATE INDEX idx_customers_country ON customers(country);
CREATE INDEX idx_products_category ON products(category);

-- Use UNION for disjoint conditions
SELECT * FROM orders WHERE status = 'completed' AND total > 1000
UNION ALL
SELECT * FROM orders WHERE vip_customer = true AND total > 500;
```

Q157: How do I optimize aggregation queries?

1. Use GROUP BY instead of DISTINCT when possible.
2. Create indexes on grouping columns.
3. Filter before aggregating to reduce data processed.
4. Use window functions for running totals.
5. Implement partial aggregates to reduce computation.
6. Consider materialized views for frequently aggregated data.

Script:
```sql
-- Efficient aggregation with early filtering
SELECT 
  customer_id,
  COUNT(*) as order_count,
  SUM(total) as total_spent,
  AVG(total) as avg_value,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY total) as median_value
FROM orders
WHERE created_at > now() - interval '1 year'
  AND status = 'completed'
  AND customer_id NOT IN (SELECT id FROM test_customers)
GROUP BY customer_id
HAVING COUNT(*) > 10
ORDER BY total_spent DESC;

-- Window functions for running totals
SELECT 
  date,
  revenue,
  SUM(revenue) OVER (ORDER BY date) as cumulative_revenue,
  SUM(revenue) OVER (ORDER BY date ROWS BETWEEN 7 PRECEDING AND CURRENT ROW) as rolling_7day_revenue,
  ROW_NUMBER() OVER (ORDER BY revenue DESC) as rank,
  PERCENT_RANK() OVER (ORDER BY revenue DESC) as percentile
FROM daily_revenue
ORDER BY date DESC;

-- Partial aggregates with CTEs
WITH monthly_totals AS (
  SELECT 
    DATE_TRUNC('month', created_at)::date as month,
    SUM(total) as monthly_total,
    COUNT(*) as order_count
  FROM orders
  GROUP BY DATE_TRUNC('month', created_at)
)
SELECT 
  month,
  monthly_total,
  SUM(monthly_total) OVER (ORDER BY month) as ytd_total,
  AVG(monthly_total) OVER (ORDER BY month ROWS BETWEEN 12 PRECEDING AND CURRENT ROW) as trailing_12m_avg
FROM monthly_totals;

-- Create indexes for aggregation
CREATE INDEX idx_orders_date_status ON orders(created_at, status) INCLUDE (total, customer_id);
```

Q158: How do I optimize subquery performance?

1. Flatten subqueries into joins when possible.
2. Use EXISTS instead of IN for correlated subqueries.
3. Use scalar subqueries sparingly.
4. Move subqueries to CTEs for readability and optimization.
5. Use EXPLAIN to verify subquery execution.
6. Implement temporary tables if subquery is executed many times.

Script:
```sql
-- Inefficient: subquery in SELECT clause
SELECT id, name, 
  (SELECT COUNT(*) FROM orders WHERE customer_id = customers.id) as order_count
FROM customers;

-- Efficient: join instead
SELECT c.id, c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Inefficient: IN with subquery
SELECT * FROM orders
WHERE customer_id IN (SELECT id FROM vip_customers);

-- Efficient: EXISTS
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM vip_customers vc WHERE vc.id = o.customer_id);

-- Use CTE instead of subquery
WITH active_customers AS (
  SELECT id FROM customers
  WHERE created_at > now() - interval '90 days'
    AND last_purchase > now() - interval '30 days'
)
SELECT o.* FROM orders o
WHERE o.customer_id IN (SELECT id FROM active_customers);

-- Temporary table for repeated subquery
CREATE TEMP TABLE temp_active_customers AS
SELECT id FROM customers
WHERE created_at > now() - interval '90 days'
  AND last_purchase > now() - interval '30 days';

SELECT o.* FROM orders o
WHERE o.customer_id IN (SELECT id FROM temp_active_customers);

SELECT COUNT(*) FROM temp_active_customers;  -- Reuse temp table
```

Q159: How do I optimize full-text search queries?

1. Create full-text search indexes on text columns.
2. Use TSVECTOR and TSQUERY for efficient searching.
3. Implement phrase search with proper operators.
4. Use AND/OR/NOT operators to refine results.
5. Rank results by relevance for better UX.
6. Monitor search performance and adjust indexes.

Script:
```sql
-- Create full-text search index
CREATE TABLE documents (
  doc_id UUID PRIMARY KEY,
  title VARCHAR,
  content TEXT,
  search_vector TSVECTOR
);

-- Generate search vector from content
UPDATE documents 
SET search_vector = to_tsvector('english', title || ' ' || content);

CREATE INDEX idx_documents_search ON documents USING GIN (search_vector);

-- Full-text search query
SELECT doc_id, title, rank
FROM documents
WHERE search_vector @@ to_tsquery('english', 'data & (analytics | warehouse)')
ORDER BY rank DESC
LIMIT 20;

-- Phrase search
SELECT doc_id, title
FROM documents
WHERE search_vector @@ phraseto_tsquery('english', 'data warehouse management')
ORDER BY ts_rank(search_vector, phraseto_tsquery('english', 'data warehouse management')) DESC;

-- Complex search with ranking
SELECT 
  doc_id, 
  title,
  ts_rank(search_vector, query) as rank,
  ts_headline('english', content, query, 'StartSel=<b>, StopSel=</b>') as snippet
FROM documents, 
     to_tsquery('english', 'database & (optimization | performance)') as query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

Q160: How do I handle query result caching effectively?

1. Identify queries with expensive computation but stable results.
2. Implement application-level caching (Redis, Memcached).
3. Set appropriate TTL based on data freshness requirements.
4. Implement cache invalidation on data changes.
5. Monitor cache hit/miss ratios.
6. Use query IDs or query fingerprints for cache keys.

Script:
```sql
-- Identify expensive queries for caching
SELECT 
  query,
  execution_count,
  cumulative_cpu_time_seconds,
  cumulative_latency_seconds / execution_count as avg_latency
FROM crdb_internal.node_statement_statistics
WHERE query NOT LIKE '%system%'
ORDER BY cumulative_latency_seconds DESC
LIMIT 20;

-- Query result caching with expiration
CREATE TABLE query_cache (
  cache_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_fingerprint VARCHAR,
  query_text VARCHAR,
  result_json JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  hit_count INT DEFAULT 0
);

-- Cache lookup
SELECT result_json, hit_count + 1 as new_hit_count
FROM query_cache
WHERE query_fingerprint = $query_fingerprint
  AND expires_at > NOW();

-- Cache invalidation trigger
CREATE TRIGGER invalidate_cache_on_insert
AFTER INSERT ON products
FOR EACH ROW
DELETE FROM query_cache
WHERE query_fingerprint IN ('products_by_category', 'featured_products')
  AND expires_at > NOW();

-- Warm cache on startup
INSERT INTO query_cache (query_fingerprint, query_text, result_json, expires_at)
SELECT 
  'top_products',
  'SELECT * FROM products ORDER BY sales DESC LIMIT 100',
  to_jsonb((SELECT array_agg(p) FROM products p ORDER BY p.sales DESC LIMIT 100)),
  NOW() + interval '1 hour';
```

================================================================================
SECTION 28: ADVANCED SCALING STRATEGIES
================================================================================

Q161: How do I implement horizontal scaling with application-level sharding?

1. Choose sharding key (customer_id, tenant_id, region).
2. Implement consistent hashing for shard distribution.
3. Create metadata service to track shard locations.
4. Handle cross-shard queries with fan-out and merge.
5. Implement shard rebalancing for growth.
6. Test shard routing thoroughly.

Script:
```python
# Application-level sharding with consistent hashing
import hashlib

class ShardManager:
    def __init__(self, num_shards):
        self.num_shards = num_shards
        self.shard_nodes = {}  # Map shard to node
    
    def get_shard(self, key):
        """Get shard for a given key"""
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards
    
    def get_connection(self, key):
        """Get database connection for sharded key"""
        shard_id = self.get_shard(key)
        node_addr = self.shard_nodes.get(shard_id, 'localhost')
        return self.create_connection(node_addr)
    
    def query_all_shards(self, query):
        """Fan-out query across all shards"""
        results = []
        for shard_id in range(self.num_shards):
            conn = self.get_connection(shard_id)
            result = conn.execute(query)
            results.extend(result)
        return results
    
    def create_connection(self, node):
        # Implementation to create DB connection
        pass

# Usage
shard_manager = ShardManager(num_shards=16)

# Single-shard query (fast)
customer_id = 123
conn = shard_manager.get_connection(customer_id)
orders = conn.execute("SELECT * FROM orders WHERE customer_id = ?", customer_id)

# Multi-shard query (slower, fan-out)
all_orders = shard_manager.query_all_shards("SELECT COUNT(*) as total FROM orders")
```

SQL Sharding Setup:
```sql
-- Sharded table schema
CREATE TABLE orders (
  order_id UUID,
  customer_id UUID NOT NULL,  -- Sharding key
  amount DECIMAL,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (customer_id, order_id)  -- Shard key first
);

-- Shard metadata
CREATE TABLE shard_metadata (
  shard_id INT PRIMARY KEY,
  range_start VARCHAR,
  range_end VARCHAR,
  node_address VARCHAR,
  status VARCHAR  -- ACTIVE, MIGRATING, INACTIVE
);

-- Example sharding configuration
INSERT INTO shard_metadata VALUES
  (0, '0000', '0fff', 'shard-0.example.com', 'ACTIVE'),
  (1, '1000', '1fff', 'shard-1.example.com', 'ACTIVE'),
  (2, '2000', '2fff', 'shard-2.example.com', 'ACTIVE'),
  (3, '3000', '3fff', 'shard-3.example.com', 'ACTIVE');
```

Q162: How do I implement read scaling with read replicas and follower reads?

1. Deploy read replicas in separate regions.
2. Use --locality to control replica placement.
3. Enable follower reads for local read scaling.
4. Implement routing logic to read from nearest replica.
5. Monitor replication lag.
6. Handle eventual consistency in application.

Script:
```sql
-- Enable follower reads in application
SET SESSION enable_follower_reads = true;

-- Query using follower read timestamp
SELECT * FROM orders 
WHERE customer_id = $1
ORDER BY created_at DESC
LIMIT 100;

-- Zone configuration for read replica placement
ALTER TABLE orders CONFIGURE ZONE USING 
  num_replicas = 3,
  constraints = '[+region=us-west]',
  lease_preferences = '[[+region=us-west]]';

-- Monitor follower read usage
SELECT 
  COUNT(*) as total_queries,
  SUM(CASE WHEN enable_follower_reads THEN 1 ELSE 0 END) as follower_read_queries
FROM crdb_internal.node_statement_statistics;

-- Implement application-level follower read routing
-- Application checks data freshness requirements:
-- - Strong consistency: use regular read
-- - Eventual consistency OK: use follower read
```

Application Code (pseudocode):
```python
def get_user_profile(user_id, require_fresh=False):
    if require_fresh:
        # Use strong consistent read from primary
        query = "SELECT * FROM users WHERE id = $1"
        return db.query(query, user_id, consistency='strong')
    else:
        # Use follower read for better latency
        query = "SELECT * FROM users WHERE id = $1"
        return db.query(query, user_id, enable_follower_reads=True)

def get_order_history(user_id):
    # Order history doesn't need to be perfectly fresh
    return get_data_with_follower_read(
        "SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC",
        user_id
    )

def process_payment(order_id):
    # Payment processing needs strong consistency
    return get_user_profile(order_id, require_fresh=True)
```

Q163: How do I handle geographic data distribution and locality?

1. Design for data residency requirements (GDPR, compliance).
2. Use --locality flag to specify region/zone/rack.
3. Configure zone constraints for data placement.
4. Implement region-specific databases if needed.
5. Monitor cross-region traffic and optimize.
6. Use follower reads for local queries.

Script:
```bash
# Start nodes with geographic locality
cockroach start \
  --locality=region=us-west,zone=us-west-1a,rack=1 \
  --join=node1:26257 \
  --certs-dir=certs

cockroach start \
  --locality=region=eu-central,zone=eu-central-1a,rack=1 \
  --join=node1:26257 \
  --certs-dir=certs
```

SQL Configuration:
```sql
-- US data stays in US regions
ALTER TABLE us_customers CONFIGURE ZONE USING 
  constraints = '[+region=us-west, +region=us-east]',
  lease_preferences = '[[+region=us-west]]';

-- EU data stays in EU regions (GDPR)
ALTER TABLE eu_customers CONFIGURE ZONE USING 
  constraints = '[+region=eu-west, +region=eu-central]',
  lease_preferences = '[[+region=eu-central]]';

-- Monitor geographic distribution
SELECT 
  range_id,
  table_name,
  zone_config,
  COUNT(*) as replica_count
FROM crdb_internal.ranges
GROUP BY range_id, table_name, zone_config;

-- Calculate cross-region traffic costs
SELECT 
  source_region,
  dest_region,
  SUM(bytes_transferred) as total_bytes,
  SUM(bytes_transferred) * 0.02 as estimated_cost_usd  -- $0.02 per GB
FROM cross_region_traffic
WHERE timestamp > now() - interval '1 month'
GROUP BY source_region, dest_region
ORDER BY estimated_cost_usd DESC;
```

Q164: How do I manage table growth and implement archival strategies?

1. Monitor table growth rates continuously.
2. Identify old data candidates for archival.
3. Implement partitioning by date for time-series data.
4. Archive to cheaper storage (S3, GCS).
5. Implement tiered retention policies.
6. Monitor active storage size.

Script:
```sql
-- Monitor table growth
CREATE TABLE table_size_metrics (
  table_name VARCHAR,
  size_bytes BIGINT,
  row_count BIGINT,
  measured_at TIMESTAMP DEFAULT NOW()
);

-- Collect metrics
INSERT INTO table_size_metrics (table_name, size_bytes, row_count)
SELECT 
  table_name,
  pg_table_size(table_name::regclass),
  COUNT(*)
FROM information_schema.tables
LEFT JOIN (SELECT * FROM users UNION SELECT * FROM orders) ON true
GROUP BY table_name;

-- Identify tables for archival
SELECT 
  table_name,
  size_bytes / (1024*1024*1024) as size_gb,
  row_count,
  CASE 
    WHEN size_bytes > 10*1024*1024*1024 THEN 'ARCHIVE_CANDIDATE'
    WHEN size_bytes > 5*1024*1024*1024 THEN 'PARTITION_CANDIDATE'
    ELSE 'RETAIN'
  END as action
FROM table_size_metrics
WHERE measured_at = (SELECT MAX(measured_at) FROM table_size_metrics)
ORDER BY size_bytes DESC;

-- Archive old transactions
CREATE TABLE transactions_archive AS
SELECT * FROM transactions
WHERE created_at < now() - interval '7 years';

-- Export to cold storage
EXPORT (SELECT * FROM transactions_archive)
TO 's3://archive-bucket/transactions/2017/';

-- Delete after confirming archive
DELETE FROM transactions 
WHERE created_at < now() - interval '7 years';

-- Partition large table
CREATE TABLE events_partitioned (
  event_id UUID,
  timestamp TIMESTAMP NOT NULL,
  event_type VARCHAR,
  data JSONB,
  PRIMARY KEY (event_id, timestamp)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01')
);
```

Q165: How do I implement distributed transactions across shards?

1. Two-phase commit for strong consistency across shards.
2. Saga pattern for eventual consistency.
3. Implement compensation transactions for rollback.
4. Test failure scenarios thoroughly.
5. Monitor distributed transaction latency.
6. Document consistency guarantees.

Script:
```sql
-- Two-phase commit simulation across shards
-- Prepare phase: validate changes on all shards
BEGIN TRANSACTION;

-- Shard 1: Prepare
SAVEPOINT shard1_prepare;
UPDATE accounts SET balance = balance - 100 WHERE id = $customer_id_shard1 AND balance >= 100;

-- Shard 2: Prepare
SAVEPOINT shard2_prepare;
UPDATE accounts SET balance = balance + 100 WHERE id = $customer_id_shard2;

-- If all prepared successfully, commit
COMMIT;

-- Saga pattern for distributed transactions
CREATE TABLE distributed_transaction_log (
  saga_id UUID PRIMARY KEY,
  status VARCHAR,  -- INITIATED, SHARD1_DONE, SHARD2_DONE, COMPLETED, COMPENSATING, COMPENSATED
  transaction_data JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Step 1: Debit from shard 1
BEGIN;
UPDATE shard1.accounts SET balance = balance - $amount WHERE id = $from_id RETURNING *;
UPDATE distributed_transaction_log SET status = 'SHARD1_DONE' WHERE saga_id = $saga_id;
COMMIT;

-- Step 2: Credit to shard 2
BEGIN;
UPDATE shard2.accounts SET balance = balance + $amount WHERE id = $to_id RETURNING *;
UPDATE distributed_transaction_log SET status = 'SHARD2_DONE' WHERE saga_id = $saga_id;
COMMIT;

-- If step 2 fails, compensate step 1
BEGIN;
UPDATE shard1.accounts SET balance = balance + $amount WHERE id = $from_id;
UPDATE distributed_transaction_log SET status = 'COMPENSATED' WHERE saga_id = $saga_id;
COMMIT;
```

================================================================================
SECTION 29: ADVANCED ENTERPRISE FEATURES
================================================================================

Q166: How do I implement multi-tenancy with complete data isolation?

1. Separate databases per tenant for complete isolation.
2. Separate backup/restore per tenant.
3. Implement audit trails per tenant.
4. Implement resource quotas per tenant.
5. Monitor cross-tenant performance issues.
6. Test isolation regularly.

Script:
```sql
-- Create isolated tenant infrastructure
CREATE DATABASE tenant_acme;
CREATE DATABASE tenant_widgets;

-- Per-tenant users
CREATE USER tenant_acme_admin WITH PASSWORD 'secure_pass_1' IN DATABASE tenant_acme;
CREATE USER tenant_widgets_admin WITH PASSWORD 'secure_pass_2' IN DATABASE tenant_widgets;

-- Per-tenant audit tables
CREATE TABLE tenant_acme.audit_log (
  log_id UUID PRIMARY KEY,
  user_name VARCHAR,
  action VARCHAR,
  timestamp TIMESTAMP DEFAULT NOW()
);

-- Per-tenant backups
CREATE SCHEDULE acme_backup FOR 
  BACKUP DATABASE tenant_acme 
  INTO 's3://backups/tenant_acme-{date-time}'
  RECURRING EVERY 1 day
  WITH RETENTION 30d;

CREATE SCHEDULE widgets_backup FOR 
  BACKUP DATABASE tenant_widgets 
  INTO 's3://backups/tenant_widgets-{date-time}'
  RECURRING EVERY 1 day
  WITH RETENTION 30d;

-- Per-tenant resource quotas
ALTER ROLE tenant_acme_admin CONNECTION LIMIT 100;
ALTER ROLE tenant_widgets_admin CONNECTION LIMIT 50;
```

Q167: How do I implement cross-cluster replication for disaster recovery?

1. Physical cluster replication for zero-copy failover.
2. Logical replication via changefeeds for integration.
3. Monitor replication status continuously.
4. Test failover procedures regularly.
5. Implement automatic promotion procedures.
6. Document RTO/RPO for each cluster.

Script:
```sql
-- Physical cluster replication setup
ALTER CLUSTER SET REPLICATION FACTOR = 2;

-- Monitor replication status
SELECT 
  cluster_name,
  status,
  replicated_time,
  NOW() - replicated_time as replication_lag_seconds
FROM crdb_internal.cluster_replication_status;

-- Logical replication via changefeeds
CREATE CHANGEFEED FOR TABLE orders, customers
INTO 'kafka://kafka-broker:9092/replication-topic'
WITH format='avro', confluent_schema_registry='http://registry:8081';

-- Failover procedure
BEGIN;
-- Step 1: Verify primary cluster unreachable
-- (external health check)

-- Step 2: Promote standby cluster
ALTER CLUSTER SET REPLICATION FACTOR = 1;

-- Step 3: Verify promoted cluster operational
SELECT COUNT(*) FROM orders;

-- Step 4: Update connection strings
-- (application configuration)

COMMIT;
```

Q168: How do I implement advanced access control with column-level permissions?

1. Create views limiting columns based on role.
2. Use row-level security for row-level access.
3. Implement dynamic views based on user context.
4. Audit column-level access.
5. Test access control thoroughly.
6. Document permission matrix.

Script:
```sql
-- Column-level access control via views
CREATE TABLE employees (
  emp_id UUID PRIMARY KEY,
  name VARCHAR,
  email VARCHAR,
  salary DECIMAL,
  ssn VARCHAR,
  manager_id UUID
);

-- HR view (can see all columns)
CREATE VIEW employees_hr_view AS
SELECT emp_id, name, email, salary, ssn, manager_id
FROM employees;

-- Manager view (cannot see salary/SSN)
CREATE VIEW employees_manager_view AS
SELECT emp_id, name, email, manager_id
FROM employees
WHERE manager_id = current_setting('app.user_id')::UUID;

-- Public view (minimal info only)
CREATE VIEW employees_public_view AS
SELECT emp_id, name, email
FROM employees;

-- Grant access to views
GRANT SELECT ON employees_hr_view TO hr_role;
GRANT SELECT ON employees_manager_view TO manager_role;
GRANT SELECT ON employees_public_view TO public_role;

-- Audit column access
CREATE TABLE column_access_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  table_name VARCHAR,
  accessed_columns VARCHAR[],
  accessed_at TIMESTAMP DEFAULT NOW()
);

-- Implement triggers to log sensitive access
CREATE TRIGGER audit_salary_access
AFTER SELECT ON employees
FOR EACH ROW
INSERT INTO column_access_audit (user_name, table_name, accessed_columns)
VALUES (current_user, 'employees', ARRAY['salary', 'ssn']);
```

Q169: How do I implement fine-grained encryption for sensitive columns?

1. Identify sensitive columns (PII, payment data).
2. Encrypt at application layer before storing.
3. Use KMS for key management.
4. Implement decryption only when needed.
5. Audit decryption access.
6. Monitor encryption/decryption performance.

Script:
```sql
-- Encrypted column schema
CREATE TABLE customers (
  customer_id UUID PRIMARY KEY,
  name VARCHAR,
  encrypted_ssn BYTEA,  -- Encrypted with KMS
  encrypted_payment_method BYTEA,  -- Encrypted with KMS
  email VARCHAR,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Application-level encryption (Python example)
from google.cloud import kms
import base64

class ColumnEncryption:
    def __init__(self, project_id, key_ring, key):
        self.client = kms.KeyManagementServiceClient()
        self.key_path = self.client.crypto_key_path(
            project_id, 'global', key_ring, key
        )
    
    def encrypt_value(self, plaintext):
        response = self.client.encrypt(
            request={'name': self.key_path, 'plaintext': plaintext.encode()}
        )
        return base64.b64encode(response.ciphertext).decode()
    
    def decrypt_value(self, ciphertext):
        response = self.client.decrypt(
            request={'name': self.key_path, 'ciphertext': base64.b64decode(ciphertext)}
        )
        return response.plaintext.decode()

# Usage
encryption = ColumnEncryption('my-project', 'my-keyring', 'my-key')
encrypted_ssn = encryption.encrypt_value('123-45-6789')

# Insert into database
db.execute(
    "INSERT INTO customers (customer_id, encrypted_ssn) VALUES (%s, %s)",
    customer_id, encrypted_ssn
)

# Audit decryption access
CREATE TABLE encryption_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  column_name VARCHAR,
  table_name VARCHAR,
  decrypted_at TIMESTAMP DEFAULT NOW()
);
```

Q170: How do I implement database activity streaming to external systems?

1. Use changefeeds for data change streaming.
2. Stream to Kafka, Pub/Sub, or webhooks.
3. Implement filtering for sensitive events.
4. Handle streaming failures with retries.
5. Monitor streaming lag.
6. Implement consumer groups for scaling.

Script:
```sql
-- Stream changes to Kafka
CREATE CHANGEFEED FOR TABLE orders, customers
INTO 'kafka://kafka-broker:9092'
WITH 
  resolved='1s',
  format='avro',
  confluent_schema_registry='http://schema-registry:8081',
  topic_prefix='crdb_';

-- Stream to webhook
CREATE CHANGEFEED FOR TABLE high_value_orders
INTO 'webhook-https://my-service/events?ca_cert=$CA_CERT'
WITH format='json';

-- Stream to Google Pub/Sub
CREATE CHANGEFEED FOR TABLE transactions
INTO 'pubsub://my-project/my-topic'
WITH format='json';

-- Monitor changefeed status
SELECT job_id, status, high_water_timestamp
FROM crdb_internal.jobs
WHERE job_type = 'CHANGEFEED'
ORDER BY created DESC;

-- Pause/resume changefeed
PAUSE JOB changefeed_job_id;
RESUME JOB changefeed_job_id;

-- Filter sensitive events
CREATE CHANGEFEED FOR TABLE orders
WHERE amount > 10000  -- Only stream high-value orders
INTO 'kafka://broker/high-value-orders';
```

================================================================================
SECTION 30: PRODUCTION INCIDENT CASE STUDIES
================================================================================

Q171: Case Study: Recovering from cascading failure due to quorum loss

Scenario: All nodes in primary region went offline simultaneously during maintenance window.

Root Cause Analysis:
1. Maintenance activity caused all nodes in region to restart.
2. Quorum required 2 out of 3 nodes, both in primary region.
3. Standby region had single node (not enough for quorum).
4. System became read-only immediately.

Recovery Steps:
1. Identified primary region completely offline.
2. Promoted standby region node to primary (no quorum available).
3. Restored quorum by bringing primary region nodes back online.
4. Allowed automatic re-synchronization between regions.
5. Verified data consistency and application operations.
6. Implemented improvements to prevent recurrence.

SQL Recovery Procedures:
```sql
-- Check cluster health during failure
SELECT node_id, address, is_live 
FROM crdb_internal.nodes;
-- Result: All primary region nodes show is_live = false

-- Verify standby region has nodes
SELECT node_id, address, is_live 
FROM crdb_internal.nodes 
WHERE address LIKE '%standby%';
-- Result: Single node in standby region

-- Promote standby to handle current cluster state
ALTER CLUSTER SET REPLICATION FACTOR = 1;

-- Bring primary region nodes online
-- (infrastructure command to restart nodes)

-- Verify quorum restored
SELECT node_id, COUNT(*) as replica_count 
FROM crdb_internal.ranges_no_leases 
GROUP BY node_id;

-- Check for any divergence after re-sync
SELECT table_name, COUNT(*) as row_count 
FROM information_schema.tables 
GROUP BY table_name 
ORDER BY table_name;
```

Prevention Implementation:
```sql
-- Distribute replicas across regions
ALTER TABLE critical_table CONFIGURE ZONE USING
  num_replicas = 5,
  constraints = '[+region=us-west, +region=us-east, +region=eu]';

-- Monitor region-specific node status
CREATE ALERT region_node_down
  WHEN COUNT(nodes_down{region=~'primary|secondary'}) > 0
  NOTIFY pagerduty;

-- Implement maintenance window notifications
CREATE SCHEDULE maintenance_notification FOR
  (SELECT notify_on_maintenance())
  RECURRING EVERY 7 days;
```

Q172: Case Study: Data corruption from application bug causing cascading updates

Scenario: Application bug caused NULL values in critical column, corrupting 2M+ rows.

Root Cause: SQL query with incorrect NULL coalescing logic:
```sql
-- Buggy application code
UPDATE orders 
SET status = COALESCE(NULL, current_status)  -- Bug: NULL coalesced to NULL
WHERE order_id IN (SELECT ...);
```

Discovery: Queries started returning unexpected NULL values for status column.

Recovery Approach:
1. Identified bug timestamp through logs: 2024-01-15 14:32:00
2. Used point-in-time recovery to restore data: AS OF SYSTEM TIME '2024-01-15 14:30:00'
3. Created recovery table with uncorrupted data
4. Fixed application bug
5. Applied fix to corrupted data
6. Validated integrity

SQL Recovery:
```sql
-- Step 1: Identify corruption extent
SELECT COUNT(*) as corrupted_rows
FROM orders
WHERE status IS NULL 
  AND created_at < '2024-01-15 14:32:00';  -- Should have status values

-- Step 2: Recovery table with point-in-time data
CREATE TABLE orders_recovered AS
SELECT * FROM orders AS OF SYSTEM TIME '2024-01-15 14:30:00'
WHERE status IS NOT NULL;

-- Step 3: Backup corrupted table
ALTER TABLE orders RENAME TO orders_corrupted;

-- Step 4: Restore recovered data
ALTER TABLE orders_recovered RENAME TO orders;

-- Step 5: Fix application logic
-- UPDATE to use COALESCE(new_status, current_status) correctly

-- Step 6: Apply fixes
UPDATE orders 
SET status = COALESCE(new_status, status)
WHERE order_id IN (SELECT id FROM orders_corrupted WHERE status IS NULL);

-- Step 7: Validation
SELECT COUNT(*) as row_count, COUNT(DISTINCT status) as status_count
FROM orders;
```

Incident Timeline and Learnings:
- 14:32 - Corruption occurs
- 14:45 - Monitoring alerts trigger (12 min delay)
- 14:55 - Investigation identifies corrupted column
- 15:10 - Decision to restore from PITR
- 15:45 - Recovery complete and validated
- Total impact: 45 minutes data unavailable

Prevention Measures:
```sql
-- Add check constraint
ALTER TABLE orders
ADD CONSTRAINT chk_status_not_null
CHECK (status IS NOT NULL);

-- Implement validation triggers
CREATE TRIGGER validate_status_update
BEFORE UPDATE ON orders
FOR EACH ROW
BEGIN
  IF NEW.status IS NULL THEN
    RAISE EXCEPTION 'Status cannot be NULL';
  END IF;
END;

-- Automated integrity checks
CREATE SCHEDULE check_orders_integrity FOR
  (SELECT COUNT(*) as null_status_rows FROM orders WHERE status IS NULL)
  RECURRING EVERY 1 hour;
```

Q173: Case Study: Query performance regression causing service degradation

Scenario: Query response time increased from 50ms to 500ms+ causing user-facing delays.

Root Cause Investigation:
1. Performance degradation coincided with data load increase
2. Query plan analysis showed sequential scan instead of index scan
3. Table statistics outdated due to large insert batch
4. Query optimizer chose expensive sequential scan

SQL Troubleshooting:
```sql
-- Identify slow queries
SELECT query, execution_count, latency_p99
FROM crdb_internal.node_statement_statistics
WHERE latency_p99 > 500000  -- 500ms
ORDER BY latency_p99 DESC
LIMIT 10;

-- Analyze query plan
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = $1
  AND created_at > now() - interval '30 days'
ORDER BY created_at DESC;

-- Check table statistics
SHOW STATISTICS FOR TABLE orders;

-- Identify missing statistics
SELECT * FROM crdb_internal.index_usage_statistics
WHERE seq_scans > index_scans
  AND seq_scans > 1000;
```

Resolution Steps:
```sql
-- Step 1: Rebuild statistics
ANALYZE TABLE orders;

-- Step 2: Recreate index if needed
DROP INDEX IF EXISTS idx_orders_customer_date;
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);

-- Step 3: Verify improved plan
EXPLAIN (VERBOSE, ANALYZE)
SELECT * FROM orders
WHERE customer_id = $1
  AND created_at > now() - interval '30 days'
ORDER BY created_at DESC;

-- Step 4: Monitor improvement
SELECT query, latency_p99
FROM crdb_internal.node_statement_statistics
WHERE query LIKE '%orders%'
ORDER BY latency_p99 DESC;
```

Prevention:
```sql
-- Automatic statistics update schedule
CREATE SCHEDULE analyze_tables FOR ANALYZE
RECURRING EVERY 1 hour;

-- Monitor statistics staleness
CREATE ALERT stale_statistics
  WHEN stats_age_days > 7
  NOTIFY pagerduty;
```

================================================================================
COCKROACHDB ADMINISTRATION GUIDE - FINAL SECTION
Questions Q174-Q250 covering Specialized Topics, Multi-Cloud, Benchmarking, and Enterprise Scenarios

================================================================================
SECTION 31: ADVANCED MONITORING AND OBSERVABILITY
================================================================================

Q174: How do I implement custom dashboards for cluster health visualization?

1. Query key metrics from crdb_internal tables.
2. Build dashboards in Grafana with real-time updates.
3. Create different dashboards for operators, developers, executives.
4. Implement drill-down capabilities to root cause issues.
5. Set dashboard refresh rates based on importance.
6. Document dashboard KPIs and thresholds.

Script:
```sql
-- Create views for dashboard data
CREATE VIEW cluster_health_summary AS
SELECT 
  (SELECT COUNT(*) FROM crdb_internal.nodes WHERE is_live) as live_nodes,
  (SELECT COUNT(*) FROM crdb_internal.nodes) as total_nodes,
  (SELECT COUNT(*) FROM crdb_internal.ranges WHERE unavailable_replicas > 0) as unhealthy_ranges,
  (SELECT COUNT(*) FROM crdb_internal.ranges) as total_ranges,
  (SELECT AVG(latency_p99) FROM crdb_internal.node_statement_statistics) as avg_p99_latency,
  (SELECT SUM(bytes) FROM crdb_internal.stores) as total_storage_bytes;

-- Create performance dashboard view
CREATE VIEW performance_dashboard AS
SELECT 
  DATE_TRUNC('minute', timestamp) as minute,
  COUNT(*) as query_count,
  AVG(execution_time_ms) as avg_latency,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms) as p99_latency,
  MAX(execution_time_ms) as max_latency,
  SUM(rows_affected) as total_rows_affected
FROM query_log
WHERE timestamp > now() - interval '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute DESC;

-- Create replication dashboard view
CREATE VIEW replication_dashboard AS
SELECT 
  node_id,
  COUNT(*) as replica_count,
  SUM(CASE WHEN unavailable_replicas > 0 THEN 1 ELSE 0 END) as unhealthy_replicas,
  ROUND(100.0 * SUM(CASE WHEN unavailable_replicas = 0 THEN 1 ELSE 0 END) / COUNT(*), 2) as health_percent
FROM crdb_internal.ranges_no_leases
GROUP BY node_id
ORDER BY node_id;
```

Grafana Configuration (JSON):
```json
{
  "dashboard": {
    "title": "CockroachDB Cluster Health",
    "panels": [
      {
        "title": "Live Nodes",
        "targets": [{
          "expr": "SELECT live_nodes FROM cluster_health_summary"
        }],
        "thresholds": "0:red, 2:yellow, 3:green"
      },
      {
        "title": "Query Latency P99",
        "targets": [{
          "expr": "SELECT avg_p99_latency FROM cluster_health_summary"
        }],
        "thresholds": "100:green, 500:yellow, 1000:red"
      },
      {
        "title": "Storage Usage",
        "targets": [{
          "expr": "SELECT total_storage_bytes / (1024*1024*1024) as storage_gb FROM cluster_health_summary"
        }]
      }
    ]
  }
}
```

Q175: How do I implement anomaly detection for proactive issue identification?

1. Establish baseline metrics during normal operation.
2. Use statistical methods (Z-score, IQR) to detect outliers.
3. Implement machine learning models for pattern recognition.
4. Alert before issues impact users.
5. Track false positive/negative rates.
6. Continuously improve detection accuracy.

Script:
```python
import numpy as np
from scipy import stats
from sklearn.ensemble import IsolationForest

class AnomalyDetector:
    def __init__(self, sensitivity=2.5):
        self.sensitivity = sensitivity
        self.baseline_mean = None
        self.baseline_std = None
        self.model = IsolationForest(contamination=0.05)
    
    def train_baseline(self, historical_data):
        """Train on known-good data"""
        self.baseline_mean = np.mean(historical_data)
        self.baseline_std = np.std(historical_data)
        self.model.fit(historical_data.reshape(-1, 1))
    
    def detect_anomaly_zscore(self, value):
        """Z-score based detection"""
        if self.baseline_std == 0:
            return False
        z_score = abs((value - self.baseline_mean) / self.baseline_std)
        return z_score > self.sensitivity
    
    def detect_anomaly_ml(self, value):
        """ML-based detection"""
        prediction = self.model.predict([[value]])[0]
        return prediction == -1  # -1 indicates anomaly
    
    def detect_anomaly_combined(self, value):
        """Combined approach for accuracy"""
        zscore_anomaly = self.detect_anomaly_zscore(value)
        ml_anomaly = self.detect_anomaly_ml(value)
        # Alert only if both methods agree (reduce false positives)
        return zscore_anomaly and ml_anomaly

# Usage
detector = AnomalyDetector()
detector.train_baseline(np.array(historical_latencies))

# Monitor current latency
current_latency = 1200  # ms
if detector.detect_anomaly_combined(current_latency):
    alert("High latency detected", f"P99 latency: {current_latency}ms")
```

Q176: How do I implement SLA-driven alerting with escalation?

1. Define SLOs for each service component.
2. Calculate error budget remaining.
3. Alert when error budget at risk.
4. Implement tiered escalation procedures.
5. Route alerts to appropriate on-call engineer.
6. Implement runbooks for each alert.

Script:
```sql
-- SLA tracking
CREATE TABLE sla_metrics (
  service_name VARCHAR,
  month DATE,
  target_availability DECIMAL,  -- 99.9%
  actual_availability DECIMAL,
  error_budget_remaining DECIMAL,
  alert_status VARCHAR,  -- OK, WARNING, CRITICAL
  created_at TIMESTAMP DEFAULT NOW()
);

-- Calculate SLA compliance
WITH monthly_metrics AS (
  SELECT 
    'database' as service_name,
    date_trunc('month', now())::date as month,
    99.9 as target_availability,
    (1 - (COUNT(*) FILTER (WHERE is_error) / COUNT(*))::NUMERIC * 100) as actual_availability
  FROM request_logs
  WHERE timestamp > date_trunc('month', now())
  GROUP BY 1, 2
)
INSERT INTO sla_metrics (service_name, month, target_availability, actual_availability, error_budget_remaining)
SELECT 
  service_name,
  month,
  target_availability,
  actual_availability,
  (target_availability - actual_availability) as error_budget_remaining,
  CASE 
    WHEN (target_availability - actual_availability) < 0.1 THEN 'CRITICAL'
    WHEN (target_availability - actual_availability) < 0.3 THEN 'WARNING'
    ELSE 'OK'
  END as alert_status
FROM monthly_metrics;

-- Alert if error budget exhausted
CREATE ALERT sla_at_risk AS
SELECT service_name, error_budget_remaining
FROM sla_metrics
WHERE actual_availability < target_availability * 0.95
  AND month = date_trunc('month', now())::date
WITH (
  severity = 'CRITICAL',
  escalation = [
    {delay: '0min', notify: 'on_call_primary'},
    {delay: '15min', notify: 'on_call_backup'},
    {delay: '30min', notify: 'manager'}
  ],
  runbook = 'SLA_RECOVERY.md'
);
```

================================================================================
SECTION 32: MULTI-CLOUD AND HYBRID DEPLOYMENT
================================================================================

Q177: How do I implement multi-cloud CockroachDB deployment strategy?

1. Design for cloud provider independence.
2. Use Terraform/CloudFormation for IaC.
3. Implement cross-cloud replication.
4. Manage credentials securely across clouds.
5. Monitor performance and costs per cloud.
6. Plan failover between cloud providers.

Script:
```hcl
# Terraform for multi-cloud deployment

# AWS Region
module "aws_cockroachdb" {
  source = "./modules/cockroachdb"
  
  provider = aws
  region = "us-east-1"
  
  cluster_name = "cockroach-aws"
  node_count = 3
  instance_type = "c5.2xlarge"
  
  tags = {
    Environment = "production"
    Cloud = "AWS"
  }
}

# GCP Region
module "gcp_cockroachdb" {
  source = "./modules/cockroachdb"
  
  provider = google
  region = "us-central1"
  
  cluster_name = "cockroach-gcp"
  node_count = 3
  machine_type = "n1-standard-8"
  
  tags = {
    Environment = "production"
    Cloud = "GCP"
  }
}

# Azure Region
module "azure_cockroachdb" {
  source = "./modules/cockroachdb"
  
  provider = azurerm
  region = "eastus"
  
  cluster_name = "cockroach-azure"
  node_count = 3
  vm_size = "Standard_D4s_v3"
  
  tags = {
    Environment = "production"
    Cloud = "Azure"
  }
}

# Multi-cloud networking
resource "aws_vpc_peering_connection" "aws_gcp" {
  vpc_id = module.aws_cockroachdb.vpc_id
  peer_vpc_id = module.gcp_cockroachdb.vpc_id
}

resource "aws_vpc_peering_connection" "aws_azure" {
  vpc_id = module.aws_cockroachdb.vpc_id
  peer_vpc_id = module.azure_cockroachdb.vpc_id
}
```

Q178: How do I handle data sovereignty and compliance across regions?

1. Map data to specific regions based on regulations.
2. Implement geo-fencing at database level.
3. Verify data never leaves specified regions.
4. Audit cross-region data movement.
5. Test compliance regularly.
6. Document data residency architecture.

Script:
```sql
-- Data sovereignty configuration
-- EU data must stay in EU regions
ALTER TABLE eu_customers CONFIGURE ZONE USING 
  constraints = '[+region=eu-west-1, +region=eu-central-1]',
  lease_preferences = '[[+region=eu-central-1]]',
  num_replicas = 3;

-- US data must stay in US regions
ALTER TABLE us_customers CONFIGURE ZONE USING 
  constraints = '[+region=us-east-1, +region=us-west-1]',
  lease_preferences = '[[+region=us-east-1]]',
  num_replicas = 3;

-- Compliance audit query
SELECT 
  table_name,
  zone_config,
  COUNT(*) as replica_count,
  COUNT(*) FILTER (WHERE region NOT IN ('eu-west-1', 'eu-central-1')) as out_of_region_replicas
FROM crdb_internal.ranges
WHERE table_name LIKE 'eu_%'
GROUP BY table_name, zone_config
HAVING COUNT(*) FILTER (WHERE region NOT IN ('eu-west-1', 'eu-central-1')) > 0;

-- Alert if data leaves region
CREATE ALERT data_residency_violation AS
SELECT table_name, out_of_region_replicas
FROM compliance_audit
WHERE out_of_region_replicas > 0
WITH (severity = 'CRITICAL');
```

Q179: How do I optimize costs in multi-cloud deployment?

1. Monitor costs per cloud provider continuously.
2. Implement reserved instances/commitments.
3. Right-size instances based on utilization.
4. Implement auto-scaling for variable workloads.
5. Archive old data to cheaper storage.
6. Use spot instances for non-critical workloads.

Script:
```sql
-- Track costs by cloud provider
CREATE TABLE cloud_cost_tracking (
  provider VARCHAR,
  region VARCHAR,
  date DATE,
  compute_cost DECIMAL,
  storage_cost DECIMAL,
  network_cost DECIMAL,
  total_cost DECIMAL,
  nodes_active INT,
  avg_cpu_percent DECIMAL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Cost analysis query
SELECT 
  provider,
  SUM(total_cost) as monthly_cost,
  SUM(compute_cost) as compute_total,
  SUM(storage_cost) as storage_total,
  AVG(nodes_active) as avg_nodes,
  ROUND(SUM(total_cost) / NULLIF(SUM(nodes_active), 0), 2) as cost_per_node
FROM cloud_cost_tracking
WHERE date >= date_trunc('month', now())::date
GROUP BY provider
ORDER BY monthly_cost DESC;

-- Identify cost optimization opportunities
SELECT 
  provider,
  CASE 
    WHEN avg_cpu_percent < 20 THEN 'DOWNSIZE_INSTANCE'
    WHEN avg_cpu_percent > 80 THEN 'UPSIZE_INSTANCE'
    ELSE 'OPTIMAL'
  END as recommendation,
  ROUND(total_cost * 0.3, 2) as potential_monthly_savings
FROM cloud_cost_tracking
WHERE date = CURRENT_DATE - 1
ORDER BY potential_monthly_savings DESC;
```

================================================================================
SECTION 33: ADVANCED BENCHMARKING AND PERFORMANCE TESTING
================================================================================

Q180: How do I design and execute comprehensive benchmark tests?

1. Define realistic workload patterns.
2. Create schema matching production.
3. Generate representative data volumes.
4. Run baseline measurements.
5. Compare against performance targets.
6. Document results and findings.

Script:
```bash
#!/bin/bash
# Comprehensive benchmark suite

# 1. Setup test cluster
cockroach start-single-node --insecure --cache=4GB --max-sql-memory=2GB

# 2. Create benchmark schema
cockroach sql --insecure <<EOF
CREATE TABLE benchmark_orders (
  order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (customer_id) REFERENCES benchmark_customers(id)
);

CREATE TABLE benchmark_customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR,
  email VARCHAR UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_orders_customer ON benchmark_orders(customer_id);
CREATE INDEX idx_orders_created ON benchmark_orders(created_at);
EOF

# 3. Generate test data
cockroach workload fixtures load tpcc --warehouses 10 --drop

# 4. Run benchmark workload
echo "=== TPC-C Benchmark ==="
cockroach workload run tpcc \
  --warehouses 10 \
  --duration 10m \
  --concurrency 32 \
  --max-rate 0 \
  --ramp 5m

# 5. Measure single-query performance
echo "=== Single Query Benchmark ==="
time cockroach sql --insecure <<EOF
SELECT COUNT(*) FROM orders;
SELECT * FROM orders WHERE customer_id = 'some-id' ORDER BY created_at DESC LIMIT 100;
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
EOF

# 6. Collect metrics
echo "=== Performance Metrics ==="
cockroach sql --insecure <<EOF
SELECT 
  query,
  execution_count,
  latency_p50,
  latency_p99,
  cumulative_cpu_time_seconds
FROM crdb_internal.node_statement_statistics
ORDER BY execution_count DESC
LIMIT 20;
EOF
```

Q181: How do I conduct load testing for capacity planning?

1. Simulate realistic user load patterns.
2. Gradually increase load to find breaking point.
3. Monitor system metrics under load.
4. Identify bottlenecks and constraints.
5. Extrapolate capacity requirements.
6. Plan infrastructure accordingly.

Script:
```python
import concurrent.futures
import time
from cockroachdb import connect

class LoadTest:
    def __init__(self, num_threads=32, duration_seconds=300):
        self.num_threads = num_threads
        self.duration = duration_seconds
        self.results = []
    
    def run_query(self, query_func, thread_id):
        """Run queries from a thread"""
        start_time = time.time()
        query_count = 0
        error_count = 0
        
        while time.time() - start_time < self.duration:
            try:
                latency = query_func()
                self.results.append({
                    'thread_id': thread_id,
                    'latency_ms': latency,
                    'timestamp': time.time()
                })
                query_count += 1
            except Exception as e:
                error_count += 1
        
        return {
            'thread_id': thread_id,
            'query_count': query_count,
            'error_count': error_count,
            'duration': time.time() - start_time
        }
    
    def execute_load_test(self, query_func):
        """Run load test with thread pool"""
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_threads) as executor:
            futures = [
                executor.submit(self.run_query, query_func, i)
                for i in range(self.num_threads)
            ]
            results = [f.result() for f in concurrent.futures.as_completed(futures)]
        
        return self.analyze_results(results)
    
    def analyze_results(self, results):
        """Analyze load test results"""
        total_queries = sum(r['query_count'] for r in results)
        total_errors = sum(r['error_count'] for r in results)
        throughput = total_queries / self.duration
        error_rate = total_errors / (total_queries + total_errors)
        
        latencies = [r['latency_ms'] for r in self.results]
        
        return {
            'total_queries': total_queries,
            'throughput_qps': throughput,
            'error_rate': error_rate,
            'p50_latency': np.percentile(latencies, 50),
            'p95_latency': np.percentile(latencies, 95),
            'p99_latency': np.percentile(latencies, 99),
            'max_latency': max(latencies)
        }

# Usage
def simple_query():
    conn = connect()
    start = time.time()
    conn.execute("SELECT * FROM orders LIMIT 1")
    return (time.time() - start) * 1000  # Return ms

load_test = LoadTest(num_threads=32, duration_seconds=300)
results = load_test.execute_load_test(simple_query)

print(f"Throughput: {results['throughput_qps']} QPS")
print(f"P99 Latency: {results['p99_latency']} ms")
print(f"Error Rate: {results['error_rate']*100}%")
```

Q182: How do I perform competitive benchmarking against PostgreSQL/MySQL?

1. Create identical schemas in all databases.
2. Generate same test data volume.
3. Execute same workload queries.
4. Measure performance across all systems.
5. Compare results objectively.
6. Document findings and trade-offs.

Script:
```sql
-- Benchmark comparison query
-- Execute same query on all three databases

-- CockroachDB
EXPLAIN ANALYZE
SELECT c.id, c.name, COUNT(o.id) as order_count, SUM(o.amount) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at > now() - interval '1 year'
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 10
ORDER BY total_spent DESC
LIMIT 100;

-- Benchmark results storage
CREATE TABLE benchmark_comparison (
  test_name VARCHAR,
  database_system VARCHAR,
  query_text VARCHAR,
  execution_time_ms DECIMAL,
  rows_examined BIGINT,
  rows_returned BIGINT,
  index_used VARCHAR,
  cpu_time_ms DECIMAL,
  memory_used_mb INT,
  benchmark_date DATE,
  notes VARCHAR
);

-- Analysis query
SELECT 
  database_system,
  AVG(execution_time_ms) as avg_time,
  MIN(execution_time_ms) as min_time,
  MAX(execution_time_ms) as max_time,
  ROUND(100.0 * (MAX(execution_time_ms) - MIN(execution_time_ms)) / MIN(execution_time_ms), 2) as variance_percent
FROM benchmark_comparison
WHERE benchmark_date = CURRENT_DATE
GROUP BY database_system
ORDER BY avg_time;
```

================================================================================
SECTION 34: ADVANCED APPLICATION INTEGRATION PATTERNS
================================================================================

Q183: How do I implement database-driven microservices coordination?

1. Use database as coordination point for microservices.
2. Implement distributed lock tables for coordination.
3. Use changefeeds to propagate state changes.
4. Implement event sourcing for audit trails.
5. Handle eventual consistency in application logic.
6. Test failure scenarios thoroughly.

Script:
```sql
-- Microservice coordination schema
CREATE TABLE service_heartbeats (
  service_id VARCHAR PRIMARY KEY,
  service_name VARCHAR,
  last_heartbeat TIMESTAMP,
  status VARCHAR,  -- HEALTHY, DEGRADED, OFFLINE
  version VARCHAR
);

CREATE TABLE distributed_locks (
  resource_id VARCHAR PRIMARY KEY,
  service_id VARCHAR,
  acquired_at TIMESTAMP,
  expires_at TIMESTAMP,
  FOREIGN KEY (service_id) REFERENCES service_heartbeats(service_id)
);

-- Service registration
INSERT INTO service_heartbeats (service_id, service_name, last_heartbeat, status)
VALUES ('svc-123', 'order-processor', NOW(), 'HEALTHY')
ON CONFLICT (service_id) DO UPDATE SET 
  last_heartbeat = NOW(),
  status = 'HEALTHY';

-- Acquire distributed lock
BEGIN;
INSERT INTO distributed_locks (resource_id, service_id, expires_at)
VALUES ('process-order-123', 'svc-123', NOW() + interval '30 seconds');

-- Process work
UPDATE orders SET status = 'processing' WHERE id = 'order-123';

-- Release lock on commit
DELETE FROM distributed_locks WHERE resource_id = 'process-order-123';
COMMIT;

-- Detect unhealthy services
SELECT service_id, service_name, NOW() - last_heartbeat as age
FROM service_heartbeats
WHERE last_heartbeat < NOW() - interval '30 seconds'
  AND status = 'HEALTHY';
```

Q184: How do I implement CDC-based real-time analytics?

1. Enable changefeeds on transactional tables.
2. Stream changes to analytics platform.
3. Implement real-time data transformation.
4. Update analytics views continuously.
5. Monitor data freshness and lag.
6. Implement alerting for data quality issues.

Script:
```sql
-- Enable CDC for analytics
CREATE CHANGEFEED FOR TABLE orders, customers, products
INTO 'kafka://kafka-broker:9092'
WITH 
  resolved='10s',
  format='avro',
  confluent_schema_registry='http://registry:8081',
  topic_prefix='analytics_';

-- Real-time materialized view
CREATE MATERIALIZED VIEW orders_analytics AS
SELECT 
  DATE_TRUNC('hour', o.created_at)::date as order_date,
  DATE_PART('hour', o.created_at) as hour,
  p.category,
  COUNT(DISTINCT o.id) as order_count,
  SUM(o.amount) as revenue,
  AVG(o.amount) as avg_order_value,
  COUNT(DISTINCT o.customer_id) as unique_customers
FROM orders o
JOIN products p ON o.product_id = p.id
WHERE o.created_at > NOW() - interval '7 days'
GROUP BY 1, 2, 3;

-- Refresh analytics view on schedule
CREATE SCHEDULE refresh_analytics FOR 
  REFRESH MATERIALIZED VIEW orders_analytics
  RECURRING EVERY 5 minutes;

-- Monitor CDC lag
SELECT 
  job_id,
  status,
  high_water_timestamp,
  NOW() - high_water_timestamp as lag_seconds
FROM crdb_internal.jobs
WHERE job_type = 'CHANGEFEED';
```

================================================================================
SECTION 35: PRODUCTION READINESS AND OPERATIONAL EXCELLENCE
================================================================================

Q185: What is the comprehensive production readiness checklist?

1. Infrastructure: 3+ nodes across regions, auto-scaling configured.
2. Backup/DR: Automated backups, tested recovery, documented RTO/RPO.
3. Monitoring: All critical metrics tracked, alerting configured.
4. Security: TLS enabled, roles/permissions configured, audit logging active.
5. Performance: Baseline established, slow queries optimized, capacity planned.
6. Documentation: Runbooks, playbooks, architecture documented.

Checklist Script:
```bash
#!/bin/bash
# Production Readiness Verification

echo "=== CockroachDB Production Readiness Checklist ==="

# 1. Infrastructure
echo "1. Infrastructure Check"
node_count=$(cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM crdb_internal.nodes")
echo "   Nodes: $node_count $([ "$node_count" -ge 3 ] && echo '✓' || echo '✗')"

regions=$(cockroach sql --certs-dir=certs -c "SELECT COUNT(DISTINCT region) FROM crdb_internal.nodes")
echo "   Regions: $regions $([ "$regions" -ge 2 ] && echo '✓' || echo '✗')"

# 2. Backup & DR
echo "2. Backup & DR Check"
backup_count=$(cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM system.scheduled_jobs WHERE schedule_name LIKE '%backup%'")
echo "   Backup Schedules: $backup_count $([ "$backup_count" -ge 1 ] && echo '✓' || echo '✗')"

# 3. Monitoring
echo "3. Monitoring Check"
curl -s http://localhost:8080/_status/vars > /dev/null && echo "   Metrics: ✓" || echo "   Metrics: ✗"

# 4. Security
echo "4. Security Check"
[ -f "/path/to/certs/node.crt" ] && echo "   TLS Certs: ✓" || echo "   TLS Certs: ✗"

users=$(cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM system.users WHERE username != 'root'")
echo "   Custom Users: $users $([ "$users" -gt 0 ] && echo '✓' || echo '✗')"

# 5. Performance
echo "5. Performance Check"
disk_usage=$(cockroach sql --certs-dir=certs -c "SELECT ROUND(100.0*SUM(capacity_used)/SUM(capacity_total), 2) FROM crdb_internal.stores")
echo "   Disk Usage: $disk_usage% $([ "${disk_usage%.*}" -lt 70 ] && echo '✓' || echo '✗')"

# 6. Documentation
echo "6. Documentation Check"
[ -f "ARCHITECTURE.md" ] && echo "   Architecture Doc: ✓" || echo "   Architecture Doc: ✗"
[ -f "RUNBOOK.md" ] && echo "   Runbook: ✓" || echo "   Runbook: ✗"
[ -f "DR_PROCEDURES.md" ] && echo "   DR Procedures: ✓" || echo "   DR Procedures: ✗"

echo "=== Checklist Complete ==="
```

Q186: How do I implement continuous improvement and operational maturity?

1. Collect operational metrics and track trends.
2. Conduct regular postmortems on incidents.
3. Implement improvements based on learnings.
4. Automate manual procedures progressively.
5. Train team members on best practices.
6. Measure and track operational KPIs.

Script:
```sql
-- Operational KPI tracking
CREATE TABLE operational_kpis (
  kpi_name VARCHAR,
  week_starting DATE,
  target_value DECIMAL,
  actual_value DECIMAL,
  status VARCHAR,  -- MET, MISSED
  notes VARCHAR,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Example KPIs
INSERT INTO operational_kpis VALUES
  ('deployment_success_rate', CURRENT_DATE - 1, 99.5, 100.0, 'MET', NULL, NOW()),
  ('incident_response_time_minutes', CURRENT_DATE - 1, 15, 8, 'MET', NULL, NOW()),
  ('backup_success_rate', CURRENT_DATE - 1, 99.9, 100.0, 'MET', NULL, NOW()),
  ('slo_compliance', CURRENT_DATE - 1, 99.9, 99.95, 'MET', NULL, NOW());

-- Track improvement trends
SELECT 
  kpi_name,
  week_starting,
  actual_value,
  LAG(actual_value) OVER (PARTITION BY kpi_name ORDER BY week_starting) as previous_week,
  (actual_value - LAG(actual_value) OVER (PARTITION BY kpi_name ORDER BY week_starting)) as trend
FROM operational_kpis
ORDER BY week_starting DESC, kpi_name;

-- Incident postmortem tracking
CREATE TABLE incident_postmortems (
  incident_id UUID PRIMARY KEY,
  incident_date DATE,
  description VARCHAR,
  severity VARCHAR,
  resolution_time_minutes INT,
  root_cause VARCHAR,
  improvements_implemented INT,
  improvements_pending INT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Count improvements per quarter
SELECT 
  DATE_TRUNC('quarter', incident_date)::date as quarter,
  COUNT(*) as incidents,
  SUM(improvements_implemented) as improvements_done,
  SUM(improvements_pending) as improvements_pending
FROM incident_postmortems
GROUP BY quarter
ORDER BY quarter DESC;
```

Q187: How do I establish SLA/SLO targets and track compliance?

1. Define business-driven SLO targets.
2. Map SLOs to infrastructure capacity.
3. Implement error budget tracking.
4. Alert when SLO at risk.
5. Report compliance to stakeholders.
6. Use SLO failures to drive improvements.

Script:
```sql
-- SLO tracking database
CREATE TABLE service_slos (
  service_name VARCHAR,
  slo_metric VARCHAR,
  slo_target DECIMAL,  -- e.g., 99.9 for availability
  slo_window VARCHAR,  -- MONTH, QUARTER, YEAR
  error_budget DECIMAL,  -- Remaining budget
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO service_slos VALUES
  ('database', 'availability', 99.95, 'MONTH', 0.05, NOW()),
  ('database', 'p99_latency', 100, 'MONTH', NULL, NOW()),
  ('api', 'availability', 99.9, 'MONTH', 0.10, NOW()),
  ('api', 'p99_latency', 200, 'MONTH', NULL, NOW());

-- Calculate monthly SLO compliance
WITH monthly_metrics AS (
  SELECT 
    'database' as service_name,
    'availability' as metric,
    99.97 as actual_value
  UNION ALL
  SELECT 'api', 'availability', 99.92
)
SELECT 
  mm.service_name,
  mm.metric,
  ss.slo_target,
  mm.actual_value,
  CASE 
    WHEN mm.actual_value >= ss.slo_target THEN 'MET'
    ELSE 'MISSED'
  END as status,
  (ss.slo_target - mm.actual_value) as shortfall
FROM monthly_metrics mm
JOIN service_slos ss ON mm.service_name = ss.service_name 
  AND mm.metric = ss.slo_metric;

-- Error budget consumption
SELECT 
  service_name,
  slo_target,
  (100 - slo_target) as total_error_budget,
  error_budget as remaining_budget,
  ROUND(100.0 * error_budget / (100 - slo_target), 2) as remaining_percent
FROM service_slos
WHERE slo_window = 'MONTH'
ORDER BY remaining_percent ASC;
```

================================================================================
SECTION 36: FINAL PRODUCTION BEST PRACTICES
================================================================================

Q188: What are the top 20 critical production best practices?

1. Implement 3+ node cluster across regions for resilience.
2. Automated backups with tested recovery procedures.
3. Comprehensive monitoring with SLO tracking.
4. TLS encryption for all connections.
5. Role-based access control (RBAC) implementation.
6. Audit logging for compliance.
7. Capacity planning with 1.5x headroom.
8. Connection pooling for scalability.
9. Query optimization and indexing.
10. Incident response playbooks documented.
11. Disaster recovery procedures tested monthly.
12. Change management with approvals.
13. Performance baseline established.
14. Cost monitoring and optimization.
15. Security hardening (patches, scanning).
16. Team training on cluster operations.
17. Runbooks for common procedures.
18. Health checks and automatic recovery.
19. Data validation and consistency checks.
20. Documentation kept current.

Q189: How do I transition from legacy systems to CockroachDB?

1. Parallel run: legacy and CockroachDB simultaneously.
2. Dual-write: writes go to both systems.
3. Validate data consistency continuously.
4. Gradual traffic migration to CockroachDB.
5. Rollback procedure if issues occur.
6. Monitor both systems for anomalies.

Migration Timeline:
- Week 1-2: Schema migration and data import
- Week 3-4: Parallel operation with dual-writes
- Week 5-6: Gradual traffic shift (10%, 25%, 50%, 75%, 100%)
- Week 7-8: Monitoring and stabilization
- Week 9: Decommission legacy system

Q190: What are critical skills needed for CockroachDB operations team?

1. Distributed systems knowledge.
2. SQL and database design expertise.
3. Linux/infrastructure management.
4. Monitoring and observability tools.
5. Scripting and automation (Bash, Python, Go).
6. Problem-solving and troubleshooting.
7. Cloud platform expertise (AWS, GCP, Azure).
8. Security and compliance knowledge.
9. Capacity planning and performance tuning.
10. Communication and documentation skills.

Training Program:
- Month 1: Architecture and fundamentals
- Month 2: Operations and troubleshooting
- Month 3: Advanced tuning and optimization
- Ongoing: Hands-on practice and certifications

================================================================================
COCKROACHDB ADMINISTRATION GUIDE - COMPLETE FINAL EDITION
Questions Q191-Q250 covering Edge Cases, Specialized Workloads, and Enterprise Integration

================================================================================
SECTION 37: EDGE CASES AND CORNER SCENARIOS
================================================================================

Q191: How do I handle clock skew and time synchronization issues?

1. Monitor NTP synchronization across all nodes.
2. Set max-offset for acceptable clock divergence: --max-offset=500ms
3. Verify all nodes synchronize via ntpstat command.
4. Monitor for clock jump warnings in logs.
5. Restart nodes if clock skew exceeds threshold.
6. Implement alerting for NTP synchronization failures.

Script:
```bash
# Monitor NTP status on all nodes
for node in node1 node2 node3; do
  echo "=== NTP Status on $node ==="
  ssh $node "ntpstat"
  ssh $node "timedatectl status"
done

# Detect clock skew in CockroachDB
cockroach sql --certs-dir=certs <<EOF
-- Check node clock offset
SELECT node_id, address, clock_offset_millis
FROM crdb_internal.node_runtime_info
WHERE ABS(clock_offset_millis) > 100;  -- Alert if >100ms

-- Monitor clock jump warnings
SELECT * FROM system.event_log
WHERE event_type = 'clock_jump'
ORDER BY timestamp DESC
LIMIT 10;
EOF

# Fix NTP synchronization
sudo systemctl restart ntp
sudo ntpq -p  # Verify peers
```

Q192: How do I handle very large result sets without memory issues?

1. Use pagination or cursor-based retrieval.
2. Implement streaming result processing.
3. Use batch processing with LIMIT clauses.
4. Avoid SELECT * for large tables.
5. Implement application-level result streaming.
6. Monitor memory usage per query.

Script:
```python
# Streaming large result set processing
import sys
from cockroachdb import connect

class StreamingResultProcessor:
    def __init__(self, batch_size=10000):
        self.batch_size = batch_size
        self.conn = connect()
    
    def process_large_table(self, table_name, process_func):
        """Process large table in batches without loading all in memory"""
        offset = 0
        total_processed = 0
        
        while True:
            # Fetch batch
            query = f"""
                SELECT id, data FROM {table_name}
                ORDER BY id
                LIMIT {self.batch_size}
                OFFSET {offset}
            """
            
            rows = self.conn.execute(query)
            if not rows:
                break
            
            # Process batch
            for row in rows:
                process_func(row)
                total_processed += 1
            
            offset += self.batch_size
            print(f"Processed {total_processed} rows", file=sys.stderr)
        
        return total_processed

# Usage
processor = StreamingResultProcessor(batch_size=50000)

def process_row(row):
    # Custom processing logic
    id_val, data = row
    # Do something with row
    pass

total = processor.process_large_table('large_table', process_row)
print(f"Total processed: {total}")
```

Q193: How do I handle timezone-related issues in global deployments?

1. Store all timestamps in UTC.
2. Convert to local timezone at application layer.
3. Use TIMESTAMP WITH TIME ZONE for time zone awareness.
4. Verify all database nodes use UTC.
5. Test timezone-dependent queries thoroughly.
6. Document timezone handling in application.

Script:
```sql
-- Store times in UTC
CREATE TABLE events (
  event_id UUID PRIMARY KEY,
  event_time TIMESTAMP NOT NULL,  -- Always UTC
  user_timezone VARCHAR,
  local_event_time TIMESTAMP GENERATED ALWAYS AS 
    (event_time AT TIME ZONE user_timezone) STORED
);

-- Queries using timezones
SELECT 
  event_id,
  event_time AT TIME ZONE 'America/New_York' as eastern_time,
  event_time AT TIME ZONE 'Europe/London' as london_time,
  event_time AT TIME ZONE 'Asia/Tokyo' as tokyo_time
FROM events
WHERE event_time > now() - interval '7 days';

-- Verify database timezone
SHOW TIMEZONE;
-- Should return 'UTC'

-- Explicit UTC conversion
INSERT INTO events (event_id, event_time, user_timezone)
VALUES (gen_random_uuid(), NOW() AT TIME ZONE 'UTC', 'America/New_York');
```

Q194: How do I handle schema evolution in high-traffic production?

1. Add columns as nullable with default values (non-blocking).
2. Backfill data in background without locking.
3. Add constraints only after data validation.
4. Use views to maintain backward compatibility.
5. Coordinate with application deployment.
6. Test schema changes on staging cluster first.

Script:
```sql
-- Non-blocking schema evolution
-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN new_field VARCHAR DEFAULT NULL;

-- Step 2: Backfill in batches (non-blocking)
DO $$
BEGIN
  FOR i IN 1..1000000 BY 10000 LOOP
    UPDATE users 
      SET new_field = CONCAT('migrated_', id)
      WHERE new_field IS NULL 
      LIMIT 10000;
    COMMIT;
    -- Pause between batches to reduce load
    PERFORM pg_sleep(0.5);
  END LOOP;
END
$$;

-- Step 3: Verify backfill complete
SELECT COUNT(*) as remaining 
FROM users 
WHERE new_field IS NULL;

-- Step 4: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN new_field SET NOT NULL;

-- Backward compatibility view
CREATE VIEW users_legacy AS
SELECT 
  id, name, email, 
  new_field as legacy_field
FROM users;

-- Application uses view during transition
SELECT * FROM users_legacy WHERE id = $1;
```

Q195: How do I handle hotspot keys and contention?

1. Identify hot keys through monitoring.
2. Distribute writes across multiple rows.
3. Use sequence distribution for sequential IDs.
4. Implement bucketing for load distribution.
5. Adjust transaction isolation if appropriate.
6. Monitor contention metrics.

Script:
```sql
-- Detect hotspot keys
SELECT 
  table_name,
  index_name,
  COUNT(*) as contention_events,
  COUNT(DISTINCT transaction_id) as affected_transactions
FROM contention_events
WHERE timestamp > now() - interval '1 hour'
GROUP BY table_name, index_name
ORDER BY contention_events DESC;

-- Hotspot mitigation: distribute writes
-- Instead of single counter row
CREATE TABLE counters_distributed (
  counter_id VARCHAR,
  bucket INT,  -- Distribute across buckets
  value INT,
  PRIMARY KEY (counter_id, bucket)
);

-- Increment uses random bucket
UPDATE counters_distributed
SET value = value + 1
WHERE counter_id = 'views' 
  AND bucket = FLOOR(RANDOM() * 100);

-- Read total by summing
SELECT SUM(value) as total_views
FROM counters_distributed
WHERE counter_id = 'views';

-- Monitor transaction contention
SELECT 
  COUNT(*) as contention_count,
  MAX(duration_ms) as max_duration
FROM crdb_internal.transaction_contention_events
WHERE timestamp > now() - interval '1 hour';
```

================================================================================
SECTION 38: SPECIALIZED WORKLOAD PATTERNS
================================================================================

Q196: How do I optimize for write-heavy OLTP workloads?

1. Batch inserts to reduce transaction overhead.
2. Minimize transaction scope.
3. Use partitioning to distribute writes.
4. Implement connection pooling.
5. Monitor write throughput and latency.
6. Tune cluster settings for write performance.

Script:
```sql
-- Write-optimized schema
CREATE TABLE events (
  event_id UUID,
  timestamp TIMESTAMP NOT NULL,
  event_type VARCHAR,
  data JSONB,
  PRIMARY KEY (timestamp, event_id)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_current VALUES FROM (CURRENT_DATE) TO (CURRENT_DATE + 1),
  PARTITION p_future VALUES FROM (CURRENT_DATE + 1) TO (MAXVALUE)
);

-- Batch insert for performance
BEGIN;
INSERT INTO events (event_id, timestamp, event_type, data) VALUES
  (gen_random_uuid(), NOW(), 'login', '{"user": "alice"}'),
  (gen_random_uuid(), NOW(), 'page_view', '{"page": "/home"}'),
  (gen_random_uuid(), NOW(), 'purchase', '{"amount": 99.99}');
COMMIT;

-- Cluster settings for write performance
SET CLUSTER SETTING kv.allocator.mode='aggressive';
SET CLUSTER SETTING sql.defaults.max_retries=3;
SET CLUSTER SETTING sql.defaults.statement_timeout='30s';

-- Monitor write performance
SELECT 
  DATE_TRUNC('minute', timestamp) as minute,
  COUNT(*) as events_written,
  AVG(execution_time_ms) as avg_write_time
FROM write_log
WHERE timestamp > now() - interval '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute DESC;
```

Q197: How do I optimize for read-heavy OLAP workloads?

1. Create columnar indexes for analytics.
2. Implement materialized views for aggregations.
3. Use denormalized schema for faster queries.
4. Enable follower reads for distributed queries.
5. Implement query result caching.
6. Archive old data to cold storage.

Script:
```sql
-- OLAP-optimized schema
CREATE TABLE fact_sales (
  sale_id UUID,
  date DATE NOT NULL,
  customer_id UUID,
  product_id UUID,
  amount DECIMAL,
  quantity INT,
  region VARCHAR,
  PRIMARY KEY (date, sale_id)
) PARTITION BY RANGE (date);

-- Materialized view for common aggregations
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
  date,
  region,
  COUNT(*) as transaction_count,
  SUM(amount) as total_revenue,
  AVG(amount) as avg_sale_value,
  SUM(quantity) as total_units,
  COUNT(DISTINCT customer_id) as unique_customers
FROM fact_sales
WHERE date >= CURRENT_DATE - 90
GROUP BY date, region;

-- Enable follower reads for OLAP queries
SET SESSION enable_follower_reads = true;

-- Index for analytics queries
CREATE INDEX idx_sales_region_date ON fact_sales(region, date);

-- Query analytics using materialized view
SELECT 
  date,
  region,
  total_revenue,
  ROUND(100.0 * total_revenue / 
    SUM(total_revenue) OVER (PARTITION BY date), 2) as revenue_percent
FROM daily_sales_summary
WHERE date >= CURRENT_DATE - 30
ORDER BY date DESC, revenue_percent DESC;
```

Q198: How do I implement zero-copy data sharing between clusters?

1. Use physical cluster replication for zero-copy.
2. Monitor replication lag continuously.
3. Promote secondary when needed.
4. Implement automatic failover.
5. Test failover procedures regularly.
6. Document RTO/RPO for each cluster.

Script:
```sql
-- Setup physical cluster replication
ALTER CLUSTER SET REPLICATION FACTOR = 2;

-- Monitor replication status
SELECT 
  cluster_name,
  status,
  replicated_time,
  NOW() - replicated_time as lag_seconds
FROM crdb_internal.cluster_replication_status;

-- Verify replication healthy
SELECT 
  CASE 
    WHEN status = 'REPLICATING' THEN 'HEALTHY'
    WHEN status = 'ERROR' THEN 'ERROR'
    ELSE status
  END as status,
  replicated_time
FROM crdb_internal.cluster_replication_status;

-- Promote secondary cluster
BEGIN;
-- Step 1: Stop writes to primary (application level)
-- Step 2: Wait for replication to catch up
WAIT FOR (SELECT NOW() - replicated_time < INTERVAL '1s' 
  FROM crdb_internal.cluster_replication_status);

-- Step 3: Promote secondary
ALTER CLUSTER SET REPLICATION FACTOR = 1;

-- Step 4: Verify promoted cluster operational
SELECT COUNT(*) as total_records FROM fact_table;

COMMIT;
```

Q199: How do I handle heterogeneous workloads on shared cluster?

1. Create separate node pools for different workload types.
2. Implement resource quotas per workload.
3. Use connection limits and statement timeouts.
4. Monitor resource usage per workload.
5. Implement cost allocation per workload.
6. Prioritize critical workloads.

Script:
```sql
-- Separate roles for different workloads
CREATE ROLE oltp_workload NOLOGIN;
CREATE ROLE olap_workload NOLOGIN;
CREATE ROLE batch_workload NOLOGIN;

-- Set resource limits per workload
ALTER ROLE oltp_workload SET statement_timeout = '5s';
ALTER ROLE olap_workload SET statement_timeout = '5m';
ALTER ROLE batch_workload SET statement_timeout = '1h';

ALTER ROLE oltp_workload CONNECTION LIMIT 100;
ALTER ROLE olap_workload CONNECTION LIMIT 20;
ALTER ROLE batch_workload CONNECTION LIMIT 5;

-- Monitor resource usage
SELECT 
  user_name,
  COUNT(*) as active_connections,
  AVG(execution_time_ms) as avg_latency,
  SUM(rows_affected) as total_rows_affected
FROM crdb_internal.node_sessions
GROUP BY user_name
ORDER BY active_connections DESC;

-- Cost allocation by workload
CREATE TABLE workload_costs (
  workload_name VARCHAR,
  date DATE,
  queries_executed BIGINT,
  bytes_processed BIGINT,
  execution_seconds DECIMAL,
  estimated_cost DECIMAL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

Q200: How do I optimize for hybrid transactional/analytical (HTAP) workloads?

1. Separate OLTP and OLAP workloads with indexes.
2. Use follower reads for analytics.
3. Implement materialized views for aggregations.
4. Use changefeeds to sync to analytics platform.
5. Balance query performance across workloads.
6. Monitor HTAP query patterns.

Script:
```sql
-- HTAP schema optimization
CREATE TABLE orders (
  order_id UUID PRIMARY KEY,
  customer_id UUID,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP DEFAULT NOW(),
  -- Denormalized fields for analytics
  customer_name VARCHAR,
  customer_region VARCHAR,
  product_category VARCHAR
);

-- OLTP index: fast lookups
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);

-- OLAP index: analytics queries
CREATE INDEX idx_orders_created_region ON orders(created_at, customer_region) INCLUDE (amount, status);

-- Materialized view for analytics
CREATE MATERIALIZED VIEW orders_analytics AS
SELECT 
  customer_region,
  DATE_TRUNC('day', created_at)::date as order_date,
  status,
  COUNT(*) as order_count,
  SUM(amount) as total_revenue,
  AVG(amount) as avg_value
FROM orders
WHERE created_at > NOW() - INTERVAL '90 days'
GROUP BY customer_region, DATE_TRUNC('day', created_at), status;

-- OLTP query: fast transactional
SELECT * FROM orders WHERE customer_id = $1 AND status = 'pending';

-- OLAP query: using follower read
SET SESSION enable_follower_reads = true;
SELECT * FROM orders_analytics 
WHERE customer_region = 'US' 
  AND order_date >= CURRENT_DATE - 30;
```

================================================================================
SECTION 39: ADVANCED TROUBLESHOOTING SCENARIOS
================================================================================

Q201: How do I diagnose and fix transaction conflicts at scale?

1. Monitor conflict rates continuously.
2. Identify conflicting tables and patterns.
3. Implement optimistic locking strategies.
4. Adjust retry logic in applications.
5. Denormalize if conflicts unresolvable.
6. Implement compensation logic.

Script:
```sql
-- Monitor transaction conflicts
CREATE TABLE conflict_monitoring (
  timestamp TIMESTAMP DEFAULT NOW(),
  table_name VARCHAR,
  conflict_type VARCHAR,
  conflict_count INT,
  retry_rate DECIMAL
);

-- Detect high-conflict tables
SELECT 
  table_name,
  COUNT(*) as conflict_count,
  COUNT(DISTINCT transaction_id) as affected_transactions,
  ROUND(100.0 * COUNT(*) / LAG(COUNT(*)) OVER (ORDER BY timestamp), 2) as growth_percent
FROM crdb_internal.transaction_contention_events
WHERE timestamp > now() - interval '1 hour'
GROUP BY table_name
HAVING COUNT(*) > 100
ORDER BY conflict_count DESC;

-- Implement optimistic locking
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name VARCHAR,
  price DECIMAL,
  version INT DEFAULT 1
);

-- Update with conflict detection
UPDATE products
SET price = $new_price, version = version + 1
WHERE id = $product_id AND version = $expected_version;

-- Check if update succeeded (version matches)
SELECT version FROM products WHERE id = $product_id;

-- Application retry logic (pseudocode)
def update_with_retry(product_id, new_price, max_retries=3):
    for attempt in range(max_retries):
        product = db.query("SELECT * FROM products WHERE id = ?", product_id)
        result = db.execute(
            "UPDATE products SET price = ?, version = version + 1 WHERE id = ? AND version = ?",
            new_price, product_id, product['version']
        )
        if result.rows_affected > 0:
            return True
        # Conflict, retry
    return False
```

Q202: How do I handle storage subsystem failures gracefully?

1. Monitor disk health metrics continuously.
2. Implement automatic failover for failed stores.
3. Monitor disk I/O and queue depth.
4. Implement health checks for storage.
5. Pre-stage replacement hardware.
6. Document recovery procedures.

Script:
```bash
# Monitor disk health
for node in node1 node2 node3; do
  echo "=== Disk Health on $node ==="
  ssh $node "smartctl -H /dev/sda"
  ssh $node "iostat -x 1 2 | tail -n 3"
  ssh $node "df -h | grep cockroach"
done

# Detect disk issues in logs
cockroach sql --certs-dir=certs <<EOF
SELECT * FROM system.event_log
WHERE event_type LIKE '%disk%' OR event_type LIKE '%storage%'
ORDER BY timestamp DESC
LIMIT 20;
EOF

# Manually trigger node removal if storage failed
cockroach node decommission 3 --certs-dir=certs
# Wait for ranges to move off node 3
cockroach sql --certs-dir=certs <<EOF
SELECT COUNT(*) as remaining_ranges
FROM crdb_internal.ranges_no_leases
WHERE lease_holder_node_id = 3;
EOF
```

Q203: How do I handle cascading failures and recovery?

1. Implement circuit breakers to prevent cascade.
2. Degrade gracefully when dependencies unavailable.
3. Implement bulkheads for resource isolation.
4. Monitor health of dependencies.
5. Implement automatic recovery procedures.
6. Test failure scenarios regularly.

Script:
```python
# Cascade prevention with circuit breaker
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func):
        if self.state == 'OPEN':
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = 'HALF_OPEN'
            else:
                raise Exception('Circuit breaker is OPEN')
        
        try:
            result = func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
            print(f"Circuit breaker opened after {self.failure_count} failures")

# Usage
db_breaker = CircuitBreaker(failure_threshold=5, timeout=60)

def query_database():
    def execute():
        # Actual query execution
        return db.query("SELECT * FROM orders")
    
    return db_breaker.call(execute)

# Application handles circuit breaker state
try:
    orders = query_database()
except Exception as e:
    if "circuit breaker" in str(e):
        # Use fallback or cached data
        orders = cache.get_cached_orders()
```

================================================================================
SECTION 40: ENTERPRISE INTEGRATION AND COMPLIANCE
================================================================================

Q204: How do I implement federated identity management with CockroachDB?

1. Integrate with enterprise LDAP/AD.
2. Implement OAuth2/OIDC for SSO.
3. Map enterprise groups to database roles.
4. Implement automatic user provisioning.
5. Audit identity changes and access.
6. Test identity provider failover.

Script:
```sql
-- LDAP/OIDC integration (application-layer)
-- Map enterprise identities to database users

CREATE TABLE identity_mappings (
  external_id VARCHAR PRIMARY KEY,
  enterprise_user VARCHAR UNIQUE,
  db_username VARCHAR UNIQUE,
  enterprise_groups VARCHAR[],
  db_roles VARCHAR[],
  created_at TIMESTAMP DEFAULT NOW(),
  last_sync TIMESTAMP
);

-- Example mapping
INSERT INTO identity_mappings VALUES
  ('alice@company.com', 'alice', 'db_user_alice', 
   ARRAY['engineering', 'backend'], ARRAY['read_write', 'admin'], NOW(), NOW());

-- Sync enterprise groups to database roles
CREATE PROCEDURE sync_identity_mappings() AS $$
BEGIN
  -- 1. Query enterprise identity provider
  -- 2. Compare with identity_mappings table
  -- 3. Create/update/remove users as needed
  -- 4. Update role assignments
  
  UPDATE identity_mappings 
  SET last_sync = NOW()
  WHERE external_id IS NOT NULL;
END
$$ LANGUAGE plpgsql;

-- Schedule regular sync
CREATE SCHEDULE sync_identities FOR CALL sync_identity_mappings()
RECURRING EVERY 1 hour;

-- Audit identity access
CREATE TABLE identity_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_identity VARCHAR,
  action VARCHAR,
  resource VARCHAR,
  timestamp TIMESTAMP DEFAULT NOW()
);
```

Q205: How do I implement data residency compliance for multiple jurisdictions?

1. Map data to specific countries/regions.
2. Enforce at cluster level via zone configuration.
3. Verify no cross-jurisdiction replication.
4. Test compliance regularly.
5. Audit data movement.
6. Document compliance architecture.

Script:
```sql
-- Multi-jurisdiction compliance configuration
-- US data stays in US regions only
ALTER TABLE us_customers CONFIGURE ZONE USING 
  constraints = '[+jurisdiction=US]',
  lease_preferences = '[[+jurisdiction=US]]';

-- EU data stays in EU regions (GDPR compliance)
ALTER TABLE eu_customers CONFIGURE ZONE USING 
  constraints = '[+jurisdiction=EU]',
  lease_preferences = '[[+jurisdiction=EU]]';

-- Asia-Pacific data stays in APAC regions
ALTER TABLE ap_customers CONFIGURE ZONE USING 
  constraints = '[+jurisdiction=APAC]',
  lease_preferences = '[[+jurisdiction=APAC]]';

-- Compliance audit query
SELECT 
  table_name,
  COUNT(*) as replica_count,
  COUNT(*) FILTER (WHERE jurisdiction = expected_jurisdiction) as compliant_replicas,
  COUNT(*) FILTER (WHERE jurisdiction != expected_jurisdiction) as non_compliant_replicas
FROM (
  SELECT 
    r.table_name,
    CASE WHEN r.table_name LIKE 'us_%' THEN 'US'
         WHEN r.table_name LIKE 'eu_%' THEN 'EU'
         WHEN r.table_name LIKE 'ap_%' THEN 'APAC'
    END as expected_jurisdiction,
    n.jurisdiction
  FROM crdb_internal.ranges r
  JOIN crdb_internal.nodes n ON r.replica_node_id = n.node_id
) compliance_check
GROUP BY table_name, expected_jurisdiction;
```

Q206: How do I implement field-level encryption for sensitive data?

1. Identify sensitive fields requiring encryption.
2. Encrypt at application layer before storage.
3. Use HSM or KMS for key management.
4. Implement transparent decryption on read.
5. Audit encryption key access.
6. Monitor for unencrypted sensitive data.

Script:
```python
# Field-level encryption implementation
from cryptography.fernet import Fernet
from google.cloud import kms
import base64

class FieldEncryption:
    def __init__(self, kms_key_path):
        self.kms_client = kms.KeyManagementServiceClient()
        self.kms_key_path = kms_key_path
    
    def encrypt_field(self, plaintext):
        """Encrypt sensitive field with KMS"""
        response = self.kms_client.encrypt(
            request={'name': self.kms_key_path, 
                    'plaintext': plaintext.encode()}
        )
        return base64.b64encode(response.ciphertext).decode()
    
    def decrypt_field(self, ciphertext):
        """Decrypt sensitive field with KMS"""
        response = self.kms_client.decrypt(
            request={'name': self.kms_key_path,
                    'ciphertext': base64.b64decode(ciphertext)}
        )
        return response.plaintext.decode()

# Usage in database operations
encryption = FieldEncryption('/path/to/kms/key')

# Insert with encryption
def create_user(user_data):
    encrypted_ssn = encryption.encrypt_field(user_data['ssn'])
    encrypted_email = encryption.encrypt_field(user_data['email'])
    
    db.execute(
        "INSERT INTO users (name, encrypted_ssn, encrypted_email) VALUES (?, ?, ?)",
        user_data['name'], encrypted_ssn, encrypted_email
    )

# Read with decryption
def get_user(user_id):
    result = db.query("SELECT * FROM users WHERE id = ?", user_id)
    result['ssn'] = encryption.decrypt_field(result['encrypted_ssn'])
    result['email'] = encryption.decrypt_field(result['encrypted_email'])
    return result
```

================================================================================
SECTION 41: ADVANCED PERFORMANCE TUNING
================================================================================

Q207: How do I implement adaptive query performance tuning?

1. Collect query execution statistics continuously.
2. Identify regressed queries automatically.
3. Suggest indexes or plan hints.
4. Implement automatic plan caching.
5. Monitor effectiveness of tuning.
6. Document performance improvements.

Script:
```sql
-- Adaptive tuning baseline
CREATE TABLE query_performance_baseline (
  query_id VARCHAR,
  query_hash VARCHAR,
  query_text VARCHAR,
  baseline_latency_p99 DECIMAL,
  baseline_cpu_ms DECIMAL,
  baseline_rows BIGINT,
  captured_at TIMESTAMP DEFAULT NOW()
);

-- Capture current query stats
CREATE TABLE current_query_stats (
  query_id VARCHAR,
  query_hash VARCHAR,
  current_latency_p99 DECIMAL,
  current_cpu_ms DECIMAL,
  current_rows BIGINT,
  execution_count BIGINT,
  measured_at TIMESTAMP DEFAULT NOW()
);

-- Identify regressed queries
SELECT 
  b.query_id,
  b.query_text,
  b.baseline_latency_p99,
  c.current_latency_p99,
  ROUND(100.0 * (c.current_latency_p99 - b.baseline_latency_p99) / b.baseline_latency_p99, 2) as regression_percent,
  c.execution_count,
  CASE 
    WHEN (c.current_latency_p99 - b.baseline_latency_p99) > (b.baseline_latency_p99 * 0.2) THEN 'REGRESSED'
    ELSE 'OK'
  END as status
FROM query_performance_baseline b
JOIN current_query_stats c ON b.query_id = c.query_id
WHERE (c.current_latency_p99 - b.baseline_latency_p99) > 0
ORDER BY regression_percent DESC;

-- Suggest index based on query patterns
SELECT 
  table_name,
  COUNT(*) as filter_count,
  STRING_AGG(DISTINCT column_name, ', ') as frequently_filtered_columns
FROM query_analysis
WHERE query_type = 'WHERE'
GROUP BY table_name
HAVING COUNT(*) > 100
ORDER BY filter_count DESC;
```

Q208: How do I implement query plan forcing and optimization hints?

1. Capture current query plans.
2. Force specific plans when regression detected.
3. Implement query hints for plan control.
4. Monitor plan changes over time.
5. Document plan decisions.
6. Test plan changes before deployment.

Script:
```sql
-- Store good query plans
CREATE TABLE saved_query_plans (
  plan_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_hash VARCHAR,
  query_text VARCHAR,
  plan_json JSONB,
  expected_latency_ms DECIMAL,
  status VARCHAR,  -- ACTIVE, ARCHIVED
  created_at TIMESTAMP DEFAULT NOW()
);

-- Force specific plan using hints
SELECT /*+ FORCE_INDEX(orders idx_orders_customer_date) */ *
FROM orders
WHERE customer_id = $1 AND created_at > now() - interval '30 days'
ORDER BY created_at DESC;

-- Zigzag join hint
SELECT /*+ FORCE_ZIGZAG_JOIN(orders, customers) */ *
FROM orders
JOIN customers ON orders.customer_id = customers.id
WHERE orders.status = 'pending';

-- Lookup join hint
SELECT /*+ FORCE_LOOKUP_JOIN(a, b) */ *
FROM large_table a
JOIN small_table b ON a.id = b.id;

-- Hash join hint
SELECT /*+ FORCE_HASH_JOIN(t1, t2) */ *
FROM table1 t1
JOIN table2 t2 ON t1.id = t2.id;

-- Monitor plan forcing effectiveness
SELECT 
  query_hash,
  COUNT(*) as execution_count,
  AVG(execution_latency_ms) as avg_latency,
  CASE WHEN STDDEV(execution_latency_ms) < 10 THEN 'STABLE'
       ELSE 'VARIABLE'
  END as plan_stability
FROM forced_plan_executions
GROUP BY query_hash
ORDER BY avg_latency DESC;
```

================================================================================
SECTION 42: FINAL COMPREHENSIVE REFERENCE
================================================================================

Q209: What is the ultimate CockroachDB production configuration template?

Complete Production Setup:
```bash
# Optimal node startup configuration
cockroach start \
  --certs-dir=/etc/cockroach/certs \
  --node-id=$NODE_ID \
  --locality=region=$REGION,zone=$ZONE,rack=$RACK \
  --listen-addr=$PRIVATE_IP:26257 \
  --advertise-addr=$PUBLIC_IP:26257 \
  --http-addr=$PRIVATE_IP:8080 \
  --join=$NODE1:26257,$NODE2:26257,$NODE3:26257 \
  --cache=25% \
  --max-sql-memory=25% \
  --store=type=ssd,path=/var/lib/cockroach \
  --log-dir=/var/log/cockroach \
  --pid-file=/var/run/cockroach.pid \
  --max-offset=500ms \
  --clock-source=monotonic \
  --disable-http-basic-auth=false \
  --enable-http-request-logging=true
```

Production SQL Configuration:
```sql
-- Security
SET CLUSTER SETTING server.ssl.cert_validation='require';
SET CLUSTER SETTING server.ssl.ca_cert='/path/to/ca.crt';
SET CLUSTER SETTING server.auth.log.enabled=true;

-- Performance
SET CLUSTER SETTING sql.defaults.optimizer_use_forced_plans=false;
SET CLUSTER SETTING kv.allocator.mode='balanced';
SET CLUSTER SETTING sql.metrics.statement_details.enabled=true;

-- Replication
SET CLUSTER SETTING num_replicas=3;
SET CLUSTER SETTING sql.defaults.inter_lease_transfer_min_duration='5m';

-- Backup
SET CLUSTER SETTING kv.range_merge.queue_enabled=true;
SET CLUSTER SETTING backup.gc_delete_rate='128 MiB';

-- Monitoring
SET CLUSTER SETTING server.time_until_store_dead='5m';
SET CLUSTER SETTING server.clock.forward_jump_check_enabled=true;
```

Q210: What are final recommendations for successful CockroachDB operations?

**Foundational Practices:**
1. Automate everything: deployment, backup, recovery, scaling.
2. Monitor relentlessly: know your baseline, alert on anomalies.
3. Test catastrophically: failure scenarios, recovery procedures.
4. Document exhaustively: architecture, procedures, decisions.
5. Train continuously: knowledge transfer, certifications.

**Operational Excellence:**
1. Implement SLO-driven operations.
2. Use infrastructure-as-code for reproducibility.
3. Automate incident response with runbooks.
4. Conduct regular postmortems.
5. Measure and track operational metrics.

**Security and Compliance:**
1. Encrypt at rest and in transit.
2. Implement RBAC for access control.
3. Audit all access and changes.
4. Regular security assessments.
5. Maintain compliance documentation.

**Performance and Scalability:**
1. Design for scale from day one.
2. Optimize queries proactively.
3. Implement caching strategically.
4. Monitor and plan capacity.
5. Automate scaling decisions.

**Business Continuity:**
1. Define and measure RTO/RPO.
2. Test DR procedures quarterly.
3. Maintain geographic redundancy.
4. Implement automated failover.
5. Document recovery procedures.

COCKROACHDB ADMINISTRATION GUIDE - ULTIMATE REFERENCE SUPPLEMENT
Advanced Deep-Dive Topics, Architecture Patterns, Case Studies, and Operational Frameworks

================================================================================
SECTION 43: DEEP-DIVE ARCHITECTURAL PATTERNS
================================================================================

Q211: How do I design multi-region architecture for global applications?

1. Implement follower replicas in each region for local reads.
2. Designate primary region for writes.
3. Use lease preferences to control leader placement.
4. Monitor cross-region latency and data movement.
5. Implement region-aware application routing.
6. Plan for graceful degradation on region failure.

Architecture Pattern:
```
┌─────────────────────────────────────────────────────────────┐
│                    Global Application                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  US Region            EU Region            APAC Region       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Replica 1    │    │ Replica 2    │    │ Replica 3    │   │
│  │ (Leader)     │    │ (Follower)   │    │ (Follower)   │   │
│  │              │    │              │    │              │   │
│  │ Primary      │◄───┼─ Replication ─►   │              │   │
│  │ Writes       │    │ Lag: 100ms   │    │              │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│        ▲                    ▲                    ▲            │
│        │ Follower Reads     │ Follower Reads    │            │
│        │ (Local Replicas)   │ (Local Replicas)  │            │
│        │                    │                   │            │
│  ┌─────┴────────┐      ┌────┴─────────┐    ┌───┴────────┐  │
│  │  App Tier    │      │  App Tier    │    │  App Tier  │  │
│  │  US          │      │  EU          │    │  APAC      │  │
│  └──────────────┘      └──────────────┘    └────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

SQL Implementation:
```sql
-- Primary region (US) - handles writes
ALTER TABLE critical_data CONFIGURE ZONE USING
  num_replicas = 3,
  constraints = '[+region=us-east, +region=us-west]',
  lease_preferences = '[[+region=us-east]]';

-- Secondary regions - follower reads for local latency
-- EU region reads use follower replicas
SET SESSION enable_follower_reads = true;
SELECT * FROM critical_data WHERE region = 'EU';

-- APAC region reads use follower replicas
SET SESSION enable_follower_reads = true;
SELECT * FROM critical_data WHERE region = 'APAC';

-- Monitor cross-region replication lag
SELECT 
  source_region,
  dest_region,
  AVG(replication_lag_ms) as avg_lag,
  MAX(replication_lag_ms) as max_lag
FROM crdb_internal.replication_metrics
WHERE timestamp > now() - interval '1 hour'
GROUP BY source_region, dest_region
ORDER BY max_lag DESC;

-- Graceful degradation: route to local region on primary failure
-- (Application logic)
def get_data_with_failover(data_id, preferred_region):
    try:
        # Try primary region write
        return db.query_primary_region(data_id)
    except PrimaryRegionUnavailable:
        # Fall back to local region
        return db.query_follower_region(data_id, preferred_region)
```

Q212: How do I implement hub-and-spoke database architecture?

1. Central hub cluster for global coordination.
2. Regional spoke clusters for local operations.
3. Hub-to-spoke replication for consistency.
4. Spoke-to-hub sync for global aggregates.
5. Handle network partition gracefully.
6. Monitor hub-spoke lag.

Architecture Diagram:
```
              ┌──────────────────┐
              │   Hub Cluster    │
              │  (Coordinator)   │
              │                  │
              └────────┬─────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    ┌───▼───┐      ┌──▼────┐    ┌───▼──┐
    │US Spoke│      │EU Spoke│    │APAC  │
    │Cluster │      │Cluster │    │Spoke │
    └────────┘      └────────┘    └──────┘
```

SQL Implementation:
```sql
-- Hub cluster schema
CREATE TABLE global_config (
  config_key VARCHAR PRIMARY KEY,
  config_value JSONB,
  version INT,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Hub-to-spoke replication
CREATE CHANGEFEED FOR TABLE global_config
INTO 'kafka://hub-kafka/global-config'
WITH format='json';

-- Spoke cluster local schema (US)
CREATE TABLE us_orders (
  order_id UUID PRIMARY KEY,
  customer_id UUID,
  amount DECIMAL,
  created_at TIMESTAMP DEFAULT NOW(),
  synced_to_hub BOOLEAN DEFAULT false
);

-- Spoke-to-hub sync for aggregates
CREATE SCHEDULE sync_us_aggregates FOR
  INSERT INTO hub_cluster.daily_sales_summary
    SELECT DATE(created_at), COUNT(*), SUM(amount)
    FROM us_orders
    WHERE created_at > now() - interval '1 day'
    GROUP BY DATE(created_at)
  RECURRING EVERY 1 hour;

-- Handle hub unavailability (local fallback)
-- Application queues writes locally if hub unreachable
CREATE TABLE us_orders_pending (
  order_id UUID PRIMARY KEY,
  data JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  synced_to_hub BOOLEAN DEFAULT false
);
```

Q213: How do I implement time-series database optimization?

1. Use range partitioning by timestamp.
2. Implement TTL-based data retention.
3. Compress old data partitions.
4. Create specialized indexes for time queries.
5. Monitor partition growth and splits.
6. Archive old data to cold storage.

SQL Implementation:
```sql
-- Time-series optimized schema
CREATE TABLE metrics (
  metric_id UUID,
  timestamp TIMESTAMP NOT NULL,
  metric_name VARCHAR NOT NULL,
  host_id UUID NOT NULL,
  value DECIMAL,
  tags JSONB,
  PRIMARY KEY (timestamp DESC, metric_id)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_current VALUES FROM (CURRENT_DATE) TO (CURRENT_DATE + 1),
  PARTITION p_yesterday VALUES FROM (CURRENT_DATE - 1) TO (CURRENT_DATE),
  PARTITION p_7days VALUES FROM (CURRENT_DATE - 7) TO (CURRENT_DATE - 1),
  PARTITION p_30days VALUES FROM (CURRENT_DATE - 30) TO (CURRENT_DATE - 7),
  PARTITION p_old VALUES FROM (MINVALUE) TO (CURRENT_DATE - 30)
);

-- Specialized indexes for time-series
CREATE INDEX idx_metrics_host_time ON metrics(host_id, timestamp DESC);
CREATE INDEX idx_metrics_name_time ON metrics(metric_name, timestamp DESC);
CREATE INDEX idx_metrics_tags ON metrics USING GIN(tags);

-- TTL-based retention
CREATE TABLE metric_ttl (
  metric_name VARCHAR PRIMARY KEY,
  retention_days INT
);

INSERT INTO metric_ttl VALUES 
  ('cpu_usage', 90),
  ('memory_usage', 90),
  ('disk_usage', 180),
  ('network_bytes', 365);

-- Automatic data cleanup
CREATE PROCEDURE cleanup_old_metrics() AS $$
BEGIN
  DELETE FROM metrics 
  WHERE timestamp < now() - (
    SELECT retention_days FROM metric_ttl 
    WHERE metric_name = metrics.metric_name
  ) * interval '1 day';
END
$$ LANGUAGE plpgsql;

-- Schedule cleanup
CREATE SCHEDULE cleanup_metrics FOR CALL cleanup_old_metrics()
RECURRING EVERY 1 day
FIRST RUN 'tomorrow 02:00:00';

-- Efficient time-series queries
SELECT 
  DATE_TRUNC('minute', timestamp)::timestamp as minute,
  host_id,
  AVG(value) as avg_value,
  MAX(value) as max_value,
  MIN(value) as min_value,
  STDDEV(value) as stddev_value
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND host_id = $host_id
  AND timestamp > now() - interval '7 days'
GROUP BY DATE_TRUNC('minute', timestamp), host_id
ORDER BY minute DESC;
```

Q214: How do I implement real-time OLAP with CockroachDB?

1. Use changefeeds for real-time data ingestion.
2. Implement streaming materialized views.
3. Design denormalized schemas for analytics.
4. Use aggregate storage for fast queries.
5. Monitor freshness and lag.
6. Implement incremental refresh strategies.

SQL Implementation:
```sql
-- Source transactional table
CREATE TABLE orders (
  order_id UUID PRIMARY KEY,
  customer_id UUID,
  product_id UUID,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Real-time analytics tables
CREATE TABLE daily_revenue (
  date DATE,
  revenue DECIMAL,
  order_count INT,
  unique_customers INT,
  avg_order_value DECIMAL,
  PRIMARY KEY (date DESC)
);

CREATE TABLE product_analytics (
  product_id UUID,
  total_sales DECIMAL,
  unit_count INT,
  revenue_rank INT,
  last_updated TIMESTAMP,
  PRIMARY KEY (product_id)
);

-- Streaming update via changefeed
CREATE CHANGEFEED FOR TABLE orders
INTO 'kafka://analytics-broker/orders-stream'
WITH format='json', resolved='10s';

-- Real-time materialized view (incremental)
CREATE PROCEDURE update_analytics() AS $$
BEGIN
  -- Update daily revenue
  MERGE INTO daily_revenue dr
  USING (
    SELECT 
      DATE(created_at) as date,
      SUM(amount) as revenue,
      COUNT(*) as order_count,
      COUNT(DISTINCT customer_id) as unique_customers,
      AVG(amount) as avg_order_value
    FROM orders
    WHERE created_at > now() - interval '1 day'
    GROUP BY DATE(created_at)
  ) src
  ON dr.date = src.date
  WHEN MATCHED THEN UPDATE SET 
    revenue = src.revenue,
    order_count = src.order_count,
    unique_customers = src.unique_customers,
    avg_order_value = src.avg_order_value
  WHEN NOT MATCHED THEN INSERT VALUES 
    (src.date, src.revenue, src.order_count, 
     src.unique_customers, src.avg_order_value);
  
  -- Update product analytics
  MERGE INTO product_analytics pa
  USING (
    SELECT 
      product_id,
      SUM(amount) as total_sales,
      COUNT(*) as unit_count,
      ROW_NUMBER() OVER (ORDER BY SUM(amount) DESC) as revenue_rank
    FROM orders
    WHERE created_at > now() - interval '30 days'
    GROUP BY product_id
  ) src
  ON pa.product_id = src.product_id
  WHEN MATCHED THEN UPDATE SET
    total_sales = src.total_sales,
    unit_count = src.unit_count,
    revenue_rank = src.revenue_rank,
    last_updated = NOW()
  WHEN NOT MATCHED THEN INSERT VALUES
    (src.product_id, src.total_sales, src.unit_count, 
     src.revenue_rank, NOW());
END
$$ LANGUAGE plpgsql;

-- Schedule real-time updates
CREATE SCHEDULE update_analytics_realtime FOR CALL update_analytics()
RECURRING EVERY 10 seconds;

-- Query real-time analytics
SELECT 
  date,
  revenue,
  order_count,
  unique_customers,
  avg_order_value,
  LAG(revenue) OVER (ORDER BY date) as prev_day_revenue,
  ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY date)) / LAG(revenue) OVER (ORDER BY date), 2) as revenue_growth_percent
FROM daily_revenue
WHERE date >= CURRENT_DATE - 30
ORDER BY date DESC;
```

================================================================================
SECTION 44: ADVANCED CASE STUDIES AND SOLUTIONS
================================================================================

Q215: Case Study: Scaling from single-region to multi-region production (100K+ QPS)

Scenario: E-commerce platform growing from single US region to global presence.

Challenge:
- Initial: Single region (3 nodes), handling 10K QPS
- Target: Multi-region (9 nodes across 3 regions), handle 100K QPS
- Constraint: Zero downtime during migration

Solution Architecture:
```
Phase 1 (Week 1-2): Prepare Infrastructure
├─ Provision nodes in EU and APAC regions
├─ Configure networking (VPC peering, firewalls)
├─ Setup replication channels
└─ Run baseline performance tests

Phase 2 (Week 3-4): Parallel Cluster
├─ Create secondary multi-region cluster
├─ Enable two-way replication
├─ Validate data consistency
└─ Run load testing

Phase 3 (Week 5): Gradual Traffic Migration
├─ Day 1-2: Route 10% traffic to multi-region
├─ Day 3-4: Route 25% traffic
├─ Day 5: Route 50% traffic
├─ Day 6-7: Route 100% traffic, disable single-region

Phase 4 (Week 6-8): Optimization
├─ Tune replication settings
├─ Optimize query patterns for new topology
├─ Monitor and stabilize
└─ Decommission old cluster
```

SQL Migration Plan:
```sql
-- Phase 1: Create new multi-region cluster schema
-- (New cluster initialized with replication)

-- Phase 2: Enable bidirectional replication
ALTER CLUSTER SET REPLICATION FACTOR = 2;

-- Setup replication from old to new
CREATE CHANGEFEED FOR TABLE all_tables
INTO 'kafka://new-cluster/migration-stream'
WITH format='avro';

-- Phase 3: Monitor replication lag before cutover
SELECT 
  table_name,
  COUNT(*) as replicated_rows,
  MAX(replication_lag_seconds) as max_lag
FROM cluster_replication_status
GROUP BY table_name
HAVING MAX(replication_lag_seconds) < 1;  -- < 1 second lag

-- Phase 4: Gradual traffic shift (application routing)
-- Application routes based on configuration
def get_db_connection(traffic_percentage_new_cluster):
    if random.random() < (traffic_percentage_new_cluster / 100):
        return connect_to_new_cluster()
    else:
        return connect_to_old_cluster()

-- Week 1: 10%
UPDATE traffic_routing SET new_cluster_percent = 10;

-- Week 2: 25%
UPDATE traffic_routing SET new_cluster_percent = 25;

-- Week 3: 50%
UPDATE traffic_routing SET new_cluster_percent = 50;

-- Week 4: 100% (old cluster disabled)
UPDATE traffic_routing SET new_cluster_percent = 100;
```

Results:
- Successfully scaled from 10K to 100K QPS
- Zero customer-facing downtime
- Query latency improved 40% due to follower reads
- Cross-region consistency maintained throughout
- Total migration time: 8 weeks
- No data loss or corruption

Q216: Case Study: Recovering from data center failure with PITR

Scenario: Primary data center suffered catastrophic power failure at 14:32 UTC.

Timeline:
- 14:32:00 - Power failure detected
- 14:32:30 - Quorum lost, cluster went read-only
- 14:35:00 - Issue escalated, investigation started
- 14:45:00 - Decision made to restore from backup
- 15:00:00 - Restore initiated
- 15:45:00 - Validation complete, traffic restored
- 16:00:00 - Full recovery with 28-minute downtime

Recovery Procedure:
```bash
#!/bin/bash
# Data center failure recovery

echo "=== DC Failure Recovery Started ==="
echo "Failure Time: 2024-01-15 14:32:00"
echo "Detection Time: 2024-01-15 14:32:30"

# Step 1: Identify last good backup
LAST_BACKUP=$(cockroach sql --certs-dir=certs -c "
  SELECT backup_path, backup_time 
  FROM system.backups 
  WHERE status = 'COMPLETE' 
  ORDER BY backup_time DESC 
  LIMIT 1
")
echo "Last Good Backup: $LAST_BACKUP at 2024-01-15 14:00:00"

# Step 2: Provision new cluster in secondary DC
# (Infrastructure automation)
echo "Provisioning new cluster..."
terraform apply -auto-approve

# Step 3: Restore from backup
echo "Restoring from backup..."
cockroach sql --certs-dir=certs <<EOF
-- Restore full cluster
RESTORE FROM 's3://backup-bucket/backup-2024-01-15-14-00-00/';

-- Verify restore integrity
SELECT COUNT(*) as table_count FROM information_schema.tables;
SELECT COUNT(*) as total_rows FROM (
  SELECT 1 FROM customers UNION ALL
  SELECT 1 FROM orders UNION ALL
  SELECT 1 FROM products
);

-- Verify critical data
SELECT * FROM system.node_liveness;
SELECT * FROM system.lease_holder_info;
EOF

# Step 4: Point-in-time recovery to just before failure
cockroach sql --certs-dir=certs <<EOF
RESTORE FROM 's3://backup-bucket/incremental/' 
  AS OF SYSTEM TIME '2024-01-15 14:32:00'
  WITH revision_history;
EOF

# Step 5: Redirect traffic
echo "Redirecting traffic to new cluster..."
# Update DNS/load balancer to point to new cluster

# Step 6: Verify operations
echo "Verifying cluster health..."
cockroach sql --certs-dir=certs -c "
  SELECT node_id, is_live FROM crdb_internal.nodes;
  SELECT COUNT(*) as healthy_ranges FROM crdb_internal.ranges 
  WHERE unavailable_replicas = 0;
"

echo "=== Recovery Complete ==="
echo "Data Loss: 32 minutes (14:32 to 14:00)"
echo "Downtime: 28 minutes (14:32 to 15:00)"
echo "Total RTO: 28 minutes"
echo "Total RPO: 32 minutes"
```

Post-Incident Actions:
```sql
-- Analyze what data was lost
SELECT 
  table_name,
  COUNT(*) as rows_in_backup,
  (SELECT COUNT(*) FROM [table]) as rows_current,
  COUNT(*) - (SELECT COUNT(*) FROM [table]) as rows_lost
FROM backup_tables;

-- Implement improvements
-- 1. More frequent backups (every 15 min instead of 30 min)
ALTER SCHEDULE backup_schedule SET 
  RECURRING EVERY 15 minutes;

-- 2. Cross-region backup copies
CREATE SCHEDULE backup_to_secondary_region FOR
  BACKUP INTO 's3://backup-bucket-secondary/backup-{date-time}'
  RECURRING EVERY 1 hour;

-- 3. Automatic failover to secondary DC
CREATE TABLE failover_config (
  primary_dc VARCHAR,
  secondary_dc VARCHAR,
  auto_failover BOOLEAN,
  failover_threshold_seconds INT
);

-- 4. Enhanced monitoring for DC health
CREATE ALERT dc_health_check AS
SELECT COUNT(*) as failed_nodes
FROM crdb_internal.nodes
WHERE NOT is_live
HAVING COUNT(*) >= 2
WITH severity = 'CRITICAL';
```

================================================================================
SECTION 45: OPERATIONAL FRAMEWORKS AND TOOLING
================================================================================

Q217: How do I build comprehensive operational dashboards?

Framework for multi-tiered dashboards:

```
Executive Dashboard (Business View)
├─ Revenue Impact (SLA compliance → $ impact)
├─ Customer Impact (% of users affected)
├─ Infrastructure Health (simplified traffic light)
└─ Incident Trends (MTTR, frequency, severity)

Operations Dashboard (Technical View)
├─ Cluster Health (node status, replication lag)
├─ Query Performance (latency P50/P99, throughput)
├─ Resource Utilization (CPU, memory, disk)
├─ Backup/Recovery Status (last backup, freshness)
└─ Alerts & Incidents (active, history)

Developer Dashboard (Application View)
├─ Query Latency (by service, by query type)
├─ Error Rates (by service, by error type)
├─ Connection Pool Status (usage, exhaustion)
└─ Hot Tables (contention, slow queries)

Troubleshooting Dashboard (Support View)
├─ Node Logs (searchable, filterable)
├─ Query Traces (execution details, plans)
├─ Metric Time Series (drill-down capability)
└─ Historical Comparisons (baseline vs current)
```

Implementation (Grafana/Prometheus):
```python
# Python script to generate dashboard JSON
import json
from grafana_api.grafana_face import GrafanaFace

class CockroachDashboardGenerator:
    def __init__(self):
        self.grafana = GrafanaFace(
            auth=('admin', 'password'),
            host='http://grafana:3000'
        )
    
    def create_executive_dashboard(self):
        panels = [
            self.create_slo_compliance_panel(),
            self.create_customer_impact_panel(),
            self.create_incident_trends_panel()
        ]
        
        dashboard = {
            'title': 'CockroachDB Executive Dashboard',
            'panels': panels,
            'refresh': '30s',
            'templating': {'list': []}
        }
        
        return self.grafana.dashboard.create_dashboard(
            dashboard=dashboard,
            overwrite=True
        )
    
    def create_slo_compliance_panel(self):
        return {
            'title': 'SLO Compliance',
            'type': 'gauge',
            'targets': [{
                'expr': 'slo_compliance_percent',
                'refId': 'A'
            }],
            'fieldConfig': {
                'defaults': {
                    'thresholds': {
                        'mode': 'absolute',
                        'steps': [
                            {'color': 'red', 'value': 0},
                            {'color': 'yellow', 'value': 95},
                            {'color': 'green', 'value': 99.9}
                        ]
                    }
                }
            }
        }
    
    def create_customer_impact_panel(self):
        return {
            'title': 'Customer Impact',
            'type': 'stat',
            'targets': [{
                'expr': 'affected_customers_percent',
                'refId': 'A'
            }]
        }
    
    def create_incident_trends_panel(self):
        return {
            'title': 'Incident Trends (Last 30 Days)',
            'type': 'timeseries',
            'targets': [{
                'expr': 'rate(incidents_total[1d])',
                'refId': 'A'
            }]
        }
```

Q218: How do I implement ChatOps for database operations?

Slack Integration for Database Operations:

```python
# Slack bot for CockroachDB operations
import slack
from cockroachdb import connect

class CockroachSlackBot:
    def __init__(self, bot_token):
        self.client = slack.WebClient(token=bot_token)
        self.db = connect()
    
    def handle_commands(self, command, args, user):
        """Handle database commands from Slack"""
        
        handlers = {
            'status': self.get_cluster_status,
            'health': self.get_cluster_health,
            'queries': self.get_slow_queries,
            'backup': self.check_backup_status,
            'scale': self.trigger_scaling,
            'failover': self.trigger_failover,
            'runbook': self.get_runbook
        }
        
        if command in handlers:
            result = handlers[command](args)
            self.post_response(user, result)
    
    def get_cluster_status(self, args):
        """Get current cluster status"""
        result = self.db.query("""
            SELECT node_id, address, is_live, 
              ROUND(100.0 * capacity_used / capacity_total, 2) as disk_percent
            FROM crdb_internal.nodes n
            LEFT JOIN crdb_internal.stores s ON n.node_id = s.node_id
        """)
        
        status_text = "🟢 Cluster Status\n"
        for row in result:
            status = "🟢 LIVE" if row['is_live'] else "🔴 DOWN"
            status_text += f"{status} Node {row['node_id']}: {row['disk_percent']}% disk\n"
        
        return status_text
    
    def get_slow_queries(self, args):
        """Get slowest queries"""
        result = self.db.query("""
            SELECT query, latency_p99, execution_count
            FROM crdb_internal.node_statement_statistics
            ORDER BY latency_p99 DESC
            LIMIT 5
        """)
        
        slow_text = "🐌 Slow Queries (P99 Latency)\n"
        for row in result:
            slow_text += f"⏱️ {row['latency_p99']}ms - {row['execution_count']} executions\n"
            slow_text += f"   {row['query'][:80]}...\n"
        
        return slow_text
    
    def trigger_scaling(self, args):
        """Scale cluster (with approval)"""
        if args and args[0] in ['up', 'down']:
            self.client.chat_postMessage(
                channel='#database-ops',
                text=f"Scaling {args[0]}: This requires approval. React with ✅ to proceed."
            )
        else:
            return "Usage: /cockroach scale up|down"
    
    def post_response(self, user, message):
        """Post response back to Slack"""
        self.client.chat_postMessage(
            channel=user,
            text=message,
            blocks=[{
                'type': 'section',
                'text': {'type': 'mrkdwn', 'text': message}
            }]
        )

# Usage in Slack
# @CockroachBot status
# @CockroachBot health
# @CockroachBot queries
# @CockroachBot backup
```

Q219: How do I implement comprehensive logging and log aggregation?

Logging Framework:

```yaml
# Filebeat configuration for log shipping
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/cockroach/cockroach.log
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after
  
  fields:
    service: cockroachdb
    environment: production
    cluster: primary

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "cockroachdb-%{+yyyy.MM.dd}"

logging.level: info
logging.selectors: ["*"]
```

Log Analysis Queries (Elasticsearch/Kibana):

```json
// Find authentication failures
{
  "query": {
    "bool": {
      "must": [
        {"match": {"message": "authentication failed"}},
        {"range": {"timestamp": {"gte": "now-24h"}}}
      ]
    }
  }
}

// Query latency anomalies
{
  "query": {
    "bool": {
      "must": [
        {"match": {"event_type": "query"}},
        {"range": {"latency_ms": {"gte": 1000}}}
      ]
    }
  },
  "aggs": {
    "by_query": {
      "terms": {"field": "query_template.keyword"},
      "aggs": {
        "avg_latency": {"avg": {"field": "latency_ms"}}
      }
    }
  }
}

// Replication lag tracking
{
  "query": {
    "bool": {
      "must": [
        {"match": {"event_type": "replication"}},
        {"range": {"lag_seconds": {"gte": 5}}}
      ]
    }
  },
  "aggs": {
    "lag_over_time": {
      "date_histogram": {"field": "timestamp", "interval": "1m"},
      "aggs": {
        "max_lag": {"max": {"field": "lag_seconds"}}
      }
    }
  }
}
```

================================================================================
SECTION 46: FINAL OPERATIONAL EXCELLENCE FRAMEWORK
================================================================================

Q220: What is the ultimate operational excellence framework for CockroachDB?

Comprehensive Framework:

```
┌────────────────────────────────────────────────────────┐
│         OPERATIONAL EXCELLENCE FRAMEWORK               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. PLANNING & ARCHITECTURE                           │
│  ├─ Capacity planning (1.5x headroom)                 │
│  ├─ Architecture design (resilience, scalability)     │
│  ├─ Disaster recovery planning (RTO/RPO targets)      │
│  └─ Cost optimization strategy                        │
│                                                        │
│  2. IMPLEMENTATION                                    │
│  ├─ Infrastructure-as-code (Terraform)                │
│  ├─ Automated deployment pipelines                    │
│  ├─ Security hardening (encryption, RBAC)             │
│  └─ Monitoring/alerting setup                         │
│                                                        │
│  3. OPERATIONS                                        │
│  ├─ Runbook documentation (100+ procedures)           │
│  ├─ Change management (approval workflow)             │
│  ├─ Incident response (SOC/SRE processes)            │
│  └─ Continuous improvement (postmortems)              │
│                                                        │
│  4. MEASUREMENT                                       │
│  ├─ SLO/SLA tracking (99.9%+ availability)            │
│  ├─ MTTR/MTBF monitoring                              │
│  ├─ Cost tracking per service                         │
│  └─ Team efficiency metrics                           │
│                                                        │
│  5. CONTINUOUS IMPROVEMENT                            │
│  ├─ Quarterly reviews (performance, costs)            │
│  ├─ Technology upgrades (CockroachDB versions)        │
│  ├─ Process automation (reduce manual work)           │
│  └─ Team training (skills development)                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

Key Metrics to Track:

```sql
-- Create metrics tracking database
CREATE TABLE operational_metrics (
  metric_date DATE,
  metric_name VARCHAR,
  metric_value DECIMAL,
  target_value DECIMAL,
  status VARCHAR,  -- MET, MISSED
  notes VARCHAR
);

-- Essential operational metrics
INSERT INTO operational_metrics VALUES

-- Availability metrics
('2024-01-15', 'cluster_availability_percent', 99.95, 99.90, 'MET', NULL),
('2024-01-15', 'planned_downtime_hours', 0, 0, 'MET', NULL),
('2024-01-15', 'unplanned_downtime_hours', 0.2, 0.5, 'MET', NULL),

-- Performance metrics
('2024-01-15', 'query_p99_latency_ms', 85, 100, 'MET', NULL),
('2024-01-15', 'backup_completion_time_min', 45, 60, 'MET', NULL),
('2024-01-15', 'cluster_rebalance_time_min', 120, 180, 'MET', NULL),

-- Operational metrics
('2024-01-15', 'mttr_minutes', 8, 15, 'MET', 'Mean Time To Recovery'),
('2024-01-15', 'mtbf_days', 45, 30, 'MET', 'Mean Time Between Failures'),
('2024-01-15', 'deployment_success_rate', 100, 99.5, 'MET', NULL),

-- Cost metrics
('2024-01-15', 'cost_per_qps', 0.015, 0.020, 'MET', 'USD per QPS'),
('2024-01-15', 'storage_cost_per_gb', 0.002, 0.003, 'MET', 'USD per GB/month'),

-- Team metrics
('2024-01-15', 'on_call_hours_per_engineer', 168, 168, 'MET', 'Monthly'),
('2024-01-15', 'runbook_coverage_percent', 95, 90, 'MET', '% of procedures documented'),
('2024-01-15', 'team_training_hours', 20, 20, 'MET', 'Per person per quarter');

-- Track trends over time
SELECT 
  metric_name,
  metric_date,
  metric_value,
  target_value,
  ROUND(100.0 * (metric_value - target_value) / target_value, 2) as variance_percent,
  LAG(metric_value) OVER (PARTITION BY metric_name ORDER BY metric_date) as previous_value,
  CASE 
    WHEN metric_value < LAG(metric_value) OVER (PARTITION BY metric_name ORDER BY metric_date) 
    THEN '📉 Declining'
    WHEN metric_value > LAG(metric_value) OVER (PARTITION BY metric_name ORDER BY metric_date)
    THEN '📈 Improving'
    ELSE '➡️ Stable'
  END as trend
FROM operational_metrics
WHERE metric_date >= CURRENT_DATE - 30
ORDER BY metric_date DESC, metric_name;
```

Quarterly Review Checklist:
```
Q1/Q2/Q3/Q4 Operational Review
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ Performance Review
  □ Review SLO compliance (target: 99.9%+)
  □ Analyze query latency trends
  □ Review cluster scaling decisions
  □ Identify optimization opportunities

□ Reliability Review
  □ MTTR analysis (target: <15 minutes)
  □ Incident postmortem review
  □ Disaster recovery test results
  □ Backup/restore success rate

□ Cost Review
  □ Cloud infrastructure costs
  □ Compute optimization (rightsizing)
  □ Storage utilization
  □ Network/data transfer costs

□ Security Review
  □ Security audit results
  □ Compliance status (SOC2, GDPR, etc)
  □ Access control review
  □ Encryption key rotation

□ Team Development
  □ Training completed
  □ Certifications earned
  □ New skills acquired
  □ Knowledge transfer

□ Process Improvement
  □ Runbook updates
  □ Automation additions
  □ Tooling improvements
  □ Documentation updates

Action Items for Next Quarter:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. [Priority 1]
2. [Priority 2]
3. [Priority 3]
```
COCKROACHDB ADMINISTRATION GUIDE - ULTIMATE OPERATIONAL REFERENCE
Advanced Integrations, Automation, Checklists, and Complete Operational Runbooks

================================================================================
SECTION 47: SPECIALIZED INTEGRATIONS AND CONNECTORS
================================================================================

Q221: How do I integrate CockroachDB with Apache Kafka for stream processing?

1. Enable changefeeds to stream data to Kafka topics.
2. Configure Kafka connect for CockroachDB source/sink.
3. Implement stream processing with Flink/Spark.
4. Handle exactly-once semantics for data consistency.
5. Monitor end-to-end latency through pipeline.
6. Implement backpressure handling.

Script:
```sql
-- Enable CDC to Kafka with Avro format
CREATE CHANGEFEED FOR TABLE orders, customers, products
INTO 'kafka://kafka-broker:9092'
WITH 
  format='avro',
  confluent_schema_registry='http://schema-registry:8081',
  topic_prefix='crdb.',
  resolved='10s',
  key_in_value=true;

-- Monitor changefeed status and lag
SELECT 
  job_id,
  status,
  high_water_timestamp,
  NOW() - high_water_timestamp as lag_seconds,
  rows_emitted,
  bytes_emitted
FROM crdb_internal.jobs
WHERE job_type = 'CHANGEFEED'
ORDER BY created DESC;

-- Handle backpressure: pause if lag exceeds threshold
CREATE PROCEDURE check_changefeed_health() AS $$
BEGIN
  IF (SELECT NOW() - high_water_timestamp > interval '1 minute'
      FROM crdb_internal.jobs
      WHERE job_type = 'CHANGEFEED'
      LIMIT 1)
  THEN
    -- Alert and potentially pause
    INSERT INTO alerting (alert_type, message)
    VALUES ('CHANGEFEED_LAG', 'Changefeed lag exceeds 1 minute');
  END IF;
END
$$ LANGUAGE plpgsql;

-- Schedule health checks
CREATE SCHEDULE check_changefeed FOR CALL check_changefeed_health()
RECURRING EVERY 30 seconds;
```

Kafka Consumer Application (Python):
```python
from kafka import KafkaConsumer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroDeserializer
from cockroachdb import connect

class CockroachKafkaConsumer:
    def __init__(self, bootstrap_servers, schema_registry_url):
        self.consumer = KafkaConsumer(
            'crdb.orders', 'crdb.customers',
            bootstrap_servers=bootstrap_servers,
            group_id='crdb-consumer-group',
            value_deserializer=lambda m: m.decode('utf-8'),
            auto_offset_reset='earliest'
        )
        
        self.schema_registry = SchemaRegistryClient({'url': schema_registry_url})
        self.db = connect()
    
    def process_messages(self):
        """Process CDC messages from Kafka"""
        for message in self.consumer:
            self.handle_cdc_event(message.value)
    
    def handle_cdc_event(self, event_json):
        """Handle individual CDC event"""
        import json
        event = json.loads(event_json)
        
        # Extract change metadata
        table = event.get('table')
        after = event.get('after')
        before = event.get('before')
        updated = event.get('updated')
        
        if after:  # INSERT or UPDATE
            self.upsert_to_sink(table, after)
        elif before and not after:  # DELETE
            self.delete_from_sink(table, before)
    
    def upsert_to_sink(self, table, data):
        """Write to downstream system"""
        # Example: write to analytics warehouse
        if table == 'orders':
            # Transform and load to analytics
            self.write_to_analytics_warehouse(data)
    
    def write_to_analytics_warehouse(self, order_data):
        """Example: write to analytics system"""
        # Could write to data warehouse, cache, search engine, etc.
        print(f"Processing order: {order_data['order_id']}")
```

Q222: How do I integrate CockroachDB with Elasticsearch for full-text search?

1. Use CDC to stream data changes to Elasticsearch.
2. Create custom analyzers for your language.
3. Implement dual-write pattern during migration.
4. Keep search index synchronized with database.
5. Handle index refresh and rebalancing.
6. Implement fallback to database search if ES unavailable.

Script:
```python
# Elasticsearch integration with CockroachDB
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
from cockroachdb import connect
import json

class CockroachElasticsearchSync:
    def __init__(self, es_host='localhost:9200', db_host='localhost:26257'):
        self.es = Elasticsearch([es_host])
        self.db = connect(host=db_host)
        self.index_name = 'crdb-products'
    
    def create_index(self):
        """Create index with custom analyzer"""
        index_config = {
            'settings': {
                'number_of_shards': 3,
                'number_of_replicas': 1,
                'analysis': {
                    'analyzer': {
                        'product_analyzer': {
                            'type': 'standard',
                            'stopwords': '_english_'
                        }
                    }
                }
            },
            'mappings': {
                'properties': {
                    'product_id': {'type': 'keyword'},
                    'name': {
                        'type': 'text',
                        'analyzer': 'product_analyzer',
                        'fields': {
                            'keyword': {'type': 'keyword'}
                        }
                    },
                    'description': {
                        'type': 'text',
                        'analyzer': 'product_analyzer'
                    },
                    'category': {'type': 'keyword'},
                    'price': {'type': 'float'},
                    'last_updated': {'type': 'date'}
                }
            }
        }
        
        self.es.indices.create(index=self.index_name, body=index_config)
    
    def sync_all_products(self):
        """Initial sync of all products to Elasticsearch"""
        products = self.db.query('SELECT * FROM products')
        
        docs = []
        for product in products:
            doc = {
                '_index': self.index_name,
                '_id': product['product_id'],
                '_source': {
                    'product_id': product['product_id'],
                    'name': product['name'],
                    'description': product['description'],
                    'category': product['category'],
                    'price': product['price'],
                    'last_updated': product['updated_at']
                }
            }
            docs.append(doc)
        
        # Bulk index
        bulk(self.es, docs, chunk_size=1000)
    
    def handle_cdc_event(self, event):
        """Handle CDC event for Elasticsearch sync"""
        if event.get('after'):
            # INSERT or UPDATE
            self.es.index(
                index=self.index_name,
                id=event['after']['product_id'],
                body=event['after']
            )
        elif event.get('before') and not event.get('after'):
            # DELETE
            self.es.delete(
                index=self.index_name,
                id=event['before']['product_id']
            )
    
    def search_products(self, query, category=None):
        """Full-text search on products"""
        search_body = {
            'query': {
                'bool': {
                    'must': [
                        {'multi_match': {
                            'query': query,
                            'fields': ['name^2', 'description', 'category']
                        }}
                    ]
                }
            }
        }
        
        if category:
            search_body['query']['bool']['filter'] = [
                {'term': {'category': category}}
            ]
        
        results = self.es.search(index=self.index_name, body=search_body)
        return results['hits']['hits']
```

Q223: How do I integrate CockroachDB with Redis for caching?

1. Implement cache-aside pattern for application queries.
2. Use CDC to invalidate cache on data changes.
3. Implement cache warming strategies.
4. Handle cache misses gracefully.
5. Monitor cache hit rates.
6. Implement distributed cache invalidation.

Script:
```python
# Redis caching integration
import redis
import json
from functools import wraps
from cockroachdb import connect

class CockroachRedisCache:
    def __init__(self, redis_host='localhost', redis_port=6379, db_host='localhost'):
        self.redis = redis.Redis(host=redis_host, port=redis_port, decode_responses=True)
        self.db = connect(host=db_host)
        self.cache_ttl = 3600  # 1 hour default
    
    def cache_query(self, ttl=None):
        """Decorator for query result caching"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Generate cache key from function and arguments
                cache_key = self.generate_cache_key(func.__name__, args, kwargs)
                
                # Try cache first
                cached = self.redis.get(cache_key)
                if cached:
                    return json.loads(cached)
                
                # Cache miss: execute query
                result = func(*args, **kwargs)
                
                # Store in cache
                self.redis.setex(
                    cache_key,
                    ttl or self.cache_ttl,
                    json.dumps(result)
                )
                
                return result
            return wrapper
        return decorator
    
    def generate_cache_key(self, func_name, args, kwargs):
        """Generate unique cache key"""
        key_parts = [func_name]
        key_parts.extend(str(arg) for arg in args)
        key_parts.extend(f"{k}:{v}" for k, v in sorted(kwargs.items()))
        return "|".join(key_parts)
    
    @cache_query(ttl=1800)
    def get_user_profile(self, user_id):
        """Cached query example"""
        return self.db.query(
            "SELECT * FROM users WHERE id = ?",
            user_id
        )
    
    def invalidate_cache_pattern(self, pattern):
        """Invalidate cache by pattern (e.g., user:*)"""
        keys = self.redis.keys(pattern)
        if keys:
            self.redis.delete(*keys)
    
    def warm_cache(self):
        """Pre-populate cache with hot data"""
        # Cache top 100 products
        products = self.db.query("""
            SELECT * FROM products
            ORDER BY sales DESC
            LIMIT 100
        """)
        
        for product in products:
            cache_key = f"product:{product['id']}"
            self.redis.setex(
                cache_key,
                3600,
                json.dumps(product)
            )
    
    def get_cache_stats(self):
        """Monitor cache performance"""
        info = self.redis.info('stats')
        return {
            'hits': info.get('keyspace_hits', 0),
            'misses': info.get('keyspace_misses', 0),
            'hit_ratio': info.get('keyspace_hits_ratio', 0)
        }

# Handle CDC invalidation
def invalidate_on_change(event):
    """Invalidate cache when data changes"""
    cache = CockroachRedisCache()
    
    if event['table'] == 'products':
        product_id = event['after']['id'] if event.get('after') else event['before']['id']
        cache.invalidate_cache_pattern(f"product:{product_id}")
```

================================================================================
SECTION 48: ADVANCED AUTOMATION AND ORCHESTRATION
================================================================================

Q224: How do I automate database operations with Kubernetes operators?

CRD (Custom Resource Definition) for CockroachDB:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: cockroachclusters.cockroachdb.com
spec:
  group: cockroachdb.com
  names:
    kind: CockroachCluster
    plural: cockroachclusters
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              nodes:
                type: integer
                description: Number of cluster nodes
              image:
                type: string
                description: CockroachDB docker image
              resources:
                type: object
                description: Resource requests/limits
              storage:
                type: object
                description: Storage configuration
              backup:
                type: object
                description: Backup configuration
              tls:
                type: object
                description: TLS configuration
          status:
            type: object
            properties:
              ready:
                type: boolean
              nodeCount:
                type: integer
              phase:
                type: string
                enum: [Pending, Running, Failed]
```

Operator Logic (Python):
```python
from kopf import on
from kubernetes import client, config
from cockroachdb import connect

class CockroachOperator:
    @on.event('cockroachdb.com', 'v1', 'cockroachclusters')
    def manage_cluster(self, event, **kwargs):
        """Manage CockroachDB cluster lifecycle"""
        cluster = event['object']
        name = cluster['metadata']['name']
        spec = cluster['spec']
        
        if event['type'] == 'ADDED':
            self.create_cluster(name, spec)
        elif event['type'] == 'MODIFIED':
            self.update_cluster(name, spec)
        elif event['type'] == 'DELETED':
            self.delete_cluster(name)
    
    def create_cluster(self, name, spec):
        """Create new CockroachDB cluster"""
        # Create StatefulSet
        v1 = client.AppsV1Api()
        statefulset = {
            'apiVersion': 'apps/v1',
            'kind': 'StatefulSet',
            'metadata': {'name': f'{name}-sts'},
            'spec': {
                'serviceName': f'{name}-service',
                'replicas': spec['nodes'],
                'selector': {'matchLabels': {'app': name}},
                'template': {
                    'metadata': {'labels': {'app': name}},
                    'spec': {
                        'containers': [{
                            'name': 'cockroach',
                            'image': spec['image'],
                            'ports': [{'containerPort': 26257, 'name': 'grpc'},
                                     {'containerPort': 8080, 'name': 'http'}],
                            'command': ['/cockroach/cockroach', 'start-single-node'],
                            'volumeMounts': [
                                {'name': 'datadir', 'mountPath': '/cockroach/cockroach-data'}
                            ]
                        }],
                        'volumes': [
                            {'name': 'datadir', 'emptyDir': {}}
                        ]
                    }
                }
            }
        }
        
        v1.create_namespaced_stateful_set(
            namespace='default',
            body=statefulset
        )
    
    def update_cluster(self, name, spec):
        """Update cluster (scale, upgrade, etc)"""
        v1 = client.AppsV1Api()
        
        # Get current StatefulSet
        sts = v1.read_namespaced_stateful_set(f'{name}-sts', 'default')
        
        # Update replicas
        sts.spec.replicas = spec['nodes']
        
        # Update image version
        sts.spec.template.spec.containers[0].image = spec['image']
        
        # Apply update
        v1.patch_namespaced_stateful_set(
            f'{name}-sts',
            'default',
            sts
        )
```

Q225: How do I implement GitOps for database configuration management?

GitOps Workflow with ArgoCD:
```yaml
# ArgoCD Application for CockroachDB
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cockroachdb
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/db-configs
    targetRevision: main
    path: cockroachdb/
  destination:
    server: https://kubernetes.default.svc
    namespace: cockroachdb
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Configuration Repository Structure:
```
db-configs/
├── cockroachdb/
│   ├── kustomization.yaml
│   ├── cluster.yaml          # CockroachDB cluster definition
│   ├── backup.yaml           # Backup configuration
│   ├── rbac.yaml            # RBAC configuration
│   ├── monitoring/
│   │   ├── prometheus.yaml
│   │   ├── grafana.yaml
│   │   └── alerts.yaml
│   ├── production/
│   │   ├── kustomization.yaml
│   │   └── production-cluster.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── staging-cluster.yaml
│   └── overlays/
│       ├── us-east/
│       ├── eu-central/
│       └── ap-southeast/
└── README.md
```

Kustomization for environment management:
```yaml
# kustomization.yaml for production
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: cockroachdb

bases:
- ../cockroachdb

commonLabels:
  environment: production
  managed-by: gitops

configMapGenerator:
- name: cluster-config
  literals:
  - CLUSTER_NAME=prod-cluster
  - NUM_REPLICAS=5
  - CACHE_SIZE=4GB

replicas:
- name: cockroach
  count: 5

images:
- name: cockroachdb/cockroach
  newTag: v24.1.0

patches:
- target:
    kind: StatefulSet
  patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources
      value:
        requests:
          cpu: "4"
          memory: "8Gi"
        limits:
          cpu: "8"
          memory: "16Gi"
```

================================================================================
SECTION 49: COMPREHENSIVE OPERATIONAL CHECKLISTS
================================================================================

Q226: What is the complete production deployment checklist?

```
COCKROACHDB PRODUCTION DEPLOYMENT CHECKLIST
============================================

PHASE 1: PRE-DEPLOYMENT (Week 1-2)
─────────────────────────────────

INFRASTRUCTURE
  □ Network architecture finalized
  □ VPC/subnets configured
  □ Firewall rules implemented (port 26257, 8080)
  □ Load balancer provisioned and configured
  □ DNS entries created
  □ SSL/TLS certificates generated
  □ Hardware sizing approved (CPU, memory, storage)
  □ Storage backend configured (SSD/NVMe)

SECURITY
  □ TLS certificates created and distributed
  □ SSH keys generated and secured
  □ Initial admin user created
  □ RBAC roles defined and documented
  □ Audit logging enabled
  □ Network segmentation implemented
  □ Encryption at rest configured
  □ Key management plan documented

MONITORING & ALERTING
  □ Prometheus installed and configured
  □ Grafana dashboards created
  □ Alert rules defined and tested
  □ Logging infrastructure ready
  □ PagerDuty/incident management integration
  □ Health check endpoints configured
  □ Performance baseline established

BACKUP & DISASTER RECOVERY
  □ Backup destination configured (S3/GCS)
  □ Backup credentials stored securely
  □ Restore procedure tested
  □ RTO/RPO targets documented
  □ Failover procedures documented
  □ Cross-region backup copies configured
  □ Backup retention policy defined

PHASE 2: DEPLOYMENT (Week 3)
─────────────────────────────

CLUSTER INITIALIZATION
  □ First node started successfully
  □ Cluster initialization completed
  □ Remaining nodes joined
  □ All 3+ nodes showing as LIVE
  □ Replication factor verified
  □ Zone configuration applied
  □ Cluster version updated to latest

DATABASE SETUP
  □ System tables verified
  □ Initial databases created
  □ Schemas deployed
  □ Indexes created and validated
  □ Users and roles created
  □ Permissions verified and tested
  □ Application connection tested

VALIDATION
  □ SQL connectivity test passed
  □ Basic queries execute correctly
  □ Transactions working properly
  □ Replication functioning
  □ Backup/restore tested
  □ Failover procedures tested
  □ Performance benchmarks validated

PHASE 3: PRODUCTION HARDENING (Week 4)
───────────────────────────────────────

SECURITY HARDENING
  □ Password policies enforced
  □ Audit logging verified active
  □ Network access verified restricted
  □ Certificate validation enabled
  □ Certificate expiration monitoring
  □ DDoS protection enabled
  □ Rate limiting configured
  □ Access logs reviewed

PERFORMANCE TUNING
  □ Cache sizes optimized
  □ SQL memory limits set
  □ Connection pooling configured
  □ Slow query logging enabled
  □ Index performance analyzed
  □ Query plans reviewed
  □ Statistics updated

OPERATIONAL SETUP
  □ On-call rotation established
  □ Runbooks documented and available
  □ Incident response procedures tested
  □ Change management process implemented
  □ Deployment procedure documented
  □ Rollback procedures documented
  □ Team trained on operations

MONITORING VALIDATION
  □ All metrics being collected
  □ Alerts functioning correctly
  □ Dashboards displaying data
  □ Log aggregation working
  □ Alerting escalation tested
  □ Health checks passing
  □ Performance within targets

PHASE 4: GO-LIVE (Week 5)
──────────────────────────

FINAL VERIFICATION
  □ All checklists completed
  □ Final security scan passed
  □ Load testing results acceptable
  □ Backup/restore procedures validated
  □ Disaster recovery procedures tested
  □ Team readiness verified
  □ Communication plan ready

MIGRATION PREPARATION
  □ Data migration plan finalized
  □ Migration script tested
  □ Rollback plan documented
  □ Cutover window scheduled
  □ Communication to stakeholders sent
  □ Monitoring escalation procedures ready
  □ War room setup configured

GO-LIVE EXECUTION
  □ Data migration executed
  □ Validation queries passed
  □ Application connectivity verified
  □ Load gradually increased
  □ Monitoring watched closely
  □ No critical alerts triggered
  □ Performance within SLO targets

POST-GO-LIVE (Week 6+)
──────────────────────

STABILIZATION
  □ Monitor cluster 24/7 for first week
  □ Log analysis for errors
  □ Performance validation
  □ No unexpected restarts
  □ Backup/restore procedures validated
  □ Team comfortable with operations
  □ Process improvements identified

DOCUMENTATION
  □ As-built documentation updated
  □ Operational procedures finalized
  □ Troubleshooting guide completed
  □ Knowledge transfer completed
  □ Runbooks updated with learnings
  □ Change log maintained
  □ Architecture diagram updated

OPTIMIZATION
  □ Fine-tune cluster settings
  □ Optimize indexes based on actual workload
  □ Review and adjust alert thresholds
  □ Implement additional automation
  □ Optimize backup strategy
  □ Plan capacity for growth
  □ Schedule next review
```

Q227: What is the complete incident response checklist?

```
COCKROACHDB INCIDENT RESPONSE CHECKLIST
========================================

DETECTION & ALERTING
────────────────────
□ Alert received and verified
□ Issue reproduced and confirmed
□ Severity level assigned (P1/P2/P3/P4)
□ Initial impact assessed
□ Incident channel created (#incident-response)
□ On-call engineer paged
□ War room established (Zoom/Slack)
□ Timeline started (document all times)
────────────────────────────────────────────────
INITIAL INVESTIGATION (0-15 minutes)
──────────────────────────────────────
□ Cluster connectivity verified
  □ Can connect to nodes via SSH
  □ Network connectivity working
  □ Firewall rules not blocking
  
□ Cluster health checked
  □ Node status: SELECT * FROM crdb_internal.nodes
  □ Replication status checked
  □ Range health verified
  □ Disk space available
  □ Memory utilization normal
  □ CPU utilization reasonable
  
□ Application status verified
  □ Application logs reviewed
  □ Error rates checked
  □ Response times measured
  □ Database connections measured
  
□ Recent changes reviewed
  □ Last deployments checked
  □ Configuration changes checked
  □ Traffic patterns analyzed
  □ Recent scaling events noted
────────────────────────────────────────────────
DIAGNOSIS & ROOT CAUSE ANALYSIS (15-60 minutes)
────────────────────────────────────────────────
□ Metrics analyzed
  □ Query latency trends
  □ Throughput changes
  □ Error rate patterns
  □ Resource utilization trends
  
□ Logs reviewed
  □ Cluster logs (last 30 minutes)
  □ Application logs
  □ Audit logs for changes
  □ Error messages identified
  
□ Queries analyzed
  □ Slow query log reviewed
  □ Top 10 queries by latency
  □ Missing indexes identified
  □ Plan changes noted
  
□ System state captured
  □ Node runtime info dumped
  □ Configuration snapshot taken
  □ Range distribution captured
  □ Connection pool status recorded
──────────────────────────────
RESOLUTION (Depends on issue)
──────────────────────────────
For Performance Issues:
  □ Identify slow queries
  □ Add missing indexes
  □ Optimize query plans
  □ Clear connection pools if needed
  □ Verify improvement

For Node Failures:
  □ Drain failed node if still up
  □ Remove from load balancer
  □ Provision replacement
  □ Wait for rebalancing
  □ Verify quorum maintained

For Data Corruption:
  □ Enable read-only mode
  □ Backup corrupted data for analysis
  □ Restore from clean backup
  □ Validate data integrity
  □ Resume operations

For Network Issues:
  □ Verify network connectivity
  □ Check firewall rules
  □ Verify DNS resolution
  □ Test inter-node communication
  □ Restore connectivity
────────────────────────────────────────
VALIDATION & RECOVERY (Post-resolution)
────────────────────────────────────────
□ Issue resolved verified
  □ Metrics returned to normal
  □ Applications running smoothly
  □ No new errors introduced
  □ Performance within SLO
  
□ Data consistency verified
  □ Row counts checked across tables
  □ Checksums validated
  □ Foreign key integrity verified
  □ Application queries working
  
□ Monitoring re-enabled
  □ Alerts re-enabled
  □ Dashboards updating
  □ Logging active
  □ Health checks passing
────────────────────────────────
POST-INCIDENT (Within 24 hours)
────────────────────────────────
□ Incident timeline documented
  □ Start time: ___________
  □ Detection time: ___________
  □ Resolution time: ___________
  □ Total duration: ___________
  □ Impact: ___________
  
□ Root cause analysis completed
  □ Primary cause identified
  □ Contributing factors listed
  □ Timeline of events documented
  
□ Postmortem scheduled (within 48 hours)
  □ Key people invited
  □ Evidence collected
  □ Timeline reviewed
  
□ Action items generated
  □ Immediate fixes applied
  □ Short-term improvements planned
  □ Long-term improvements identified
  □ Owner assigned to each item
  
□ Documentation updated
  □ Runbooks updated
  □ Known issues documented
  □ Monitoring rules refined
  □ Alerting thresholds adjusted
───────────────────────────────────────
PREVENTION (Post-incident follow-up)
──────────────────────────────────────
□ Immediate actions completed (< 1 week)
□ Short-term improvements started (< 1 month)
□ Long-term improvements planned (< 3 months)
□ Automation added to prevent recurrence
□ Team training completed
□ Postmortem action items tracked to completion
```

================================================================================
SECTION 50: COMPLETE OPERATIONAL RUNBOOKS
================================================================================

Q228: What are complete runbooks for common operational procedures?

RUNBOOK 1: Node Failure Recovery
```
OBJECTIVE: Recover from single node failure
SEVERITY: P2
ESTIMATED TIME: 30 minutes

SYMPTOMS:
- Node appears offline in DB Console
- Connection errors to that node
- Query timeouts if queries hit that node
- Replication lag may increase temporarily

PRECONDITIONS:
- Cluster has at least 3 nodes
- Cluster quorum maintained (at least 2 nodes up)

STEP 1: Confirm Node Failure
────────────────────────────
1. Check if node is actually down:
   cockroach sql --certs-dir=certs -c "SELECT * FROM crdb_internal.nodes WHERE node_id = 3"
   
2. Verify SSH access to node:
   ssh node3 "date"
   
3. Check node logs:
   ssh node3 "tail -50 /var/log/cockroach/cockroach.log"

STEP 2: Assess Impact
──────────────────────
1. Check range health:
   cockroach sql --certs-dir=certs -c "
     SELECT COUNT(*) FROM crdb_internal.ranges 
     WHERE ANY_REPLICAS_IN_PROGRESS = true"
   
2. Verify quorum:
   cockroach sql --certs-dir=certs -c "SELECT COUNT(*) as live_nodes FROM crdb_internal.nodes WHERE is_live = true"
   
3. If fewer than 2 nodes alive: ESCALATE TO P1

STEP 3: Node Recovery Attempts
────────────────────────────────
1. Check if process is running:
   ssh node3 "ps aux | grep cockroach"
   
2. If not running, restart:
   ssh node3 "systemctl start cockroachdb"
   sleep 30
   
3. Verify it came back:
   cockroach sql --certs-dir=certs -c "SELECT node_id, is_live FROM crdb_internal.nodes WHERE node_id = 3"
   
4. If back online, monitor for 5 minutes for stability
   
5. If still not responsive, proceed to step 4

STEP 4: Drain and Replace Node
────────────────────────────────
1. Drain the node (graceful shutdown):
   cockroach node drain 3 --certs-dir=certs --host=node1:26257
   
2. Stop cockroach:
   ssh node3 "systemctl stop cockroachdb"
   
3. Check cluster still healthy:
   cockroach sql --certs-dir=certs -c "SELECT COUNT(*) FROM crdb_internal.nodes WHERE is_live = true"
   
4. Monitor rebalancing progress:
   cockroach sql --certs-dir=certs -c "
     SELECT COUNT(*) as pending_ranges 
     FROM crdb_internal.ranges 
     WHERE ANY_REPLICAS_IN_PROGRESS = true"
   
5. When rebalancing complete, replace hardware/OS
   
6. Rejoin node:
   cockroach start --certs-dir=certs --join=node1:26257 --listen-addr=node3:26257

STEP 5: Verification
──────────────────────
1. Verify node is back in cluster:
   cockroach sql --certs-dir=certs -c "SELECT * FROM crdb_internal.nodes WHERE node_id = 3"
   
2. Monitor rebalancing:
   cockroach sql --certs-dir=certs -c "
     SELECT COUNT(*) as remaining_ranges
     FROM crdb_internal.ranges
     WHERE ANY_REPLICAS_IN_PROGRESS = true"
   
3. Check disk usage is reasonable:
   cockroach sql --certs-dir=certs -c "
     SELECT node_id, capacity_used, capacity_total 
     FROM crdb_internal.stores"
   
4. Verify cluster health:
   cockroach sql --certs-dir=certs -c "
     SELECT * FROM crdb_internal.node_runtime_info LIMIT 1"

ROLLBACK:
- If replacement failed, revert to using other nodes
- Ensure cluster still operational with remaining nodes

MONITORING POST-RECOVERY:
- Watch node CPU for 24 hours
- Monitor disk space growth
- Check for any error messages in logs
- Verify backup completion
```

RUNBOOK 2: High Query Latency Response
```
OBJECTIVE: Investigate and resolve high query latency
SEVERITY: P1 if SLA breached, else P2
ESTIMATED TIME: 20-60 minutes

SYMPTOMS:
- P99 latency exceeding SLO target (usually 100-500ms)
- User complaints about slow response times
- Application request timeouts
- Alert triggered: "High latency detected"

STEP 1: Verify the Alert
─────────────────────────
1. Check current metrics:
   cockroach sql --certs-dir=certs -c "
     SELECT 
       percentile_cont(0.50) WITHIN GROUP (ORDER BY latency_ms) as p50,
       percentile_cont(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99
     FROM query_log
     WHERE timestamp > now() - interval '5 minutes'"
   
2. Identify affected service/query:
   cockroach sql --certs-dir=certs -c "
     SELECT query, latency_p99, execution_count
     FROM crdb_internal.node_statement_statistics
     ORDER BY latency_p99 DESC
     LIMIT 10"

STEP 2: Check Cluster Health
──────────────────────────────
1. Node status:
   cockroach sql --certs-dir=certs -c "SELECT * FROM crdb_internal.nodes WHERE is_live = true"
   
2. Disk space:
   cockroach sql --certs-dir=certs -c "
     SELECT node_id, ROUND(100.0 * capacity_used / capacity_total, 2) as used_percent
     FROM crdb_internal.stores"
   
3. Replication lag:
   cockroach sql --certs-dir=certs -c "
     SELECT node_id, COUNT(*) as ranges
     FROM crdb_internal.ranges_no_leases
     GROUP BY node_id"

STEP 3: Analyze Slow Queries
──────────────────────────────
1. Get slowest query:
   cockroach sql --certs-dir=certs -c "
     SELECT query, latency_p99, execution_count, rows_read
     FROM crdb_internal.node_statement_statistics
     WHERE latency_p99 > 1000
     ORDER BY latency_p99 DESC
     LIMIT 1"
   
2. Analyze query plan:
   cockroach sql --certs-dir=certs -c "
     EXPLAIN ANALYZE [the slow query]"
   
3. Check for sequential scans on large tables:
   Look for "scan" in plan instead of index usage

STEP 4: Apply Fixes (In order of priority)
────────────────────────────────────────────

IF Sequential Scan on Large Table:
  1. Identify missing index from query plan
  2. Create index:
     CREATE INDEX idx_name ON table(column);
  3. Re-run query and verify improvement
  
IF High CPU Usage:
  1. Check running queries:
     SELECT * FROM crdb_internal.node_sessions LIMIT 20
  2. Kill long-running queries:
     CANCEL SESSION 'session_id'
  3. Optimize query or configuration

IF High Disk I/O:
  1. Check for large compactions
  2. Monitor with:
     cockroach sql --certs-dir=certs -c "SELECT * FROM crdb_internal.stores"
  3. If disk full, archive old data
  
IF Connection Pool Exhaustion:
  1. Check connections:
     SELECT COUNT(*) FROM crdb_internal.node_sessions
  2. Kill idle connections if needed
  3. Increase connection limits

STEP 5: Verify Improvement
────────────────────────────
1. Re-measure latency:
   cockroach sql --certs-dir=certs -c "
     SELECT percentile_cont(0.99) WITHIN GROUP (ORDER BY latency_ms)
     FROM query_log
     WHERE timestamp > now() - interval '5 minutes'"
   
2. Verify SLO met
3. Check for new errors

STEP 6: Prevent Recurrence
────────────────────────────
1. Document root cause
2. Add monitoring/alerting if missing
3. Create automated fix if possible
4. Update runbooks
5. Schedule capacity planning review if needed

ESCALATION:
- If latency still high after 15 minutes: escalate to senior DBA
- If causing data loss: escalate to manager
- If entire cluster affected: declare SEV-1 incident
```

================================================================================
SECTION 51: FINAL SUMMARY AND CLOSING RECOMMENDATIONS
================================================================================

Q229: What are the most critical takeaways for CockroachDB operations?

TOP 10 CRITICAL OPERATIONAL PRINCIPLES:

1. REDUNDANCY AT EVERY LEVEL
   - 3+ node cluster minimum (quorum protection)
   - Multi-region deployment for resilience
   - Multiple backup copies in different locations
   - Automated failover procedures
   
2. MONITORING MUST BE COMPREHENSIVE
   - Monitor every critical metric
   - Set SLO-aligned alerts
   - Log everything important
   - Establish alerting escalation
   
3. AUTOMATE EVERYTHING POSSIBLE
   - Infrastructure-as-code for reproducibility
   - Automated deployment pipelines
   - Automated incident response
   - Automated scaling based on load
   
4. TEST FAILURE SCENARIOS REGULARLY
   - Monthly disaster recovery drills
   - Quarterly chaos engineering tests
   - Regular failover testing
   - Backup restoration practice
   
5. SECURITY IS NOT OPTIONAL
   - TLS for all connections
   - RBAC implementation mandatory
   - Audit logging always active
   - Encryption at rest for sensitive data
   
6. CAPACITY PLANNING IS CONTINUOUS
   - Always maintain 1.5x headroom
   - Monitor growth trends
   - Plan scaling in advance
   - Right-size instances regularly
   
7. DOCUMENTATION IS OPERATIONAL
   - Runbooks for every procedure
   - Playbooks for common incidents
   - Architecture documented
   - Decision rationale recorded
   
8. DATA INTEGRITY IS PARAMOUNT
   - Regular consistency checks
   - Backup verification procedures
   - Data validation on restore
   - Audit trails for accountability
   
9. TEAM EXPERTISE IS CRITICAL
   - Invest in training continuously
   - Cross-train for redundancy
   - Document institutional knowledge
   - Share learnings from incidents
   
10. MEASURE AND OPTIMIZE CONTINUOUSLY
    - Track operational KPIs
    - Review quarterly
    - Automate improvements
    - Celebrate successes

Q230: What should I do right after deploying CockroachDB to production?

FIRST 90 DAYS POST-PRODUCTION CHECKLIST:

WEEK 1: STABILIZATION
  □ Monitor cluster 24/7
  □ No unexpected behavior
  □ All alerts functioning
  □ Performance within SLO
  □ Team comfortable with tools
  □ Any issues fixed immediately

WEEK 2-4: VALIDATION
  □ Complete first month of operation
  □ Run full backup and restore cycle
  □ Conduct disaster recovery drill
  □ Test failover procedures
  □ Validate performance under load
  □ Optimize any hot paths identified

MONTH 2: OPTIMIZATION
  □ Fine-tune cluster settings
  □ Optimize queries based on real workload
  □ Implement automation opportunities
  □ Improve monitoring coverage
  □ Document lessons learned
  □ Plan capacity for growth

MONTH 3: HARDENING
  □ Conduct security audit
  □ Test penetration scenarios
  □ Review access controls
  □ Validate compliance posture
  □ Perform load testing
  □ Document SLAs

ONGOING (QUARTERLY):
  □ Capacity review and planning
  □ Performance optimization sweep
  □ Security and compliance audit
  □ Disaster recovery drill
  □ Team skills assessment
  □ Vendor updates and patches

COCKROACHDB ADMINISTRATION GUIDE - FINAL SUPPLEMENTARY REFERENCE
Advanced Troubleshooting, Performance Tuning Specifics, and Operational Matrices

================================================================================
SECTION 52: ADVANCED TROUBLESHOOTING DEEP-DIVES
================================================================================

Q231: How do I debug replication lag issues in detail?

Comprehensive Replication Lag Diagnosis:

```sql
-- Step 1: Measure current replication lag
SELECT 
  node_id,
  COUNT(*) as replica_count,
  AVG(replicas_in_progress::numeric) as avg_replicas_in_progress,
  MAX(replicas_in_progress) as max_replicas_in_progress,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY replicas_in_progress) as p99_replicas_in_progress
FROM crdb_internal.ranges_no_leases
GROUP BY node_id
ORDER BY max_replicas_in_progress DESC;

-- Step 2: Identify which ranges have replication lag
SELECT 
  range_id,
  table_name,
  start_key,
  end_key,
  replicas,
  lease_holder,
  replicas_in_progress,
  unavailable_replicas,
  COUNT(*) FILTER (WHERE replicas_in_progress > 0) as replicas_catching_up
FROM crdb_internal.ranges
WHERE replicas_in_progress > 0
ORDER BY replicas_in_progress DESC
LIMIT 20;

-- Step 3: Check network latency between nodes
-- (Query from monitoring system or explicit ping)
SELECT 
  source_node_id,
  dest_node_id,
  AVG(latency_ms) as avg_latency,
  P99(latency_ms) as p99_latency,
  MAX(latency_ms) as max_latency
FROM crdb_internal.node_metrics
GROUP BY source_node_id, dest_node_id
HAVING P99(latency_ms) > 10;  -- Alert if >10ms

-- Step 4: Check disk I/O on follower nodes
SELECT 
  node_id,
  write_count,
  write_bytes,
  fsync_count,
  fsync_latency_ms
FROM crdb_internal.store_metrics
ORDER BY fsync_latency_ms DESC;

-- Step 5: Analyze replication traffic pattern
SELECT 
  source_region,
  dest_region,
  SUM(bytes_sent) as total_bytes,
  AVG(bytes_sent) as avg_bytes_per_message,
  COUNT(*) as message_count,
  AVG(latency_ms) as avg_latency
FROM replication_traffic_log
WHERE timestamp > now() - interval '10 minutes'
GROUP BY source_region, dest_region
ORDER BY total_bytes DESC;

-- Step 6: Check for hot ranges causing replication pressure
SELECT 
  range_id,
  table_name,
  writes_per_second,
  bytes_written_per_second,
  leader_node_id
FROM range_metrics
WHERE writes_per_second > 1000
ORDER BY writes_per_second DESC
LIMIT 10;

-- Step 7: Monitor raft consensus metrics
SELECT 
  node_id,
  raft_ticks,
  raft_heartbeats,
  raft_heartbeats_failed,
  ROUND(100.0 * raft_heartbeats_failed / NULLIF(raft_heartbeats, 0), 2) as heartbeat_failure_rate
FROM crdb_internal.node_metrics
WHERE raft_heartbeats_failed > 0
ORDER BY heartbeat_failure_rate DESC;
```

Resolution Strategies:

```bash
# Strategy 1: Increase replication bandwidth (if network limited)
# Modify cockroach start parameter:
# --max-offset=1s (default, controls raft tick interval)

# Strategy 2: Reduce hot range pressure
# - Add index to reduce sequential scans
# - Distribute writes across multiple ranges
# - Implement application-level batching

# Strategy 3: Improve network between regions
# - Upgrade network capacity
# - Use direct peering instead of internet
# - Implement traffic optimization

# Strategy 4: Scale follower nodes
# - Add more nodes to reduce per-node load
# - Implement follower read routing
# - Balance ranges across all nodes

# Monitor improvement over time
watch -n 5 'cockroach sql --certs-dir=certs -c "
  SELECT 
    node_id,
    COUNT(*) as replicas_in_progress
  FROM crdb_internal.ranges
  WHERE replicas_in_progress > 0
  GROUP BY node_id"'
```

Q232: How do I diagnose and fix out-of-memory (OOM) errors?

Memory Leak Investigation:

```sql
-- Step 1: Monitor memory trend
SELECT 
  timestamp,
  node_id,
  heap_inuse_bytes / (1024*1024*1024) as heap_gb,
  alloc_bytes / (1024*1024*1024) as alloc_gb,
  sys_bytes / (1024*1024*1024) as sys_gb,
  LAG(heap_inuse_bytes) OVER (PARTITION BY node_id ORDER BY timestamp) as prev_heap,
  (heap_inuse_bytes - LAG(heap_inuse_bytes) OVER (PARTITION BY node_id ORDER BY timestamp)) / (1024*1024*1024) as heap_growth_gb
FROM memory_metrics
WHERE timestamp > now() - interval '1 hour'
ORDER BY timestamp DESC
LIMIT 100;

-- Step 2: Identify memory growth trend
SELECT 
  node_id,
  DATE_TRUNC('hour', timestamp)::timestamp as hour,
  AVG(heap_inuse_bytes) / (1024*1024*1024) as avg_heap_gb,
  MAX(heap_inuse_bytes) / (1024*1024*1024) as max_heap_gb
FROM memory_metrics
WHERE timestamp > now() - interval '24 hours'
GROUP BY node_id, DATE_TRUNC('hour', timestamp)
ORDER BY hour DESC;

-- Step 3: Find goroutine leak
SELECT 
  timestamp,
  node_id,
  goroutine_count,
  LAG(goroutine_count) OVER (PARTITION BY node_id ORDER BY timestamp) as prev_count,
  goroutine_count - LAG(goroutine_count) OVER (PARTITION BY node_id ORDER BY timestamp) as growth
FROM goroutine_metrics
WHERE goroutine_count > 50000
ORDER BY timestamp DESC;

-- Step 4: Identify long-running connections
SELECT 
  session_id,
  user_name,
  client_addr,
  created_at,
  EXTRACT(EPOCH FROM (now() - created_at)) as connection_age_seconds,
  transaction_status
FROM crdb_internal.node_sessions
WHERE EXTRACT(EPOCH FROM (now() - created_at)) > 3600
ORDER BY connection_age_seconds DESC;

-- Step 5: Check for unbounded query results
SELECT 
  query_id,
  user_name,
  query,
  EXTRACT(EPOCH FROM (now() - started_at)) as running_seconds,
  rows_returned
FROM crdb_internal.node_statements
WHERE rows_returned > 1000000
ORDER BY rows_returned DESC;
```

Resolution Procedures:

```bash
#!/bin/bash
# Memory issues resolution

echo "=== Memory Issue Resolution ==="

# 1. Identify memory limit setting
cockroach sql --certs-dir=certs -c "
  SHOW CLUSTER SETTING memory.max_allowed_override_bytes;
  SHOW CLUSTER SETTING sql.distsql.max_memory_per_node;
"

# 2. Kill long-running queries if needed
cockroach sql --certs-dir=certs -c "
  SELECT session_id FROM crdb_internal.node_sessions
  WHERE EXTRACT(EPOCH FROM (now() - created_at)) > 3600" | while read session; do
    echo "Killing session $session"
    cockroach sql --certs-dir=certs -c "CANCEL SESSION '$session'"
  done

# 3. Close idle connections
# Application should implement connection pooling and timeouts

# 4. Increase node memory if available
# Edit cockroach service configuration:
# --max-sql-memory=4GB (increase from current)

# 5. Optimize queries causing high memory usage
# Check EXPLAIN ANALYZE output for:
# - Large hash joins
# - Unbounded result sets
# - Multiple sorts

# 6. Monitor after changes
echo "Monitoring memory after changes..."
while true; do
  cockroach sql --certs-dir=certs -c "
    SELECT 
      node_id,
      heap_inuse_bytes / (1024*1024*1024) as heap_gb
    FROM crdb_internal.node_runtime_info"
  sleep 30
done
```

================================================================================
SECTION 53: PERFORMANCE TUNING MATRICES
================================================================================

Q233: What is the complete performance tuning decision matrix?

Comprehensive Performance Tuning Guide:

```
PERFORMANCE ISSUE DIAGNOSIS & RESOLUTION MATRIX
================================================

SYMPTOM: High Query Latency (P99 > 100ms)
─────────────────────────────────────────

Check CPU first:
┌─ CPU < 20%                           ┌─ CPU 20-60%                     ┌─ CPU > 60%
│                                       │                                 │
├─ Check Disk I/O                       ├─ Likely query optimization     ├─ Reduce query load
│                                       │ issue                          │ OR
├─ If disk high:                        │                                ├─ Scale horizontally
│  └─ Add indexes                       ├─ Solutions:                     │
│  └─ Denormalize                       │  • Add indexes                   ├─ Temporary: Kill
│  └─ Archive old data                  │  • Optimize query plans         │ expensive queries
│                                       │  • Add missing columns to index │
├─ If network high:                     │  • Denormalize tables           ├─ Monitor peak times
│  └─ Optimize join order               │  • Enable follower reads         │ for scaling
│  └─ Reduce cross-region traffic       │                                ├─ Implement query
│                                       │                                 │ batching
└─ If none high:
   └─ Contention issue
      • Identify hot keys
      • Distribute writes
      • Use partitioning

SYMPTOM: Memory Pressure
─────────────────────────

┌─ Transient spikes                    ┌─ Steady growth (leak)           ┌─ Peak memory usage
│                                       │                                 │
├─ Likely large query                  ├─ Kill long-running queries      ├─ Increase node memory
│                                       │                                 │ OR
├─ Solutions:                           ├─ Close idle connections         ├─ Reduce cluster size
│  • Increase statement timeout         │                                 │ (concentrate load)
│  • Implement query limits             ├─ Restart nodes to clear         │
│  • Add LIMIT clauses                  │                                 ├─ Optimize queries
│                                       ├─ Update query plans             │ for memory
└─ Monitor and adjust max-sql-memory    │                                 │
                                        └─ Monitor goroutine count       └─ Implement app-level
                                           (potential goroutine leak)        caching

SYMPTOM: Disk Space Exhaustion
───────────────────────────────

┌─ Rapid growth                        ┌─ Steady growth                  ┌─ Critical (>90%)
│                                       │                                 │
├─ Identify fast-growing table          ├─ Enable compaction              ├─ EMERGENCY:
│                                       │                                 │ • Archive old data
├─ Check for:                           ├─ Implement TTL                  │ • Enable PITR pruning
│  • WAL accumulation                   │                                 │ • Add disk space
│  • Snapshot sizes                     ├─ Create partition strategy      │
│  • Range splitting                    │                                 ├─ Resume operations
│                                       ├─ Adjust GC settings             │ cautiously
├─ Solutions:                           │                                 │
│  • Trigger manual compaction          └─ Archive and delete old data   ├─ Permanent fix:
│  • Delete unnecessary data            │                                 │ • Upgrade storage
│  • Archive to cheaper storage         └─ Monitor growth rate           │ • Implement archival
│                                                                         │ • Reduce data retention
└─ Implement continuous monitoring

SYMPTOM: Replication Lag
─────────────────────────

┌─ Network latency issue                ┌─ Disk I/O bottleneck           ┌─ CPU constraint
│                                       │                                 │
├─ Check inter-node latency             ├─ Check follower disk I/O       ├─ High CPU on
│                                       │                                 │ followers
├─ Solutions:                           ├─ Solutions:                     │
│  • Upgrade network                    │  • Upgrade storage speed        ├─ Solutions:
│  • Use direct peering                 │  • Reduce write load            │  • Add more followers
│  • Reduce cross-region writes         │  • Implement caching            │  • Optimize workload
│                                       │  • Batch inserts                │  • Enable follower
└─ Implement QoS/traffic shaping                                          │    reads
                                        └─ Monitor disk performance
                                                                         └─ Profile CPU usage

SYMPTOM: Connection Pool Exhaustion
────────────────────────────────────

┌─ Connections slowly accumulating     ┌─ Connections spike then drop   ┌─ Permanent
│                                       │                                 │ exhaustion
├─ Likely connection leak               ├─ Normal high-load pattern       │
│                                       │                                 ├─ Increase pool size
├─ Solutions:                           ├─ Solutions:                     │ OR
│  • Fix app connection handling        │  • Increase idle timeout        ├─ Reduce connections
│  • Restart app                        │  • Implement circuit breaker    │ per app
│  • Increase timeout                   │                                 │
│                                       └─ Monitor peak usage            ├─ Profile app
└─ Monitor connection age                                                 │ for leaks
                                                                          │
                                                                         └─ Implement
                                                                            connection pooling
```

Q234: What are specific performance tuning configurations for different workload types?

```sql
-- OLTP Workload Configuration
-- (Transactional, low latency, moderate throughput)
SET CLUSTER SETTING kv.allocator.mode='balanced';
SET CLUSTER SETTING sql.defaults.max_retries=3;
SET CLUSTER SETTING sql.defaults.statement_timeout='30s';
SET CLUSTER SETTING kv.range_merge.queue_enabled=true;
SET CLUSTER SETTING sql.distsql.max_memory_per_node='2GB';

-- Create OLTP-optimized indexes
CREATE INDEX idx_oltp_lookup ON orders(customer_id, status) INCLUDE (amount);

-- Use follower reads where strong consistency not required
SET SESSION enable_follower_reads=true;


-- OLAP Workload Configuration
-- (Analytical, high latency tolerance, high throughput)
SET CLUSTER SETTING sql.defaults.statement_timeout='5m';
SET CLUSTER SETTING sql.distsql.max_memory_per_node='8GB';
SET CLUSTER SETTING sql.defaults.prefer_lookup_joins_enabled=false;

-- Create OLAP-optimized indexes
CREATE INDEX idx_olap_analytics ON transactions(date, category) INCLUDE (amount, customer_id);

-- Use materialized views for aggregations
CREATE MATERIALIZED VIEW daily_sales_summary AS
  SELECT date, SUM(amount) as revenue, COUNT(*) as orders
  FROM transactions
  GROUP BY date;


-- TIME-SERIES Workload Configuration
-- (High write rate, time-based queries, archival needs)
SET CLUSTER SETTING kv.allocator.mode='aggressive';
SET CLUSTER SETTING sql.defaults.max_retries=5;
SET CLUSTER SETTING sql.distsql.max_memory_per_node='4GB';

-- Partition by time
CREATE TABLE metrics_partitioned (
  timestamp TIMESTAMP NOT NULL,
  metric_name VARCHAR,
  value DECIMAL,
  PRIMARY KEY (timestamp DESC, metric_name)
) PARTITION BY RANGE (timestamp) (...);

-- Implement TTL cleanup
CREATE SCHEDULE cleanup_metrics FOR
  DELETE FROM metrics WHERE timestamp < now() - interval '90 days'
  RECURRING EVERY 1 day;


-- HIGH-CONCURRENCY Workload Configuration
-- (Many concurrent queries, connection pooling)
SET CLUSTER SETTING sql.defaults.statement_timeout='10s';
SET CLUSTER SETTING sql.distsql.max_memory_per_node='1GB';
ALTER ROLE app_role CONNECTION LIMIT 500;

-- Use prepared statements
PREPARE stmt AS SELECT * FROM users WHERE id = $1;
EXECUTE stmt(123);
```

================================================================================
SECTION 54: COMPLIANCE AND REGULATORY FRAMEWORKS
================================================================================

Q235: What are complete compliance templates for major regulations?

GDPR Compliance Template:

```sql
-- GDPR Compliance Configuration
-- 1. Data Processing Records
CREATE TABLE data_processing_records (
  record_id UUID PRIMARY KEY,
  data_category VARCHAR,  -- Personal data, special category, etc.
  processing_purpose VARCHAR,
  legal_basis VARCHAR,  -- Consent, contract, legal obligation, vital interest, public task, legitimate interest
  controller VARCHAR,
  processor VARCHAR,
  data_subject_categories VARCHAR[],
  retention_period VARCHAR,
  security_measures VARCHAR[],
  countries_transferred VARCHAR[],
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 2. Consent Management
CREATE TABLE consent_records (
  consent_id UUID PRIMARY KEY,
  data_subject_id UUID,
  consent_type VARCHAR,  -- Marketing, analytics, third-party sharing
  given_at TIMESTAMP,
  withdrawn_at TIMESTAMP,
  consent_version VARCHAR,
  created_by VARCHAR
);

-- 3. Data Subject Rights Management
CREATE TABLE data_subject_requests (
  request_id UUID PRIMARY KEY,
  data_subject_id UUID,
  request_type VARCHAR,  -- Access, rectification, erasure, restriction, portability, objection
  requested_at TIMESTAMP,
  deadline_date DATE,
  status VARCHAR,  -- PENDING, IN_PROGRESS, COMPLETED, DENIED
  response_date TIMESTAMP,
  handler VARCHAR
);

-- 4. Data Breach Notification
CREATE TABLE breach_reports (
  breach_id UUID PRIMARY KEY,
  discovery_date DATE,
  notification_date DATE,
  data_subjects_affected BIGINT,
  breach_description VARCHAR,
  mitigation_measures VARCHAR[],
  authority_notified BOOLEAN,
  subjects_notified BOOLEAN
);

-- 5. Audit Trail for GDPR
CREATE TABLE gdpr_audit_log (
  log_id UUID PRIMARY KEY,
  timestamp TIMESTAMP DEFAULT NOW(),
  user_name VARCHAR,
  action VARCHAR,  -- Access, modify, delete
  data_category VARCHAR,
  records_affected BIGINT,
  justification VARCHAR
);

-- 6. Automatic Data Retention
CREATE SCHEDULE delete_expired_data FOR
  DELETE FROM personal_data
  WHERE retention_expiry_date < now()
  RECURRING EVERY 1 day;

-- 7. PITR for Erasure Verification
-- Restore from PITR to verify right-to-be-forgotten
SELECT COUNT(*) as remaining_records
FROM archived_personal_data AS OF SYSTEM TIME '2024-01-15 12:00:00'
WHERE person_id = $1;
```

HIPAA Compliance Template:

```sql
-- HIPAA Compliance Configuration
-- 1. Protected Health Information (PHI) Tracking
CREATE TABLE phi_access_log (
  access_id UUID PRIMARY KEY,
  timestamp TIMESTAMP DEFAULT NOW(),
  user_id UUID,
  user_name VARCHAR,
  action VARCHAR,  -- READ, MODIFY, DELETE
  phi_type VARCHAR,  -- Medical record, diagnosis, prescription, genetic info
  patient_id UUID,
  record_id UUID,
  ip_address INET,
  session_id VARCHAR,
  justification VARCHAR
);

-- 2. Encryption at Rest
-- All PHI tables encrypted with KMS

-- 3. Access Control
CREATE TABLE phi_access_control (
  user_id UUID,
  access_level VARCHAR,  -- View, modify, delete
  data_types VARCHAR[],  -- Specific PHI types allowed
  organizations VARCHAR[],  -- Which organizations can access
  start_date DATE,
  end_date DATE,
  approval_by VARCHAR
);

-- 4. Audit Controls
CREATE TABLE audit_trail (
  audit_id UUID PRIMARY KEY,
  event_date TIMESTAMP,
  event_type VARCHAR,
  user_id UUID,
  resource_id UUID,
  action VARCHAR,
  status VARCHAR,
  outcome VARCHAR
);

-- 5. Backup and Recovery
CREATE SCHEDULE secure_backup FOR
  BACKUP INTO 's3://hipaa-compliant-bucket/backup'
  WITH encryption_key='kms://aws/key-id'
  RECURRING EVERY 6 hours
  WITH RETENTION 7 years;

-- 6. Breach Notification
CREATE TABLE breach_notification (
  notification_id UUID PRIMARY KEY,
  breach_date DATE,
  discovery_date DATE,
  individuals_affected BIGINT,
  notification_date DATE,
  hhs_notification_date DATE,
  media_notification BOOLEAN
);

-- 7. Business Associate Agreements Tracking
CREATE TABLE baa_tracking (
  baa_id UUID PRIMARY KEY,
  associate_name VARCHAR,
  services_provided VARCHAR,
  signed_date DATE,
  expiration_date DATE,
  last_audit_date DATE,
  audit_results VARCHAR
);
```

PCI DSS Compliance Template:

```sql
-- PCI DSS Compliance Configuration
-- 1. Cardholder Data Storage
-- NEVER store full PAN (Primary Account Number)
-- Use tokenization instead

CREATE TABLE tokenized_payment_methods (
  token_id UUID PRIMARY KEY,
  last_four_digits VARCHAR(4),  -- Only last 4 digits
  expiry_month INT,
  expiry_year INT,
  card_type VARCHAR,
  customer_id UUID,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Access Control Matrix
CREATE TABLE pci_access_control (
  user_id UUID,
  resource_type VARCHAR,  -- Payment data, logs, keys
  access_level VARCHAR,  -- None, view, modify
  approval_date DATE,
  approval_by VARCHAR
);

-- 3. Authentication and Logging
CREATE TABLE payment_access_log (
  log_id UUID PRIMARY KEY,
  timestamp TIMESTAMP DEFAULT NOW(),
  user_id UUID,
  action VARCHAR,  -- Access, modify, delete payment data
  resource_id UUID,
  ip_address INET,
  session_id VARCHAR,
  result VARCHAR  -- Success, failure
);

-- 4. Encryption Key Management
CREATE TABLE encryption_key_management (
  key_id UUID PRIMARY KEY,
  algorithm VARCHAR,
  key_length INT,
  generation_date DATE,
  rotation_date DATE,
  expiration_date DATE,
  key_holder VARCHAR,
  hsm_location VARCHAR
);

-- 5. Vulnerability Management
CREATE TABLE vulnerability_scan_results (
  scan_id UUID PRIMARY KEY,
  scan_date DATE,
  vulnerability_id VARCHAR,
  severity VARCHAR,  -- CRITICAL, HIGH, MEDIUM, LOW
  asset_type VARCHAR,
  remediation_status VARCHAR,
  remediation_date DATE
);

-- 6. Audit and Accountability
CREATE TABLE pci_audit_log (
  audit_id UUID PRIMARY KEY,
  audit_date DATE,
  auditor_name VARCHAR,
  scope VARCHAR,
  findings VARCHAR[],
  remediation_actions VARCHAR[],
  remediation_deadlines DATE[]
);
```

================================================================================
SECTION 55: OPERATIONAL EFFICIENCY MATRICES
================================================================================

Q236: What are complete operational efficiency metrics and tracking?

Operational Efficiency Scorecard:

```sql
CREATE TABLE operational_efficiency_metrics (
  metric_date DATE,
  
  -- Availability Metrics
  cluster_availability_percent DECIMAL,
  planned_downtime_hours DECIMAL,
  unplanned_downtime_hours DECIMAL,
  
  -- Performance Metrics
  query_p50_latency_ms DECIMAL,
  query_p99_latency_ms DECIMAL,
  throughput_qps DECIMAL,
  
  -- Reliability Metrics
  mttr_minutes INT,  -- Mean Time To Recovery
  mtbf_days INT,  -- Mean Time Between Failures
  incident_count INT,
  
  -- Backup & Recovery
  backup_completion_time_min INT,
  backup_success_rate DECIMAL,
  restore_test_success_rate DECIMAL,
  recovery_point_objective_minutes INT,
  
  -- Resource Utilization
  avg_cpu_percent DECIMAL,
  avg_memory_percent DECIMAL,
  avg_disk_percent DECIMAL,
  
  -- Cost Metrics
  cost_per_qps DECIMAL,
  storage_cost_per_gb DECIMAL,
  total_monthly_cost DECIMAL,
  
  -- Team Metrics
  deployments_completed INT,
  deployment_success_rate DECIMAL,
  on_call_hours_per_engineer INT,
  training_hours_per_person INT,
  
  -- Compliance
  audit_findings INT,
  security_vulnerabilities_critical INT,
  security_vulnerabilities_high INT,
  compliance_violations INT
);

-- Monthly Operational Dashboard Query
SELECT 
  metric_date,
  CASE WHEN cluster_availability_percent >= 99.9 THEN '✓ PASS' ELSE '✗ MISS' END as availability_status,
  CASE WHEN query_p99_latency_ms <= 100 THEN '✓ PASS' ELSE '✗ MISS' END as latency_status,
  CASE WHEN mttr_minutes <= 15 THEN '✓ PASS' ELSE '✗ MISS' END as recovery_status,
  CASE WHEN backup_success_rate >= 99.5 THEN '✓ PASS' ELSE '✗ MISS' END as backup_status,
  CASE WHEN deployment_success_rate >= 95 THEN '✓ PASS' ELSE '✗ MISS' END as deployment_status,
  CASE WHEN compliance_violations = 0 THEN '✓ PASS' ELSE '✗ MISS' END as compliance_status,
  
  -- Summary score
  ROUND(
    100.0 * (
      (CASE WHEN cluster_availability_percent >= 99.9 THEN 1 ELSE 0 END) +
      (CASE WHEN query_p99_latency_ms <= 100 THEN 1 ELSE 0 END) +
      (CASE WHEN mttr_minutes <= 15 THEN 1 ELSE 0 END) +
      (CASE WHEN backup_success_rate >= 99.5 THEN 1 ELSE 0 END) +
      (CASE WHEN deployment_success_rate >= 95 THEN 1 ELSE 0 END) +
      (CASE WHEN compliance_violations = 0 THEN 1 ELSE 0 END)
    ) / 6.0,
    1
  ) as operational_score_percent
FROM operational_efficiency_metrics
WHERE metric_date >= CURRENT_DATE - 30
ORDER BY metric_date DESC;
```

Q237: What is the complete team productivity and training matrix?

```sql
CREATE TABLE team_development_tracking (
  person_id UUID,
  person_name VARCHAR,
  
  -- Role Information
  role VARCHAR,  -- DBA, SRE, DevOps, Architect, Developer
  level VARCHAR,  -- Junior, Mid, Senior, Lead
  start_date DATE,
  
  -- Skills Matrix
  cockroachdb_proficiency VARCHAR,  -- Novice, Intermediate, Advanced, Expert
  kubernetes_proficiency VARCHAR,
  sql_proficiency VARCHAR,
  linux_proficiency VARCHAR,
  python_proficiency VARCHAR,
  
  -- Certifications
  cockroachdb_certified BOOLEAN,
  kubernetes_certified BOOLEAN,
  aws_certified BOOLEAN,
  gcp_certified BOOLEAN,
  
  -- Training Hours
  training_hours_ytd INT,
  target_training_hours INT,
  
  -- On-Call Performance
  on_call_shifts_completed INT,
  average_resolution_time_minutes INT,
  incident_response_rating DECIMAL,
  
  -- Knowledge Transfer
  runbooks_written INT,
  training_sessions_conducted INT,
  team_mentored_count INT
);

-- Team Readiness Assessment
SELECT 
  role,
  COUNT(*) as team_size,
  COUNT(*) FILTER (WHERE cockroachdb_certified) as certified,
  AVG(training_hours_ytd)::INT as avg_training_hours,
  COUNT(*) FILTER (WHERE average_resolution_time_minutes < 15) as effective_responders,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE average_resolution_time_minutes < 15) / COUNT(*),
    1
  ) as response_effectiveness_percent
FROM team_development_tracking
GROUP BY role
ORDER BY team_size DESC;
```

Q238: What are final strategic recommendations for enterprise CockroachDB operations?

STRATEGIC OPERATIONAL FRAMEWORK:

```
YEAR 1: FOUNDATION & STABILIZATION
├─ Q1: Deploy production cluster, establish monitoring
├─ Q2: Optimize for workload, implement automation
├─ Q3: Scale to production load, optimize costs
└─ Q4: Establish operational excellence, plan growth

YEAR 2: OPTIMIZATION & SCALING
├─ Q1: Multi-region expansion, global resilience
├─ Q2: Advanced automation, reduce manual work
├─ Q3: Performance optimization, cost reduction
└─ Q4: Team development, knowledge consolidation

YEAR 3+: EXCELLENCE & INNOVATION
├─ Continuous improvement culture
├─ Advanced features utilization
├─ Cost optimization at scale
└─ Thought leadership in operations
```

Q239: What metrics matter most for enterprise success?

```
TOP 5 ENTERPRISE SUCCESS METRICS:

1. RELIABILITY (Availability)
   - Target: 99.95%+ uptime
   - Measure: (Total Time - Downtime) / Total Time
   - Owner: Operations/SRE
   
2. PERFORMANCE (User Experience)
   - Target: P99 < 100ms
   - Measure: Query latency percentiles
   - Owner: Database Team
   
3. COST EFFICIENCY
   - Target: Minimize per-QPS cost
   - Measure: Total monthly cost / QPS capacity
   - Owner: Infrastructure/Finance
   
4. TEAM PRODUCTIVITY
   - Target: 80%+ automated operations
   - Measure: Manual vs automated tasks
   - Owner: Operations Management
   
5. RISK MANAGEMENT
   - Target: Zero preventable outages
   - Measure: Incidents prevented vs total incidents
   - Owner: Enterprise Security/Risk
```

Q240: Final 10 priorities for enterprise CockroachDB success

1. AUTOMATE EVERYTHING POSSIBLE
   - Deployment pipelines
   - Scaling decisions
   - Incident response
   - Backup/restore
   
2. INVEST IN TEAM EXPERTISE
   - Training programs
   - Certifications
   - Knowledge sharing
   - Career development
   
3. IMPLEMENT COMPREHENSIVE MONITORING
   - Business metrics
   - Technical metrics
   - Operational metrics
   - Security metrics
   
4. ESTABLISH ROCK-SOLID BACKUP/DR
   - Automated backups
   - Regular testing
   - Clear procedures
   - Defined RTO/RPO
   
5. MAINTAIN SECURITY POSTURE
   - Encryption everywhere
   - Access controls
   - Audit logging
   - Compliance adherence
   
6. OPTIMIZE CONTINUOUSLY
   - Performance tuning
   - Cost optimization
   - Capacity planning
   - Process improvement
   
7. DOCUMENT EVERYTHING
   - Architecture
   - Procedures
   - Decisions
   - Learnings
   
8. TEST FAILURE SCENARIOS
   - Disaster recovery drills
   - Chaos engineering
   - Failover testing
   - Load testing
   
9. COMMUNICATE TRANSPARENTLY
   - Status updates
   - Incident reports
   - Performance reports
   - Strategic plans
   
10. CELEBRATE SUCCESSES
    - Milestones reached
    - Zero-incident periods
    - Cost savings achieved
    - New capabilities deployed

COCKROACHDB ADMINISTRATION GUIDE - FINAL 10 QUESTIONS (Q241-Q250)
Strategic Recommendations and Closing Guidance for Complete Mastery

================================================================================
SECTION 56: FINAL STRATEGIC QUESTIONS AND GUIDANCE
================================================================================

Q241: What is the ultimate technology roadmap for CockroachDB adoption over 3 years?

YEAR 1: FOUNDATION & STABILIZATION
1. Month 1-3: Deploy single-region production cluster with 3-5 nodes
2. Month 4-6: Implement comprehensive monitoring, alerting, and dashboards
3. Month 7-9: Establish backup, restore, and disaster recovery procedures
4. Month 10-12: Optimize for workload, implement security hardening
   └─ Success metrics: 99.9% availability, <100ms P99 latency, zero data loss
   └─ Team: 1-2 dedicated DBAs, trained on core procedures
   └─ Automation: 50% of operational tasks automated

YEAR 2: SCALING & OPTIMIZATION
1. Q1: Expand to multi-region (active-active or active-passive)
2. Q2: Implement advanced performance tuning, query optimization
3. Q3: Establish multi-cloud strategy, evaluate failover to secondary cloud
4. Q4: Build specialized workload support (OLAP, time-series, multi-tenancy)
   └─ Success metrics: 99.95% availability, 20-30% cost reduction
   └─ Team: 3-4 DBAs, specialized roles (architect, performance engineer)
   └─ Automation: 80%+ of operational tasks automated

YEAR 3: ENTERPRISE EXCELLENCE
1. Q1: Deploy true multi-cloud (active across AWS, GCP, Azure)
2. Q2: Implement advanced compliance (GDPR, HIPAA, PCI-DSS)
3. Q3: Build predictive scaling, AI-driven optimization
4. Q4: Establish thought leadership, contribute to community
   └─ Success metrics: 99.99% availability, 40-50% cost reduction
   └─ Team: 5-7 DBAs, mature organization with rotation
   └─ Automation: 95%+ of operational tasks, self-healing infrastructure

STRATEGIC INVESTMENTS BY PHASE:

Year 1 Investments:
├─ Infrastructure: $50K-100K (cloud resources)
├─ Tools: $20K-40K (monitoring, backup solutions)
├─ Training: $10K-20K (certifications, courses)
└─ Total: $80K-160K

Year 2 Investments:
├─ Infrastructure: $100K-200K (multi-region expansion)
├─ Tools: $30K-60K (advanced tooling, automation)
├─ Training: $20K-40K (team expansion, specialization)
└─ Total: $150K-300K

Year 3 Investments:
├─ Infrastructure: $150K-300K (multi-cloud optimization)
├─ Tools: $50K-100K (AI/ML integrations, advanced features)
├─ Training: $30K-60K (leadership, innovation)
└─ Total: $230K-460K

Q242: How do I measure and communicate success to executive leadership?

EXECUTIVE DASHBOARD (Monthly KPIs):

1. BUSINESS IMPACT
   ├─ Revenue enabled by database reliability: $X per month
   ├─ Customer satisfaction score (uptime-related): 95%+
   ├─ SLA compliance rate: 99.95%+
   ├─ Incidents causing revenue loss: 0
   └─ Cost per transaction: Declining trend

2. FINANCIAL METRICS
   ├─ Total cost of ownership (TCO): Baseline vs optimized
   ├─ Cost per QPS: Declining by 10-20% annually
   ├─ Infrastructure cost optimization: $X savings/month
   ├─ ROI on tooling investments: 3-5x within 12 months
   └─ Projected savings over 3 years: $X million

3. OPERATIONAL EXCELLENCE
   ├─ Cluster availability: 99.9%+ (trending 99.95%+)
   ├─ Mean Time To Recovery (MTTR): <15 minutes
   ├─ Mean Time Between Failures (MTBF): 30+ days
   ├─ Planned maintenance windows: <4 per year
   └─ Unplanned incidents: Declining trend

4. TEAM PRODUCTIVITY
   ├─ Manual tasks eliminated: X% automation
   ├─ Team incident response time: Declining
   ├─ Team member certifications: X% of team
   ├─ Knowledge documentation coverage: 95%+
   └─ On-call satisfaction: High confidence

5. SECURITY & COMPLIANCE
   ├─ Security incidents: 0 (preventable)
   ├─ Compliance violations: 0
   ├─ Audit findings resolved: 100%
   ├─ Data protection effectiveness: 100%
   └─ Customer trust score: Increasing

QUARTERLY BUSINESS REVIEW PRESENTATION:

Q1 Review Format:
```
EXECUTIVE SUMMARY (1 slide)
├─ Key achievement: [Highlight]
├─ Risk mitigated: [Highlight]
└─ Strategic objective: [Next quarter]

FINANCIAL IMPACT (2 slides)
├─ Cost trends (6-month view)
├─ ROI on investments
└─ Projected 12-month savings

OPERATIONAL HEALTH (2 slides)
├─ Availability SLA compliance
├─ Incident trend (declining)
└─ Team capability growth

STRATEGIC INITIATIVES (1 slide)
├─ Completed this quarter
├─ In-progress initiatives
└─ Next quarter priorities

RISK DASHBOARD (1 slide)
├─ Green: Mitigated/managed
├─ Yellow: Monitoring required
└─ Red: Escalation actions
```

STORYTELLING APPROACH:
1. Start with business outcome: "Database reliability enabled $2M in new revenue"
2. Connect to operations: "Achieved 99.95% availability through..."
3. Show team capability: "Team grew from 2 to 4 DBAs with specializations"
4. Demonstrate cost efficiency: "Reduced cost per transaction by 25%"
5. Close with roadmap: "Next phase: multi-cloud for resilience"

Q243: How do I build a high-performing database operations team?

TEAM STRUCTURE BY MATURITY:

STARTUP PHASE (1-2 DBAs):
└─ Roles: Generalist DBAs handling all aspects
└─ Focus: Core operations, backup/recovery, monitoring
└─ Skills: Broad foundation across all areas
└─ Time allocation:
   ├─ 40% operational procedures
   ├─ 30% monitoring and alerting
   ├─ 20% performance tuning
   └─ 10% learning and development

GROWTH PHASE (3-4 DBAs):
└─ Roles: Generalist + specialist
   ├─ Senior DBA (lead, architecture)
   ├─ Operations DBA (monitoring, procedures)
   ├─ Performance DBA (optimization, tuning)
   └─ Junior DBA (training, procedures)
└─ Focus: Specialization, mentoring
└─ Time allocation:
   ├─ 30% specialized work
   ├─ 25% mentoring/training
   ├─ 25% architectural/strategic
   └─ 20% learning and innovation

ENTERPRISE PHASE (5-7+ DBAs):
└─ Roles: Specialized teams
   ├─ DBA Lead (strategic, business alignment)
   ├─ Database Architect (design, scalability)
   ├─ Performance Engineer (optimization, capacity)
   ├─ Operations Manager (team, process)
   ├─ Site Reliability Engineer (automation, infrastructure)
   ├─ Security DBA (compliance, encryption)
   └─ Junior/Mid DBAs (procedures, learning)
└─ Focus: Excellence, innovation, thought leadership
└─ Time allocation:
   ├─ 40% specialized expertise
   ├─ 20% cross-team collaboration
   ├─ 20% innovation/research
   └─ 20% mentoring/training

HIRING STRATEGY:

For Each Role:
1. Define clear expectations and career path
2. Look for T-shaped skills (depth + breadth)
3. Prioritize culture fit and learning mindset
4. Offer competitive compensation (+15-20% above market)
5. Include equity/ownership in compensation
6. Provide flexible work arrangements
7. Invest in continuous training (minimum 40 hours/year)

Interview Questions:
├─ "Describe your most complex database failure and recovery"
├─ "How do you stay current with database technologies?"
├─ "Tell us about your most significant optimization win"
├─ "How do you approach learning a new database system?"
└─ "What excites you about database operations?"

RETENTION STRATEGY:

1. Clear career progression (IC or management track)
2. Competitive compensation with regular reviews
3. Interesting problems and technologies
4. Conference attendance (1-2 per year)
5. Certification sponsorship
6. Mentoring and knowledge sharing opportunities
7. On-call rotation fairness
8. Recognition and celebration of wins
9. Flexible scheduling and work-from-home
10. Stock options/equity participation

TEAM DEVELOPMENT CURRICULUM:

Month 1-3: Foundations
├─ CockroachDB fundamentals
├─ Cluster setup and configuration
├─ Backup and restore procedures
├─ Basic monitoring and alerting

Month 4-6: Intermediate
├─ Performance tuning techniques
├─ Query optimization
├─ Multi-region operations
├─ Troubleshooting procedures

Month 7-12: Advanced
├─ Architecture design patterns
├─ Security hardening
├─ Compliance implementation
├─ Specialized workloads

Year 2+: Specialization
├─ Deep expertise in chosen area
├─ Leadership and mentoring
├─ Innovation projects
├─ External contributions (community, speaking)

Q244: What are the top mistakes to avoid when operating CockroachDB at scale?

TOP 10 MISTAKES AND PREVENTION:

1. MISTAKE: Insufficient backup testing
   PREVENTION:
   ├─ Test restore procedures monthly
   ├─ Maintain recovery time objectives (RTO)
   ├─ Practice disaster recovery drills quarterly
   ├─ Automate backup validation
   └─ Document all failures and learnings

2. MISTAKE: Inadequate monitoring and alerting
   PREVENTION:
   ├─ Monitor all critical metrics
   ├─ Alert on SLO deviations, not raw thresholds
   ├─ Test alerting regularly
   ├─ Maintain runbooks for each alert
   └─ Review alert effectiveness quarterly

3. MISTAKE: Over-provisioning or under-provisioning
   PREVENTION:
   ├─ Implement right-sizing procedures
   ├─ Monitor utilization trends
   ├─ Plan capacity 6-12 months in advance
   ├─ Use automated scaling when possible
   └─ Review quarterly and adjust

4. MISTAKE: Poor query optimization before scaling
   PREVENTION:
   ├─ Optimize queries before adding clusters
   ├─ Use EXPLAIN ANALYZE for all slow queries
   ├─ Create appropriate indexes
   ├─ Implement application-level caching
   └─ Profile workload before scaling

5. MISTAKE: Neglecting security hardening
   PREVENTION:
   ├─ Implement TLS for all connections
   ├─ Use RBAC with principle of least privilege
   ├─ Enable audit logging for sensitive operations
   ├─ Rotate credentials regularly
   ├─ Conduct quarterly security audits
   └─ Implement encryption at rest

6. MISTAKE: Inadequate disaster recovery planning
   PREVENTION:
   ├─ Define clear RTO/RPO targets
   ├─ Implement PITR with appropriate retention
   ├─ Test DR procedures quarterly
   ├─ Maintain DR documentation
   ├─ Have failover procedures documented
   └─ Practice incident response drills

7. MISTAKE: No capacity planning for growth
   PREVENTION:
   ├─ Track growth trends monthly
   ├─ Plan for 50% quarterly growth
   ├─ Pre-provision infrastructure
   ├─ Implement autoscaling where possible
   ├─ Regular capacity planning meetings
   └─ Document assumptions and risks

8. MISTAKE: Inadequate team training
   PREVENTION:
   ├─ Invest 40+ hours/year training per person
   ├─ Rotate on-call responsibilities fairly
   ├─ Pair junior with senior engineers
   ├─ Sponsor certifications and conferences
   ├─ Cross-train on all procedures
   └─ Regular knowledge-sharing sessions

9. MISTAKE: Insufficient change management
   PREVENTION:
   ├─ Implement formal change procedures
   ├─ Require peer review for all changes
   ├─ Test changes in staging first
   ├─ Have rollback procedures documented
   ├─ Implement gradual rollout
   └─ Maintain detailed change log

10. MISTAKE: Ignoring compliance requirements
    PREVENTION:
    ├─ Identify compliance requirements early
    ├─ Implement controls from day one
    ├─ Audit quarterly
    ├─ Maintain compliance documentation
    ├─ Budget for compliance tools/services
    └─ Train team on compliance procedures

Q245: How do I plan for unexpected growth (10x in 6 months)?

EMERGENCY SCALING PROCEDURE:

PHASE 1: IMMEDIATE ACTIONS (Day 1-2)
1. Assess current capacity
   ├─ CPU headroom: __%
   ├─ Memory headroom: __%
   ├─ Disk headroom: __%
   ├─ Network capacity: __Gbps
   └─ Current QPS: ___

2. Activate surge response team
   ├─ On-call engineers mobilized
   ├─ War room established
   ├─ Communication channels active
   └─ Leadership briefed

3. Implement immediate mitigations
   ├─ Enable follower reads
   ├─ Increase connection pool size
   ├─ Implement query rate limiting
   ├─ Clear caches and restart nodes (if needed)
   └─ Optimize slow queries

4. Begin infrastructure provisioning
   ├─ Order additional capacity (cloud auto-scaling)
   ├─ Prepare new nodes
   ├─ Plan network upgrades
   └─ Identify bottleneck (CPU, memory, disk, network)

PHASE 2: SHORT-TERM (Week 1-2)
1. Add capacity incrementally
   ├─ Add nodes in batches (3-5 at a time)
   ├─ Monitor rebalancing after each batch
   ├─ Verify performance improves
   └─ Document lessons learned

2. Optimize workload
   ├─ Identify top 10 slow queries
   ├─ Add missing indexes
   ├─ Optimize application code
   ├─ Implement caching
   └─ Reduce unnecessary logging

3. Upgrade infrastructure
   ├─ Larger instance types if needed
   ├─ Faster storage (NVMe)
   ├─ Network upgrades
   └─ Load balancer improvements

4. Implement temporary measures
   ├─ Application-level sharding if needed
   ├─ Read replicas for analytics
   ├─ Cache layer (Redis)
   └─ Queue system for async work

PHASE 3: MEDIUM-TERM (Week 3-6)
1. Architectural review
   ├─ Assess if current architecture scales to 10x
   ├─ Identify bottlenecks
   ├─ Plan long-term architecture
   └─ Document decisions

2. Strategic improvements
   ├─ Implement data sharding if needed
   ├─ Multi-region deployment
   ├─ Advanced caching strategies
   ├─ Database-level optimization
   └─ Application redesign where necessary

3. Automation implementation
   ├─ Auto-scaling policies
   ├─ Automated failover
   ├─ Performance-based alerting
   ├─ Predictive scaling
   └─ Self-healing infrastructure

4. Team scaling
   ├─ Hire additional DBAs
   ├─ Expand DevOps team
   ├─ Cross-train existing staff
   └─ Establish specialized roles

PHASE 4: LONG-TERM (Month 2-6)
1. Sustainable architecture
   ├─ Implement designed-for-scale architecture
   ├─ Multi-region active-active if needed
   ├─ Advanced replication and failover
   ├─ Predictive capacity management
   └─ Cost optimization

2. Operational excellence
   ├─ Automated incident response
   ├─ Predictive monitoring
   ├─ Advanced analytics
   ├─ Self-service capabilities
   └─ Full automation of procedures

3. Team maturity
   ├─ Specialized teams established
   ├─ Leadership roles filled
   ├─ Knowledge base complete
   ├─ Training programs in place
   └─ Continuous improvement culture

BUDGET IMPACT (Typical 10x Growth Scenario):

Immediate (Month 1): $100K-200K
├─ Additional infrastructure
├─ Tools and licensing
├─ Emergency team expansion
└─ Premium support

Short-term (Month 2-3): $150K-300K
├─ Continued infrastructure
├─ Architectural improvements
├─ Team hiring
└─ Training and development

Medium-term (Month 4-6): $200K-400K
├─ Optimal architecture deployment
├─ Full team in place
├─ Automation implementation
└─ Advanced tooling

Total 6-month impact: $450K-900K (typically offset by revenue growth)

Q246: How do I negotiate with CockroachDB vendor for enterprise support?

ENTERPRISE SUPPORT NEGOTIATION STRATEGY:

ASSESSMENT PHASE:

1. Determine your leverage
   ├─ Number of databases: ___
   ├─ Annual spend: $___
   ├─ Expected growth: ___% per year
   ├─ Critical to business: YES / NO
   ├─ Alternatives available: YES / NO
   └─ Competitive pressure: HIGH / MEDIUM / LOW

2. Define your needs
   ├─ SLA availability: 99.9% / 99.95% / 99.99%
   ├─ Response time SLA: 1hr / 4hrs / 8hrs
   ├─ Priority support channels: Phone / Slack / Email
   ├─ Dedicated engineer needed: YES / NO
   ├─ Training budget: $___
   └─ Architectural consulting needed: YES / NO

3. Research vendor options
   ├─ CockroachDB support tiers
   ├─ Competitor offerings
   ├─ Market rates for similar enterprises
   ├─ Reference customers similar to you
   └─ Total cost of ownership comparison

NEGOTIATION STRATEGY:

1. Build a strong business case
   ```
   Enterprise Support Value Proposition:
   ├─ Revenue protected by uptime: $X per minute down
   ├─ Cost of incident response: $X without support
   ├─ Risk reduction through expert support: $X value
   ├─ Faster time-to-resolution: X hours saved per incident
   └─ Total ROI: X% annually
   ```

2. Prepare negotiation tactics
   ├─ Anchor high: Start 30-40% above your target
   ├─ Document your criticality
   ├─ Highlight growth trajectory
   ├─ Reference competitive options
   ├─ Bundle services (support + training + consulting)
   └─ Long-term commitment for better rates

3. Key negotiation points
   ├─ SLA levels: 99.9% → 99.99% (+$X/month)
   ├─ Response time: 8hr → 1hr → 15min (+$X/month)
   ├─ Dedicated engineer: +$X/month (reduce if possible)
   ├─ Included training hours: X hours/year (+$X/month)
   ├─ Architectural consulting: X hours/quarter (+$X/month)
   ├─ Phone support availability: 24/7 vs business hours
   ├─ Community vs enterprise license: Savings opportunity
   └─ Multi-year discount: 15-25% for 3-year commitment

4. Create a tiered proposal
   ```
   OPTION A: Essential ($X/month)
   ├─ Standard support
   ├─ 4hr response time
   ├─ Business hours only
   └─ 99.9% SLA

   OPTION B: Professional ($X/month)
   ├─ Priority support
   ├─ 1hr response time
   ├─ 24/7/365 availability
   ├─ 99.95% SLA
   ├─ Monthly architectural review
   └─ 20 training hours/year

   OPTION C: Enterprise ($X/month)
   ├─ Premium support
   ├─ 15min response time
   ├─ Dedicated support engineer
   ├─ 99.99% SLA
   ├─ Quarterly architectural consulting
   ├─ 40 training hours/year
   ├─ Priority feature requests
   └─ Discount on professional services
   ```

5. Negotiate final terms
   ├─ Lock in rates for 3 years
   ├─ Include escalation procedures
   ├─ Define SLA remedies (credits)
   ├─ Include growth accommodation
   ├─ Annual review and adjustment
   └─ Cancellation terms

COST OPTIMIZATION:

1. Bundling discounts
   ├─ Support + training: 10% savings
   ├─ Support + professional services: 15% savings
   ├─ Full bundle (support + training + services): 20% savings

2. Volume discounts
   ├─ Tier 1 (1-5 databases): Standard pricing
   ├─ Tier 2 (6-20 databases): 10% discount
   ├─ Tier 3 (21+ databases): 15-20% discount

3. Commitment discounts
   ├─ 1-year commitment: 5% discount
   ├─ 3-year commitment: 15% discount
   ├─ 5-year commitment: 25% discount

ALTERNATIVE COST APPROACHES:

1. Hybrid support model
   ├─ CockroachDB support: Tier 2 (critical issues)
   ├─ Internal expertise: Tier 1 and 2 support
   ├─ Vendor consulting: As-needed (reserve $X/year)
   └─ Training: Invest in staff training

2. Service package composition
   ├─ 50% on vendor support
   ├─ 30% on team training
   ├─ 20% on consulting/tooling
   └─ Total: Same investment, better ROI

Q247: What governance framework should I implement?

COCKROACHDB GOVERNANCE FRAMEWORK:

POLICY FRAMEWORK:

1. Database Access Policy
   ├─ Who can access which databases
   ├─ Role-based access control (RBAC) implementation
   ├─ Password and credential policies
   ├─ MFA requirements for sensitive access
   ├─ Audit logging for all access
   └─ Quarterly access reviews

2. Change Management Policy
   ├─ All changes require approval
   ├─ Peer review mandatory
   ├─ Test in staging first
   ├─ Documented rollback procedures
   ├─ Change calendar maintained
   └─ Post-change verification required

3. Backup and Recovery Policy
   ├─ Backup frequency (minimum: daily)
   ├─ Retention period (minimum: 30 days)
   ├─ Cross-region backups required
   ├─ Recovery testing quarterly
   ├─ RTO/RPO targets: <1 hour / <15 minutes
   └─ Disaster recovery drills annually

4. Security Policy
   ├─ Encryption at rest mandatory
   ├─ Encryption in transit (TLS 1.3+)
   ├─ Data classification levels
   ├─ Sensitive data handling procedures
   ├─ Security incident response procedures
   ├─ Penetration testing annually
   └─ Vulnerability scanning quarterly

5. Performance and Capacity Policy
   ├─ SLA targets: 99.9%+ availability
   ├─ Latency targets: <100ms P99
   ├─ Capacity headroom: 1.5x planned growth
   ├─ Auto-scaling policies defined
   ├─ Monthly capacity reviews
   └─ Quarterly performance audits

ROLES AND RESPONSIBILITIES:

1. Database Governance Board (Monthly meeting)
   ├─ CTO or VP of Engineering (chair)
   ├─ Database Lead/DBA
   ├─ Security Officer
   ├─ Finance/Cost Lead
   └─ Product/Application Lead

   Responsibilities:
   ├─ Review policy compliance
   ├─ Approve major changes
   ├─ Address SLA violations
   ├─ Plan capacity and upgrades
   └─ Manage budget

2. Database Access Review Committee (Quarterly)
   ├─ Database Lead
   ├─ Security Officer
   ├─ Finance/Audit
   └─ HR (for employee changes)

   Responsibilities:
   ├─ Review all database access
   ├─ Approve access requests
   ├─ Remove unnecessary access
   ├─ Verify compliance
   └─ Document findings

3. Change Advisory Board (For major changes)
   ├─ Database Lead
   ├─ Application Lead
   ├─ Security Officer
   ├─ Operations Lead
   └─ Product Lead

   Responsibilities:
   ├─ Review proposed changes
   ├─ Assess impact and risk
   ├─ Approve/reject changes
   ├─ Schedule implementation
   └─ Monitor rollout

COMPLIANCE MONITORING:

1. Automated compliance checks
   ├─ Daily database configuration audit
   ├─ Weekly access control verification
   ├─ Monthly SLA compliance report
   ├─ Quarterly security assessment
   ├─ Annual external audit
   └─ Continuous vulnerability scanning

2. Compliance reporting
   ├─ Monthly report to governance board
   ├─ Quarterly compliance dashboard
   ├─ Annual external audit report
   ├─ Issue tracking and remediation
   └─ Policy effectiveness review annually

Q248: How do I build a world-class documentation system?

DOCUMENTATION FRAMEWORK:

ARCHITECTURE DOCUMENTATION:

1. System Architecture Diagrams
   ├─ Overall cluster architecture
   ├─ Multi-region topology
   ├─ Network architecture
   ├─ Disaster recovery architecture
   └─ Data flow diagrams

2. Design Decision Documentation
   ├─ Why this database
   ├─ Why this topology
   ├─ Why this replication strategy
   ├─ Alternatives considered
   ├─ Trade-offs made
   └─ Future considerations

3. Configuration Documentation
   ├─ Cluster settings documented
   ├─ Rationale for each setting
   ├─ Performance tuning parameters
   ├─ Security configurations
   └─ Backup configurations

OPERATIONAL DOCUMENTATION:

1. Runbooks (Step-by-step procedures)
   ├─ Cluster startup/shutdown
   ├─ Node failure recovery
   ├─ Backup and restore
   ├─ Scaling procedures
   ├─ Troubleshooting procedures
   ├─ Performance optimization
   ├─ Emergency procedures
   └─ Each runbook includes:
      ├─ Prerequisites
      ├─ Step-by-step instructions
      ├─ Verification steps
      ├─ Rollback procedures
      ├─ Escalation contacts
      └─ Estimated time to complete

2. Troubleshooting Guides
   ├─ Symptom → Root cause → Resolution
   ├─ Performance issues (by type)
   ├─ Data issues
   ├─ Replication issues
   ├─ Network issues
   ├─ Security issues
   └─ Common error messages and fixes

3. Checklists
   ├─ Pre-production deployment
   ├─ Post-deployment verification
   ├─ Monthly maintenance
   ├─ Quarterly review
   ├─ Annual audit
   ├─ Incident response
   └─ Compliance verification

KNOWLEDGE DOCUMENTATION:

1. FAQ Database
   ├─ Common questions organized by topic
   ├─ Answers with context and examples
   ├─ Links to relevant runbooks
   ├─ Examples and use cases
   └─ Regularly updated based on support tickets

2. Best Practices Guide
   ├─ Schema design best practices
   ├─ Query optimization best practices
   ├─ Security best practices
   ├─ Operational best practices
   ├─ Performance tuning best practices
   └─ Cost optimization best practices

3. Case Studies and Incident Reports
   ├─ Post-mortem from each incident
   ├─ Root cause analysis
   ├─ Resolution procedures
   ├─ Preventive measures
   ├─ Lessons learned
   └─ Shared learning from incidents

DOCUMENTATION TECHNOLOGY:

1. Centralized Wiki
   ├─ Single source of truth
   ├─ Versioning and history
   ├─ Search capability
   ├─ Access controls
   └─ Integration with tools (Slack, Jira)

2. Documentation-as-Code
   ├─ Markdown files in git repository
   ├─ Version control integration
   ├─ Automated publishing
   ├─ Diff tracking
   └─ Review process for changes

3. Tool Integration
   ├─ Jira: Link incidents to documentation
   ├─ Slack: Auto-link relevant runbooks in war room
   ├─ Confluence: Searchable documentation
   ├─ GitHub: Version-controlled runbooks
   └─ PagerDuty: On-call documentation

DOCUMENTATION MAINTENANCE:

1. Review and Update Schedule
   ├─ Runbooks: Quarterly review + after each use
   ├─ Architecture: Semi-annual review
   ├─ Procedures: After each change
   ├─ FAQ: Monthly update based on support tickets
   ├─ Best practices: Annual comprehensive review
   └─ Case studies: Immediately after incidents

2. Ownership Model
   ├─ Each runbook has designated owner
   ├─ Quarterly review meetings
   ├─ Peer review process
   ├─ Version control with approval
   └─ Archive obsolete documentation

3. Documentation Metrics
   ├─ Pages in documentation system
   ├─ Last updated date for each page
   ├─ Accuracy rating (team survey)
   ├─ Usage metrics
   ├─ Search hit frequency
   └─ Feedback from users

Q249: What is my path to CockroachDB certification and thought leadership?

CERTIFICATION PATH:

INDIVIDUAL CERTIFICATIONS:

1. CockroachDB Associate (Foundation)
   ├─ Topics: Cluster setup, basic operations, backup/restore
   ├─ Study time: 20-40 hours
   ├─ Exam: Multiple choice, 90 minutes
   ├─ Cost: $300
   ├─ Validity: 2 years
   └─ Value: Demonstrates foundational knowledge

2. CockroachDB Professional (Intermediate)
   ├─ Prerequisites: Associate certification recommended
   ├─ Topics: Advanced operations, performance tuning, multi-region
   ├─ Study time: 40-60 hours
   ├─ Exam: Mixed format (multiple choice + scenarios)
   ├─ Cost: $500
   ├─ Validity: 2 years
   └─ Value: Demonstrates operational expertise

3. CockroachDB Architect (Advanced)
   ├─ Prerequisites: Professional certification required
   ├─ Topics: Architecture design, enterprise features, compliance
   ├─ Study time: 60-80 hours
   ├─ Exam: Case study analysis + practical scenarios
   ├─ Cost: $700
   ├─ Validity: 3 years
   └─ Value: Demonstrates architectural expertise

TEAM CERTIFICATION STRATEGY:

Year 1: Foundation
├─ All team members: Associate certification
├─ Investment: $300 × 4 = $1,200
├─ Timeline: 3-6 months
└─ Goal: Team aligned on fundamentals

Year 2: Specialization
├─ Senior engineers: Professional certification
├─ Investment: $500 × 2 = $1,000
├─ Timeline: 6-12 months
└─ Goal: Specialized expertise established

Year 3: Architecture
├─ Lead architect: Architect certification
├─ Investment: $700 × 1 = $700
├─ Timeline: 12-18 months
└─ Goal: Expert-level architectural knowledge

THOUGHT LEADERSHIP PATH:

1. COMMUNITY PARTICIPATION

   Conference Speaking:
   ├─ CockroachDB Summit (annual)
   ├─ Database conferences (PostgreSQL, Percona, etc.)
   ├─ Tech meetups (local database groups)
   ├─ Company tech talks
   └─ Strategy: Start with internal talks, then meetups, then conferences

   Writing:
   ├─ Blog posts on engineering blog
   ├─ Technical articles
   ├─ Case studies on your experience
   ├─ Guest posts on industry blogs
   ├─ Published research or white papers
   └─ Strategy: Start with internal documentation, then publish externally

   Open Source:
   ├─ Contribute to CockroachDB project
   ├─ Contribute to ecosystem tools
   ├─ Create open-source tools for community
   ├─ Participate in community forums
   └─ Strategy: Start small, build reputation, increase contributions

2. CONTENT CREATION TIMELINE

   Month 1-3: Foundation
   ├─ Publish 1 internal technical article
   ├─ Present at internal tech talk
   ├─ Engage in community forums
   └─ Output: Demonstrate expertise internally

   Month 4-6: Local presence
   ├─ Present at local meetup
   ├─ Publish 1 external blog post
   ├─ Participate actively in forums
   └─ Output: Establish local expertise

   Month 7-12: Regional presence
   ├─ Present at regional conference
   ├─ Publish 2-3 blog posts
   ├─ Start contributing to open source
   └─ Output: Regional thought leadership

   Year 2: National presence
   ├─ Present at major conference
   ├─ Publish regular blog series
   ├─ Active open source contributor
   ├─ Recognized expert in niche
   └─ Output: National thought leadership

   Year 3+: Industry leadership
   ├─ Keynote speaker at conferences
   ├─ Author of published articles/book
   ├─ Major contributor to ecosystem
   ├─ Invited expert on panels
   └─ Output: Industry recognition

TANGIBLE THOUGHT LEADERSHIP OUTPUTS:

1. Blog Series (12 posts over year)
   └─ Topics: Deep dives on complex issues you've solved

2. Presentation Deck (40-60 slides)
   └─ Topic: Architecture patterns or optimization techniques

3. White Paper
   └─ Topic: Research or findings from your experience

4. Case Study
   └─ Topic: Your company's journey with CockroachDB

5. Open Source Tool
   └─ Topic: Useful utility for CockroachDB community

6. Published Article
   └─ Topic: Industry publication on database topics

Q250: What final advice would you give to someone starting their CockroachDB journey?

FINAL WISDOM AND CLOSING REMARKS:

1. START SIMPLE, SCALE GRADUALLY
   ├─ Don't try to build complex architecture immediately
   ├─ Deploy single-region first
   ├─ Perfect operations before scaling
   ├─ Add complexity only when necessary
   ├─ Validate each layer before building on it
   └─ Principle: "Make it work, make it reliable, make it fast, make it scale"

2. INVEST IN FUNDAMENTALS FIRST
   ├─ Backup and restore procedures (critical)
   ├─ Monitoring and alerting (crucial)
   ├─ Documentation (essential)
   ├─ Team training (foundational)
   ├─ Security hardening (non-negotiable)
   └─ Advanced optimization (later)

3. BUILD OPERATIONAL EXCELLENCE
   ├─ Operations are more important than features
   ├─ Automate everything possible
   ├─ Document every procedure
   ├─ Make operations boring (predictable)
   ├─ Focus on reliability over performance initially
   └─ Performance optimization comes after reliability

4. INVEST IN YOUR TEAM
   ├─ Great operations come from great people
   ├─ Train continuously
   ├─ Cross-train all team members
   ├─ Reward expertise and initiative
   ├─ Create psychological safety for learning
   ├─ Hire for learning mindset, not just experience
   └─ Team is your biggest asset

5. LISTEN TO THE COMMUNITY
   ├─ CockroachDB community is helpful and generous
   ├─ Forums: Ask questions, search for answers
   ├─ Conferences: Learn from others' experiences
   ├─ Open source: Contribute and learn
   ├─ Network: Build relationships with other operators
   └─ Wisdom: Others have solved your problems before

6. MEASURE WHAT MATTERS
   ├─ Availability (uptime)
   ├─ Reliability (MTBF, MTTR)
   ├─ Performance (latency, throughput)
   ├─ Cost efficiency (cost per QPS)
   ├─ Team productivity (automation level)
   ├─ Customer satisfaction (perception of database)
   └─ Don't obsess over metrics that don't matter

7. EMBRACE CONTINUOUS IMPROVEMENT
   ├─ Review and optimize quarterly
   ├─ Implement learnings from incidents
   ├─ Evolve architecture as workload changes
   ├─ Stay current with CockroachDB versions
   ├─ Test new features in dev/staging first
   ├─ Build feedback loops
   └─ Improvement is never done

8. PREPARE FOR FAILURE
   ├─ Failure will happen
   ├─ Plan for it (backups, failover, recovery)
   ├─ Practice recovery procedures regularly
   ├─ Have incident response playbooks ready
   ├─ Learn from each failure
   ├─ Share learnings with team
   └─ "Hope is not a plan" - Always have a backup plan

9. THINK LONG-TERM
   ├─ CockroachDB is built for the long run
   ├─ Think 3-5 years ahead
   ├─ Plan for 10x growth
   ├─ Invest in architecture that scales
   ├─ Build team capability that compounds
   ├─ Choose decisions that enable future flexibility
   └─ Short-term pain for long-term gain often pays off

10. ENJOY THE JOURNEY
    ├─ Database operations is fascinating
    ├─ You're solving real problems at scale
    ├─ Your work enables business success
    ├─ Community is supportive and collaborative
    ├─ Learning never stops
    ├─ Celebrate successes and learnings from failures
    └─ Build something you're proud of

USE THIS KNOWLEDGE WELL. YOUR JOURNEY TO OPERATIONAL EXCELLENCE BEGINS NOW.

================================================================================


