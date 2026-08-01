# CockroachDB DATABASE ADMINISTRATION 250 FAQs

---

## SECTION 1: CLUSTER SETUP AND INITIALIZATION

### Q1: How do I start a single-node CockroachDB cluster for development?

1. Download the latest CockroachDB binary from binaries.cockroachdb.com for your OS.
2. Create data directory: mkdir -p data
3. Start node: cockroach start-single-node --insecure --listen-addr=localhost:26257 --http-addr=localhost:8080
4. Access DB Console at http://localhost:8080 in browser.
5. Connect via SQL: cockroach sql --insecure --host=localhost --port=26257
6. Verify cluster: SELECT * FROM crdb_internal.node_runtime_info;

### Q2: What are prerequisites for multi-node CockroachDB production cluster?

1. Minimum 3 nodes for resilience with odd-numbered count for quorum.
2. Low-latency networking between nodes, typically in same datacenter/region.
3. Firewall rules: port 26257 for inter-node communication, 8080 for DB Console.
4. Shared storage or NFS mounts if using NFS for backup destinations.
5. Generate TLS certificates: cockroach cert create-ca --certs-dir=certs --ca-key=ca.key
6. Minimum 4GB RAM per node, 8GB+ recommended; fast SSD storage critical.

### Q3: How do I initialize a secure 3-node cluster with TLS certificates?

1. Create directories: mkdir -p certs my-safe-directory
2. Create CA: cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
3. Create node certs: cockroach cert create-node --certs-dir=certs --ca-key=my-safe-directory/ca.key localhost node1.example.com node2.example.com node3.example.com
4. Create client cert: cockroach cert create-client --certs-dir=certs --ca-key=my-safe-directory/ca.key root
5. Start nodes: cockroach start --certs-dir=certs --listen-addr=node1.example.com:26257 --advertise-addr=node1.example.com:26257 --join=node1.example.com:26257,node2.example.com:26257,node3.example.com:26257
6. Cluster initialization completes automatically when quorum is reached.

### Q4: What is difference between insecure mode and secure mode deployment?

1. Insecure mode: no TLS encryption, unauthenticated access, development only.
2. Secure mode: requires TLS certificates, enforces authentication, production-required.
3. Insecure: faster performance but no data encryption in transit.
4. Secure: encryption overhead justified by security benefits.
5. Production deployments must use secure mode per industry best practices.
6. Hybrid approach: insecure for testing, secure for production.

### Q5: How do I configure cluster to use join token instead of explicit IP addresses?

1. Start first node: cockroach start-single-node outputs cluster ID and certificate.
2. Subsequent nodes use --join flag: cockroach start --join=existing-node:26257
3. Node automatically discovers cluster members via gossip protocol.
4. Useful for Kubernetes deployments where addresses unknown in advance.
5. Gossip port (26257) must be open between all nodes.
6. Join flag can point to any existing cluster node.

### Q6: What should I consider when choosing between self-hosted and CockroachCloud?

1. Self-hosted: full infrastructure control, custom DR strategies, multi-region flexibility.
2. CockroachCloud: managed service, automated backups/upgrades, guaranteed uptime SLAs.
3. Self-hosted: infrastructure costs; CockroachCloud: consumption-based pricing.
4. CockroachCloud tiers: Basic (development), Standard (production), Advanced (mission-critical).
5. Self-hosted requires operational expertise; CockroachCloud reduces operational burden.
6. Choose based on team expertise, compliance requirements, and workload needs.

### Q7: How do I set up CockroachDB on Kubernetes using CockroachDB Operator?

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

### Q8: What are hardware requirements for production CockroachDB node?

1. CPU: minimum 4 cores, 8+ cores recommended for production.
2. RAM: minimum 4GB, 8GB+ recommended for query memory requirements.
3. Storage: fast SSD/NVMe critical; plan for 1-2x data size for WAL/snapshots.
4. Network: gigabit+ connectivity; low inter-node latency essential.
5. Disk I/O: at least 1000 IOPS per node for consistent performance.
6. Over-provision by 20-30% for unexpected growth and performance headroom.

### Q9: How do I configure cluster locality awareness for multi-region deployments?

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

### Q10: What happens during cluster initialization and how long does it take?

1. First node creates system tables: system.namespace, system.zones, system.settings.
2. Subsequent nodes replicate system data from existing nodes during bootstrap.
3. Cluster operational as soon as first node starts; nodes join gradually.
4. Full initialization typically takes 30-60 seconds depending on latency/disk speed.
5. Monitor progress through DB Console showing node status and replica distribution.
6. Until quorum available, some operations may have latency issues.

### Q11: How do I migrate existing PostgreSQL database to CockroachDB?

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

### Q12: What configuration changes recommended before going to production?

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

### Q13: How do I verify cluster is properly initialized and healthy?

```sql
-- Check node runtime info
SELECT * FROM crdb_internal.node_runtime_info;

-- Verify system tables
SELECT * FROM system.zones;
SELECT * FROM system.node_liveness;

-- Check replication status
SELECT * FROM crdb_internal.reports_meta;

-- View cluster info
SELECT node_id, address, sql_addr FROM crdb_internal.nodes;
```

1. Query node runtime info: SELECT * FROM crdb_internal.node_runtime_info
2. Verify system tables initialized: SELECT * FROM system.zones
3. Check replication: SELECT * FROM crdb_internal.reports_meta
4. Access DB Console dashboard showing cluster health and metrics.
5. Run simple test: SELECT 1 to confirm SQL layer operational.
6. Check node liveness: SELECT * FROM system.node_liveness

### Q14: What are common startup flags and their purposes?

1. --store: storage location, format: --store=type=ssd,path=/var/lib/cockroach
2. --listen-addr: SQL/gRPC address, default localhost:26257
3. --http-addr: DB Console server, default localhost:8080
4. --join: existing cluster nodes to join, comma-separated
5. --certs-dir: TLS certificates directory path
6. --insecure: disable TLS requirement, development only.

### Q15: How do I set up load balancer in front of CockroachDB?

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

---

## SECTION 2: NODE MANAGEMENT AND SCALING

### Q16: How do I gracefully remove node from running cluster?

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

### Q17: What is range rebalancing and how does it work?

1. Automatic data redistribution across cluster nodes for load balance.
2. CockroachDB continuously evaluates range health and moves replicas.
3. Rebalancing considers replica count, zone constraints, locality.
4. Process happens gradually in background to avoid I/O overwhelming.
5. Monitor progress through DB Console Ranges section.
6. Disable temporarily if needed: SET CLUSTER SETTING kv.allocator.mode='off'

### Q18: How do I add new node to existing cluster?

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

### Q19: How do I identify and handle unbalanced range distribution?

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

### Q20: What happens when node runs out of disk space?

1. Node enters read-only mode to prevent data corruption.
2. Other nodes continue normal operations if maintaining quorum.
3. Logs report disk space errors with clear messages.
4. Resolve by adding disk space or moving store to larger disk.
5. Node automatically resumes operations after space available.
6. Monitor disk usage proactively to prevent this situation.

### Q21: How do I scale cluster horizontally to handle increased load?

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

### Q22: How do I configure and manage different store types on node?

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

### Q23: What is purpose of --attrs flag and how do I use it?

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

### Q24: How do I handle node that is permanently unavailable or failed?

1. Confirm node unavailability: check process, logs, connectivity.
2. Cluster automatically rebalances data to healthy nodes.
3. No intervention needed unless affects quorum.
4. If more than half nodes fail, restore from backup.
5. Remove from load balancer to prevent connection attempts.
6. Decommission once ranges fully rebalanced.

### Q25: What should I consider when sizing cluster CPU requirements?

1. Baseline: 10-20% CPU at rest for system operations.
2. Each query thread uses approximately 1 core.
3. Replication/consensus consume CPU; higher replicas = more overhead.
4. Bulk operations spike CPU usage temporarily.
5. Monitor peak load; stay below 80% for headroom.
6. Auto-scale or add capacity when consistently hitting limits.

---

## SECTION 3: DATA BACKUP AND RESTORE

### Q26: What is difference between full and incremental backups?

1. Full backup: entire database state at specific timestamp, contains all data/schemas.
2. Incremental: only changes since last backup, uses less storage.
3. Full can restore independently; incremental requires preceding full.
4. Incremental completes faster, requires less bandwidth.
5. Retention policies auto-delete old backups.
6. Combine both for optimal recovery time and storage cost.

### Q27: How do I create full backup to AWS S3?

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

### Q28: How do I restore full backup from Google Cloud Storage?

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

### Q29: How do I set up incremental backups with backup schedule?

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

### Q30: What is point-in-time recovery and how do I use it?

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

### Q31: How do I restore only specific tables from backup?

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

### Q32: How do I restore entire database from backup?

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

### Q33: What should I do if backup job fails or is interrupted?

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

### Q34: How do I backup only specific databases to reduce storage?

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

### Q35: How do I manage backup retention policies to control storage costs?

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

---

## SECTION 4: DISASTER RECOVERY SCENARIOS

### Q36: How do I recover from accidental deletion of critical data?

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

### Q37: How do I recover from data corruption in table?

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

### Q38: What should I do if majority of nodes fail simultaneously?

1. Cluster enters read-only mode as quorum unavailable.
2. Existing range leaders continue serving reads.
3. Assess failure scope and determine if hardware/network/software.
4. If recovery too long, initiate disaster recovery from backups.
5. Restore cluster from recent backup.
6. Redirect traffic to recovered cluster.

### Q39: How do I handle total cluster loss scenario?

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

### Q40: How do I perform failover to standby cluster using physical replication?

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

---

## SECTION 5: USER MANAGEMENT AND SECURITY

### Q41: How do I create new SQL user and assign basic privileges?

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

### Q42: How do I change user password?

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

### Q43: How do I implement role-based access control (RBAC) for applications?

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

### Q44: How do I revoke privileges from user or role?

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

### Q45: What is difference between roles and users in CockroachDB?

1. Technically same entity with key difference.
2. Users: CREATE USER, can log in by default.
3. Roles: CREATE ROLE, NOLOGIN by default.
4. Roles serve as permission groups.
5. Multiple users can be members of role.
6. Use roles for grouping, users for individual access.

---

## SECTION 6: MONITORING AND OBSERVABILITY

### Q46: How do I set up Prometheus monitoring for CockroachDB cluster?

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

### Q47: What are key CockroachDB metrics I should monitor?

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

### Q48: How do I create alerting rules for critical cluster issues?

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

### Q49: How do I use DB Console for cluster monitoring?

1. Access at http://cluster-node:8080
2. Overview dashboard: cluster health, node status, metrics.
3. Ranges page: replica distribution, unbalanced nodes.
4. SQL dashboard: query latency, throughput, execution details.
5. Storage page: disk usage, range distribution by size.
6. Databases page: table sizes, key metrics per database.

### Q50: How do I debug slow queries using query execution insights?

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

---

## SECTION 7: PERFORMANCE TUNING

### Q51: How do I identify and optimize slow queries?

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

### Q52: How do I create effective indexes for query performance?

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

### Q53: How do I tune cluster settings for write-heavy workloads?

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

### Q54: How do I optimize read-heavy workloads?

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

### Q55: How do I reduce query latency for time-sensitive operations?

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

---

## SECTION 8: REPLICATION AND MULTI-REGION

### Q56: How do I configure replication factor for high availability?

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

### Q57: How do I set up multi-region cluster for data locality?

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

### Q58: How do I implement cross-region data replication?

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

### Q59: How do I manage replica placement constraints for compliance?

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

### Q60: How do I optimize cross-region performance?

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

---

## SECTION 9: TROUBLESHOOTING AND MAINTENANCE

### Q61: How do I diagnose connectivity issues between cluster nodes?

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

### Q62: How do I troubleshoot cluster initialization failures?

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

### Q63: How do I resolve "node_id not set" errors?

1. Indicates node cannot assign ID during startup.
2. Check node can connect to cluster via --join.
3. Verify network connectivity to join targets.
4. Verify DNS resolution if using hostnames.
5. Check system time synchronization.
6. Review logs for detailed error messages.

### Q64: How do I fix "range does not exist" errors in queries?

1. Indicates range splits/rebalancing in progress.
2. Retry query; should succeed when rebalancing completes.
3. If persists, indicates data corruption or system issues.
4. Check cluster health: verify nodes operational.
5. Review system logs for corruption/leadership issues.
6. Contact support if error persists.

### Q65: How do I troubleshoot "context deadline exceeded" errors?

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

---

## SECTION 10: ADVANCED SCENARIOS

### Q66: How do I implement schema versioning for zero-downtime migrations?

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

### Q67: How do I handle extremely large tables (terabyte-scale)?

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

### Q68: How do I implement bulk delete operations without impacting cluster?

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

### Q69: How do I implement cross-database transactions for ACID consistency?

1. CockroachDB does not support cross-database transactions.
2. Implement application-level coordination.
3. Use two-phase commit pattern for cross-cluster.
4. Document transaction scope limitations.
5. Design databases to minimize cross-database needs.
6. Use event sourcing or saga patterns.

### Q70: How do I optimize for specific query patterns in workload clusters?

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

---

## SECTION 11: COCKROACHCLOUD ADMINISTRATION

### Q71: How do I create CockroachCloud cluster for first time?

1. Visit https://cockroachlabs.cloud and sign up.
2. Create organization and select cloud provider (AWS or GCP) and region.
3. Choose cluster type: Basic (development), Standard (production), Advanced (mission-critical).
4. Configure node count, machine type, storage capacity.
5. Accept terms and initiate creation; provisioning takes 5-10 minutes.
6. Access connection string from cluster overview page.

### Q72: What is difference between CockroachCloud cluster tiers?

1. Basic: consumption-based pricing, suitable for development.
2. Standard: pre-provisioned compute, predictable costs, production-ready.
3. Advanced: multi-region, higher SLAs, PCI compliance features.
4. Basic: lower minimum cost but less control.
5. Standard/Advanced: managed backups with configurable retention.
6. Choose based on criticality, SLA, compliance requirements.

### Q73: How do I configure automated backups in CockroachCloud?

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

### Q74: How do I scale CockroachCloud cluster up or down?

1. Access cluster settings through console Cluster Overview.
2. Modify node count: add for capacity, remove for cost reduction.
3. Change machine type for performance adjustment.
4. Scaling completes without downtime; nodes join/leave gracefully.
5. Monitor scaling progress through console.
6. Costs adjust immediately based on new configuration.

### Q75: How do I manage maintenance windows in CockroachCloud?

1. Set maintenance window in cluster settings.
2. Choose time during low traffic to minimize impact.
3. Single-node clusters experience downtime.
4. Multi-node clusters continue during rolling upgrades.
5. Defer patch upgrades to later date if inconvenient.
6. Monitor completion through console activity log.

---

## SECTION 12: ADVANCED PERFORMANCE OPTIMIZATION

### Q76: How do I implement prepared statements for improved performance?

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

### Q77: How do I optimize bulk insert performance?

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

### Q78: How do I implement query result streaming for large datasets?

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

---

## SECTION 13: ADVANCED DISASTER RECOVERY

### Q79: How do I implement backup encryption with customer-managed keys?

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

### Q80: How do I test backup restore procedures without affecting production?

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

---

## SECTION 14: SECURITY HARDENING AND COMPLIANCE

### Q81: How do I implement SSL/TLS certificate rotation without downtime?

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

### Q82: How do I implement secure password storage and rotation?

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

### Q83: How do I implement network segmentation with CockroachDB?

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

### Q84: How do I audit SQL statement execution for compliance?

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

### Q85: How do I implement fine-grained access control for sensitive data?

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

---

## SECTION 15: MIGRATION AND DATA INTEGRATION

### Q86: How do I migrate from PostgreSQL to CockroachDB with zero downtime?

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

### Q87: How do I implement change data capture (CDC) for data integration?

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

### Q88: How do I integrate CockroachDB with data warehousing systems?

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

---

## SECTION 16: OPERATIONAL EXCELLENCE

### Q89: How do I implement runbook automation for common procedures?

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

### Q90: What is complete production deployment checklist?

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

---

## SECTION 17: ADVANCED CLUSTERING PATTERNS

### Q91: How do I implement CQRS pattern with CockroachDB?

1. Separate write model (commands) from read model (queries).
2. Write model executes transactions and persists to primary cluster.
3. Read model asynchronously updates read-optimized views.
4. Use CDC to stream changes from write model to read replicas.
5. Read model optimized for specific query patterns.
6. Implement eventual consistency handling in application logic.

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

### Q92: How do I implement event sourcing pattern with CockroachDB?

1. Store immutable events in event log table as single source of truth.
2. Derive application state by replaying events.
3. Implement snapshots to avoid replaying entire history.
4. Use CDC to stream events to external systems.
5. Implement event versioning for schema evolution.
6. Test event replay thoroughly.

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

### Q93: How do I implement saga pattern for distributed transactions?

1. Saga orchestrates distributed transactions across services.
2. Each step is transaction; compensating transactions handle rollback.
3. Implement saga choreography: services respond to events.
4. Implement saga orchestration: central orchestrator coordinates.
5. Monitor execution; alert on compensation/rollback.
6. Test failure scenarios.

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

### Q94: How do I implement bulkhead pattern for fault isolation?

1. Isolate critical workloads in separate resource pools.
2. Limit resource allocation per workload: memory, CPU, connections.
3. Prevent one workload exhausting cluster resources.
4. Monitor bulkhead utilization.
5. Implement automatic throttling when limit exceeded.
6. Test under failure scenarios.

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

### Q95: How do I implement circuit breaker pattern for external calls?

1. Monitor external service health continuously.
2. Open state: reject requests due to service failure.
3. Half-open: allow limited requests to test recovery.
4. Closed: normal operation, requests flow.
5. Implement graceful degradation when open.
6. Alert on state changes.

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

### Q96: How do I implement strangler pattern for gradual migration?

1. Implement new system (CockroachDB) alongside legacy system.
2. Gradually redirect traffic to new system.
3. Maintain legacy system as fallback.
4. Implement bidirectional synchronization during transition.
5. Decommission legacy system once migration complete.
6. Test fallback scenarios.

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

### Q97: How do I implement retry logic with exponential backoff?

1. Retry failed operations with exponential backoff: 1s, 2s, 4s, 8s.
2. Add jitter to prevent thundering herd.
3. Set maximum retry count.
4. Implement circuit breaker.
5. Log retry attempts.
6. Test under various failures.

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

### Q98: How do I implement graceful shutdown procedures?

1. Stop accepting new connections.
2. Wait for in-flight transactions to complete.
3. Close database connections cleanly.
4. Save in-memory state if needed.
5. Implement timeout to force shutdown.
6. Monitor shutdown completion.

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

### Q99: How do I implement health check endpoints for orchestration?

1. Implement HTTP /health endpoint.
2. Health check queries database for connectivity.
3. Return HTTP status: 200 healthy, 500 unhealthy.
4. Include detailed health info in response.
5. Use for load balancer routing decisions.
6. Monitor health check latency.

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

### Q100: How do I implement predictable latency SLOs?

1. Define SLO targets: P99 < 100ms, P50 < 10ms.
2. Monitor against targets continuously.
3. Implement circuit breakers when SLO violated.
4. Identify and fix violating queries.
5. Capacity plan for peak load.
6. Document trade-offs with cost/features.

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

---

## SECTION 18: DATA CONSISTENCY AND VALIDATION

### Q101: How do I implement application-driven consistency verification?

1. Periodically verify data consistency between primary and replicas.
2. Implement checksum queries.
3. Compare checksums across replicas.
4. Alert if checksums differ.
5. Use REPAIR procedures if corruption detected.
6. Implement as operational procedure.

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

### Q102: How do I handle foreign key constraints with eventual consistency?

1. CockroachDB enforces FK constraints synchronously.
2. For distributed systems, implement in application.
3. Use soft foreign keys (documented without constraints).
4. Implement application-level consistency checks.
5. Use background jobs for reconciliation.
6. Document consistency model.

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

### Q103: How do I implement distributed locks for inter-cluster coordination?

1. CockroachDB supports row-level locking.
2. Use FOR UPDATE clause for explicit locking.
3. Create lock tables for coordination.
4. Acquire lock within transaction.
5. Implement timeout logic.
6. Monitor lock wait times.

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

### Q104: How do I implement row-level security policies?

1. CockroachDB lacks native RLS.
2. Implement application-level filtering.
3. Create security views showing only allowed rows.
4. Grant SELECT to views instead of tables.
5. Implement audit triggers.
6. Document RLS implementation.

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

### Q105: How do I handle long-running transactions impacting cluster?

1. Long-running transactions block MVCC cleanup.
2. Monitor transaction duration via SHOW TRANSACTIONS.
3. Kill long transactions if necessary: CANCEL SESSION.
4. Set session timeout limits.
5. Implement application logic to break into smaller transactions.
6. Alert on exceeding threshold duration.

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

### Q106: How do I implement multi-tenant isolation in shared cluster?

1. Separate databases per tenant.
2. Implement authentication at application layer.
3. Create tenant-specific credentials with restricted privileges.
4. Use row-level filtering for data isolation.
5. Monitor isolation; verify no data leakage.
6. Implement backup/recovery per tenant.

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

### Q107: How do I handle hybrid SQL and NoSQL patterns?

1. CockroachDB is SQL-only.
2. Use JSON columns for semi-structured data.
3. Use JSON functions for querying.
4. Design schema for both structured/flexible data.
5. Index JSON fields for efficiency.
6. Document hybrid schema design.

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

### Q108: How do I implement feature flags tied to database state?

1. Store feature flags in database table.
2. Query flags on startup.
3. Implement caching to avoid repeated queries.
4. Use transactions for consistent flag state.
5. Monitor flag changes; alert on critical changes.
6. Document flag meanings and impact.

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

### Q109: How do I implement time-series data optimization?

1. Time-series data is append-only with time-based queries.
2. Partition by time: PARTITION BY RANGE (timestamp).
3. Use consistent hash sharding for even distribution.
4. Archive old partitions to reduce active storage.
5. Create indexes on timestamp column.
6. Monitor partition growth; split large partitions.

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

### Q110: How do I implement cost allocation for multi-tenant cluster?

1. Track resource usage per tenant: queries, storage, replication.
2. Implement resource tagging.
3. Monitor cost trends; alert on unexpected increases.
4. Implement tenant-specific resource limits.
5. Report costs to tenants for billing.
6. Optimize queries to reduce per-tenant costs.

```sql
CREATE TABLE tenant_costs (
  tenant_id UUID,
  month DATE,
  query_count BIGINT,
  bytes_processed BIGINT,
  execution_seconds DECIMAL,
  estimated_cost DECIMAL,
  created_at TIMESTAMP DEFAULT now()
);

-- Calculate monthly costs
INSERT INTO tenant_costs (tenant_id, month, query_count, bytes_processed, total_cost)
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

---

## SECTION 19: PRODUCTION TROUBLESHOOTING DEEP DIVE

### Q111: How do I diagnose slow replication and identify lag causes?

1. Monitor replication metrics: ranges.rebalancing.writes, raft.process.append.entries.
2. High lag indicates network issues or slow followers.
3. Check network latency between nodes.
4. Monitor follower CPU and disk I/O for bottlenecks.
5. Identify and fix root cause.
6. Monitor post-fix for return to normal.

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

---

## SECTION 20: PRODUCTION INCIDENT RESPONSE

### Q112: Case Study: Recovering from cascading failure due to quorum loss

**Scenario:** All nodes in primary region went offline simultaneously during maintenance window.

**Root Cause Analysis:**
1. Maintenance activity caused all nodes in region to restart.
2. Quorum required 2 out of 3 nodes, both in primary region.
3. Standby region had single node (not enough for quorum).
4. System became read-only immediately.

**Recovery Steps:**
1. Identified primary region completely offline.
2. Promoted standby region node to primary (no quorum available).
3. Restored quorum by bringing primary region nodes back online.
4. Allowed automatic re-synchronization between regions.
5. Verified data consistency and application operations.
6. Implemented improvements to prevent recurrence.

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

**Prevention Implementation:**
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

---

## SECTION 21: COMPLIANCE AND DATA GOVERNANCE

### Q113: How do I implement compliance with GDPR data residency requirements?

1. Use locality configuration to enforce data within specific regions: --locality=region=eu
2. Verify replica placement: ALTER TABLE CONFIGURE ZONE USING constraints='[+region=eu]'
3. Monitor cross-region movement; alert if data leaves allowed regions.
4. Implement backup location restrictions to comply with rules.
5. Document data residency architecture and verification procedures.
6. Perform quarterly compliance audits.

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

---

## SECTION 22: ADVANCED OBSERVABILITY AND METRICS

### Q114: How do I implement custom dashboards for cluster health visualization?

1. Query key metrics from crdb_internal tables.
2. Build dashboards in Grafana with real-time updates.
3. Create different dashboards for operators, developers, executives.
4. Implement drill-down capabilities to root cause issues.
5. Set dashboard refresh rates based on importance.
6. Document dashboard KPIs and thresholds.

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
```

---

## SECTION 23: ADVANCED OPERATIONAL PROCEDURES

### Q115: How do I implement rolling zero-downtime updates?

1. Plan update sequence for odd-numbered nodes.
2. Update one node at a time to maintain quorum.
3. Verify node health after each update.
4. Monitor cluster performance during update.
5. Rollback procedure if issues detected.
6. Document update completion.

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

---

## SECTION 24: ADVANCED TROUBLESHOOTING TECHNIQUES

### Q116: How do I handle clock skew and time synchronization issues?

1. Monitor NTP synchronization across all nodes.
2. Set max-offset for acceptable clock divergence: --max-offset=500ms
3. Verify all nodes synchronize via ntpstat command.
4. Monitor for clock jump warnings in logs.
5. Restart nodes if clock skew exceeds threshold.
6. Implement alerting for NTP synchronization failures.

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

---

## SECTION 25: PRODUCTION HARDENING CHECKLIST

### Q117: What is the complete production readiness checklist?

**PHASE 1: PRE-DEPLOYMENT (Week 1-2)**

INFRASTRUCTURE:
- [ ] Network architecture finalized
- [ ] VPC/subnets configured
- [ ] Firewall rules implemented (port 26257, 8080)
- [ ] Load balancer provisioned and configured
- [ ] DNS entries created
- [ ] SSL/TLS certificates generated
- [ ] Hardware sizing approved (CPU, memory, storage)
- [ ] Storage backend configured (SSD/NVMe)

SECURITY:
- [ ] TLS certificates created and distributed
- [ ] SSH keys generated and secured
- [ ] Initial admin user created
- [ ] RBAC roles defined and documented
- [ ] Audit logging enabled
- [ ] Network segmentation implemented
- [ ] Encryption at rest configured
- [ ] Key management plan documented

MONITORING & ALERTING:
- [ ] Prometheus installed and configured
- [ ] Grafana dashboards created
- [ ] Alert rules defined and tested
- [ ] Logging infrastructure ready
- [ ] PagerDuty/incident management integration
- [ ] Health check endpoints configured
- [ ] Performance baseline established

BACKUP & DISASTER RECOVERY:
- [ ] Backup destination configured (S3/GCS)
- [ ] Backup credentials stored securely
- [ ] Restore procedure tested
- [ ] RTO/RPO targets documented
- [ ] Failover procedures documented
- [ ] Cross-region backup copies configured
- [ ] Backup retention policy defined

**PHASE 2: DEPLOYMENT (Week 3)**

CLUSTER INITIALIZATION:
- [ ] First node started successfully
- [ ] Cluster initialization completed
- [ ] Remaining nodes joined
- [ ] All 3+ nodes showing as LIVE
- [ ] Replication factor verified
- [ ] Zone configuration applied
- [ ] Cluster version updated to latest

DATABASE SETUP:
- [ ] System tables verified
- [ ] Initial databases created
- [ ] Schemas deployed
- [ ] Indexes created and validated
- [ ] Users and roles created
- [ ] Permissions verified and tested
- [ ] Application connection tested

VALIDATION:
- [ ] SQL connectivity test passed
- [ ] Basic queries execute correctly
- [ ] Transactions working properly
- [ ] Replication functioning
- [ ] Backup/restore tested
- [ ] Failover procedures tested
- [ ] Performance benchmarks validated

**PHASE 3: PRODUCTION HARDENING (Week 4)**

SECURITY HARDENING:
- [ ] Password policies enforced
- [ ] Audit logging verified active
- [ ] Network access verified restricted
- [ ] Certificate validation enabled
- [ ] Certificate expiration monitoring
- [ ] DDoS protection enabled
- [ ] Rate limiting configured
- [ ] Access logs reviewed

PERFORMANCE TUNING:
- [ ] Cache sizes optimized
- [ ] SQL memory limits set
- [ ] Connection pooling configured
- [ ] Slow query logging enabled
- [ ] Index performance analyzed
- [ ] Query plans reviewed
- [ ] Statistics updated

OPERATIONAL SETUP:
- [ ] On-call rotation established
- [ ] Runbooks documented and available
- [ ] Incident response procedures tested
- [ ] Change management process implemented
- [ ] Deployment procedure documented
- [ ] Rollback procedures documented
- [ ] Team trained on operations

MONITORING VALIDATION:
- [ ] All metrics being collected
- [ ] Alerts functioning correctly
- [ ] Dashboards displaying data
- [ ] Log aggregation working
- [ ] Alerting escalation tested
- [ ] Health checks passing
- [ ] Performance within targets

**PHASE 4: GO-LIVE (Week 5)**

FINAL VERIFICATION:
- [ ] All checklists completed
- [ ] Final security scan passed
- [ ] Load testing results acceptable
- [ ] Backup/restore procedures validated
- [ ] Disaster recovery procedures tested
- [ ] Team readiness verified
- [ ] Communication plan ready

MIGRATION PREPARATION:
- [ ] Data migration plan finalized
- [ ] Migration script tested
- [ ] Rollback plan documented
- [ ] Cutover window scheduled
- [ ] Communication to stakeholders sent
- [ ] Monitoring escalation procedures ready
- [ ] War room setup configured

GO-LIVE EXECUTION:
- [ ] Data migration executed
- [ ] Validation queries passed
- [ ] Application connectivity verified
- [ ] Load gradually increased
- [ ] Monitoring watched closely
- [ ] No critical alerts triggered
- [ ] Performance within SLO targets

**POST-GO-LIVE (Week 6+)**

STABILIZATION:
- [ ] Monitor cluster 24/7 for first week
- [ ] Log analysis for errors
- [ ] Performance validation
- [ ] No unexpected restarts
- [ ] Backup/restore procedures validated
- [ ] Team comfortable with operations
- [ ] Process improvements identified

DOCUMENTATION:
- [ ] As-built documentation updated
- [ ] Operational procedures finalized
- [ ] Troubleshooting guide completed
- [ ] Knowledge transfer completed
- [ ] Runbooks updated with learnings
- [ ] Change log maintained
- [ ] Architecture diagram updated

OPTIMIZATION:
- [ ] Fine-tune cluster settings
- [ ] Optimize indexes based on actual workload
- [ ] Review and adjust alert thresholds
- [ ] Implement additional automation
- [ ] Optimize backup strategy
- [ ] Plan capacity for growth
- [ ] Schedule next review

---

## SECTION 26: ADVANCED DATABASE DESIGN PATTERNS

### Q118: How do I design schemas for multi-tenancy with data isolation?

1. Create separate databases per tenant for complete isolation.
2. Use tenant_id column in all tables for application-level filtering.
3. Create unique indexes on (tenant_id, business_key) for data integrity.
4. Implement views filtering by tenant_id for security.
5. Use row-level security triggers to enforce tenant boundaries.
6. Test data isolation regularly to prevent leakage.

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

---

## SECTION 27: ADVANCED QUERY OPTIMIZATION

### Q119: How do I optimize queries with complex joins?

1. Identify join selectivity: which predicates reduce rows most.
2. Reorder joins to process most selective predicates first.
3. Add indexes on join columns for efficient lookups.
4. Use EXPLAIN ANALYZE to verify join order.
5. Consider denormalizing highly joined tables.
6. Break complex queries into smaller CTEs if optimizer struggles.

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
```

---

## SECTION 28: ADVANCED SCALING STRATEGIES

### Q120: How do I implement horizontal scaling with application-level sharding?

1. Choose sharding key (customer_id, tenant_id, region).
2. Implement consistent hashing for shard distribution.
3. Create metadata service to track shard locations.
4. Handle cross-shard queries with fan-out and merge.
5. Implement shard rebalancing for growth.
6. Test shard routing thoroughly.

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

# Usage
shard_manager = ShardManager(num_shards=16)

# Single-shard query (fast)
customer_id = 123
conn = shard_manager.get_connection(customer_id)
orders = conn.execute("SELECT * FROM orders WHERE customer_id = ?", customer_id)

# Multi-shard query (slower, fan-out)
all_orders = shard_manager.query_all_shards("SELECT COUNT(*) as total FROM orders")
```

---

## SECTION 29: ADVANCED ENTERPRISE FEATURES

### Q121: How do I implement multi-tenancy with complete data isolation?

1. Separate databases per tenant for complete isolation.
2. Separate backup/restore per tenant.
3. Implement audit trails per tenant.
4. Implement resource quotas per tenant.
5. Monitor cross-tenant performance issues.
6. Test isolation regularly.

---

## SECTION 30: PRODUCTION INCIDENT CASE STUDIES

### Q122: Case Study: Scaling from single-region to multi-region production (100K+ QPS)

**Scenario:** E-commerce platform growing from single US region to global presence.

**Challenge:**
- Initial: Single region (3 nodes), handling 10K QPS
- Target: Multi-region (9 nodes across 3 regions), handle 100K QPS
- Constraint: Zero downtime during migration

**Solution Architecture:**

```
```
Phase 1 (Week 1-2): Prepare Infrastructure
├─ Provision nodes in EU and APAC regions
├─ Configure networking (VPC peering, firewalls)
├─ Set up inter-region replication
└─ Configure backup copies in each region

Phase 2 (Week 3): Deploy Multi-Region Cluster
├─ Add nodes with locality awareness
├─ Start replication to new regions
├─ Monitor replication lag
└─ Verify data consistency

Phase 3 (Week 4): Optimize Data Placement
├─ Configure zone constraints per table
├─ Implement read-local preferences
├─ Monitor query latency improvements
└─ Adjust replica placement

Phase 4 (Week 5): Cutover Application Traffic
├─ Route read traffic to local replicas
├─ Maintain write routing to primary region
├─ Monitor performance by region
└─ Validate application behavior
```

**Implementation Details:**

```sql
-- Phase 1: Configure zones with multi-region constraints
ALTER TABLE users CONFIGURE ZONE USING 
  num_replicas = 5,
  constraints = '[+region=us-east, +region=eu-west, +region=ap-south]',
  lease_preferences = '[[+region=us-east]]';

-- Phase 2: Enable follower reads for read scaling
SET SESSION enable_follower_reads = true;

-- Phase 3: Monitor replication across regions
SELECT 
  range_id,
  replica_node_id,
  crdb_internal.get_node_locality(replica_node_id) as locality,
  is_leader
FROM crdb_internal.ranges_no_leases
WHERE table_name = 'users'
ORDER BY range_id;

-- Phase 4: Implement read-local routing in application
-- Route SELECT queries to local region replicas
-- Route INSERT/UPDATE/DELETE to primary region

-- Monitor regional performance
SELECT 
  region,
  COUNT(*) as query_count,
  AVG(latency_ms) as avg_latency,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99_latency
FROM query_metrics
WHERE timestamp > now() - interval '1 hour'
GROUP BY region
ORDER BY region;
```

**Results:**
- QPS scaling from 10K to 100K+
- Query latency reduced 50% with local reads
- Zero downtime during migration
- Data consistency maintained across regions
- Backup redundancy achieved

---

## SECTION 31: ADVANCED MONITORING AND OBSERVABILITY

### Q123: How do I implement distributed tracing for query debugging?

1. Enable distributed tracing in application.
2. Propagate trace IDs across services.
3. Collect traces from database and application.
4. Visualize trace waterfall for latency analysis.
5. Correlate traces with database metrics.
6. Identify bottlenecks in trace paths.

```go
// Distributed tracing example with Jaeger
import (
    "github.com/opentracing/opentracing-go"
    jaeger "github.com/uber/jaeger-client-go"
)

func initTracer() opentracing.Tracer {
    cfg := jaeger.Config{
        ServiceName: "crdb-app",
        Sampler: &jaeger.Config_SamplerConfig{
            Type: "const",
            Param: 1,
        },
        Reporter: &jaeger.Config_ReporterLogic{
            Endpoint: "http://jaeger:14268/api/traces",
        },
    }
    
    tracer, _ := cfg.NewTracer()
    return tracer
}

func queryDatabase(tracer opentracing.Tracer, sql string) {
    span := tracer.StartSpan("database.query")
    defer span.Finish()
    
    span.SetTag("sql", sql)
    span.SetTag("db.type", "cockroachdb")
    
    // Execute query
    // Span automatically recorded with duration
}

// Application traces
func handleRequest(tracer opentracing.Tracer, req *http.Request) {
    span := tracer.StartSpan("http.request")
    defer span.Finish()
    
    queryDatabase(tracer, "SELECT * FROM users")
    queryDatabase(tracer, "SELECT * FROM orders")
}
```

### Q124: How do I implement custom metrics for application-specific monitoring?

1. Define custom metrics for business KPIs.
2. Expose metrics endpoint for Prometheus scraping.
3. Graph custom metrics on operational dashboards.
4. Alert on custom metric thresholds.
5. Correlate custom metrics with system metrics.
6. Use metrics for capacity planning.

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Define custom metrics
order_count = Counter('orders_total', 'Total orders processed', ['region', 'status'])
order_processing_time = Histogram('order_processing_seconds', 'Time to process order', buckets=(1, 5, 10, 30))
active_shopping_carts = Gauge('shopping_carts_active', 'Active shopping carts', ['region'])
revenue = Counter('revenue_total', 'Total revenue', ['product_category', 'region'])

# Record metrics in application
def process_order(order_data, region):
    start_time = time.time()
    
    try:
        # Process order
        db.execute("INSERT INTO orders VALUES (?)", order_data)
        
        # Record success metric
        order_count.labels(region=region, status='success').inc()
        revenue.labels(
            product_category=order_data['category'],
            region=region
        ).inc(order_data['amount'])
        
    except Exception as e:
        order_count.labels(region=region, status='error').inc()
        raise
    
    finally:
        # Record timing metric
        elapsed = time.time() - start_time
        order_processing_time.observe(elapsed)

# Update gauge
active_shopping_carts.labels(region='us-east').set(1234)

# Expose metrics endpoint
from prometheus_client import start_http_server
start_http_server(8000)  # Prometheus scrapes :8000/metrics
```

---

## SECTION 32: MULTI-CLOUD AND HYBRID DEPLOYMENT

### Q125: How do I deploy CockroachDB across AWS and GCP?

1. Set up VPC peering between AWS VPC and GCP VPC.
2. Start nodes with locality: --locality=cloud=aws,region=us-east or --locality=cloud=gcp,region=us-central
3. Configure firewall rules allowing cross-cloud traffic (port 26257).
4. Enable inter-cloud replication.
5. Configure backup destination in each cloud.
6. Monitor cross-cloud latency and network costs.

```bash
# AWS Node
cockroach start \
  --locality=cloud=aws,region=us-east,zone=az1 \
  --listen-addr=aws-node1.example.com:26257 \
  --advertise-addr=aws-node1.example.com:26257 \
  --join=gcp-node1.example.com:26257 \
  --certs-dir=certs

# GCP Node
cockroach start \
  --locality=cloud=gcp,region=us-central,zone=az1 \
  --listen-addr=gcp-node1.example.com:26257 \
  --advertise-addr=gcp-node1.example.com:26257 \
  --join=aws-node1.example.com:26257 \
  --certs-dir=certs

# Configure zone for cloud distribution
cockroach sql --certs-dir=certs <<EOF
ALTER TABLE critical_data CONFIGURE ZONE USING 
  num_replicas = 5,
  constraints = '[+cloud=aws, +cloud=gcp]';
EOF

# Monitor cross-cloud traffic
SELECT source_node_id, dest_node_id, packets_sent
FROM crdb_internal.node_metrics
WHERE crdb_internal.get_node_locality(source_node_id) -> 'cloud' !=
      crdb_internal.get_node_locality(dest_node_id) -> 'cloud';
```

---

## SECTION 33: ADVANCED BENCHMARKING AND PERFORMANCE TESTING

### Q126: How do I perform comprehensive load testing before production launch?

1. Set up test cluster identical to production.
2. Use workload generators (TPC-C, TPC-H).
3. Simulate peak load for sustained period.
4. Monitor system behavior under load.
5. Identify bottlenecks and optimize.
6. Document findings and capacity limits.

```bash
# TPC-C workload (OLTP benchmark)
cockroach workload init tpcc \
  --warehouses=100 \
  "postgresql://root@localhost:26257/tpcc?sslmode=disable"

cockroach workload run tpcc \
  --warehouses=100 \
  --duration=30m \
  --max-rate=10000 \
  "postgresql://root@localhost:26257/tpcc?sslmode=disable"

# TPC-H workload (OLAP benchmark)
cockroach workload init tpch \
  --scale-factor=100 \
  "postgresql://root@localhost:26257/tpch?sslmode=disable"

cockroach workload run tpch \
  --duration=30m \
  "postgresql://root@localhost:26257/tpch?sslmode=disable"

# Monitor during test
cockroach sql --certs-dir=certs <<EOF
SELECT 
  node_id,
  (uptime_seconds - lag(uptime_seconds) OVER (ORDER BY collected_at)) as cpu_seconds,
  (bytes_read - lag(bytes_read) OVER (ORDER BY collected_at)) as bytes_read_delta
FROM crdb_internal.node_metrics
WHERE collected_at > now() - interval '30 minutes'
ORDER BY collected_at DESC;
EOF
```

---

## SECTION 34: ADVANCED APPLICATION INTEGRATION PATTERNS

### Q127: How do I implement optimistic locking for concurrent updates?

1. Add version column to table.
2. Increment version on each update.
3. Check version in WHERE clause to detect conflicts.
4. Application retries on conflict.
5. Monitor conflict rates.
6. Adjust retry strategy based on conflict patterns.

```sql
-- Table with optimistic locking
CREATE TABLE inventory (
  product_id INT PRIMARY KEY,
  quantity INT,
  version INT DEFAULT 1
);

-- Update with version check
UPDATE inventory
SET quantity = quantity - ?,
    version = version + 1
WHERE product_id = ? AND version = ?;

-- Check for conflict
SELECT CHANGES() as rows_affected;
-- If rows_affected = 0, version conflict detected; retry with new version

-- Application-level retry logic
def update_inventory(product_id, quantity_decrease, retry_count=3):
    for attempt in range(retry_count):
        # Get current version
        current = db.query_one(
            "SELECT quantity, version FROM inventory WHERE product_id = ?",
            product_id
        )
        
        # Attempt update with version check
        result = db.execute(
            "UPDATE inventory SET quantity = ?, version = version + 1 "
            "WHERE product_id = ? AND version = ?",
            current['quantity'] - quantity_decrease,
            product_id,
            current['version']
        )
        
        if result.rows_affected > 0:
            return True  # Success
        
        # Conflict detected, retry
        time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
    
    raise UpdateConflictError("Failed to update after retries")
```

---

## SECTION 35: PRODUCTION READINESS AND OPERATIONAL EXCELLENCE

### Q128: How do I implement continuous compliance validation?

1. Schedule automated compliance checks.
2. Verify data residency constraints.
3. Validate encryption status.
4. Check backup configurations.
5. Audit user access logs.
6. Generate compliance reports.

```sql
-- Automated compliance checks
CREATE TABLE compliance_checks (
  check_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  check_name VARCHAR,
  check_result BOOLEAN,
  check_details JSONB,
  checked_at TIMESTAMP DEFAULT now()
);

-- Data residency verification
INSERT INTO compliance_checks (check_name, check_result, check_details)
SELECT 
  'Data Residency EU' as check_name,
  (SELECT COUNT(*) = 0 FROM crdb_internal.ranges_no_leases 
    WHERE table_name = 'sensitive_data' 
    AND crdb_internal.get_node_locality(replica_node_id) -> 'region' != 'eu') as check_result,
  jsonb_build_object(
    'table', 'sensitive_data',
    'expected_region', 'eu',
    'checked_at', now()
  ) as check_details;

-- Encryption verification
INSERT INTO compliance_checks (check_name, check_result, check_details)
SELECT 
  'Encryption at Rest' as check_name,
  (SELECT (SELECT setting FROM system.settings WHERE name = 'server.encryption_at_rest.enabled') = 'true') as check_result,
  jsonb_build_object(
    'setting', 'server.encryption_at_rest.enabled',
    'checked_at', now()
  ) as check_details;

-- Backup configuration verification
INSERT INTO compliance_checks (check_name, check_result, check_details)
SELECT 
  'Daily Backups Configured' as check_name,
  (SELECT COUNT(*) > 0 FROM system.scheduled_jobs WHERE schedule_name LIKE '%backup%') as check_result,
  jsonb_build_object(
    'backup_count', (SELECT COUNT(*) FROM system.scheduled_jobs WHERE schedule_name LIKE '%backup%'),
    'checked_at', now()
  ) as check_details;

-- Generate compliance report
SELECT 
  check_name,
  check_result,
  check_details,
  checked_at
FROM compliance_checks
WHERE checked_at > now() - interval '1 day'
ORDER BY check_name;
```

---

## SECTION 36: FINAL PRODUCTION BEST PRACTICES

### Q129: How do I document operational procedures for knowledge transfer?

1. Create runbooks for common tasks (scale, backup, failover).
2. Document troubleshooting decision trees.
3. Record video walkthroughs for complex procedures.
4. Maintain wiki with cluster topology and configuration.
5. Keep disaster recovery procedures updated.
6. Conduct quarterly knowledge transfer sessions.

```markdown
# CockroachDB Operations Runbook

## Table of Contents
1. [Cluster Overview](#cluster-overview)
2. [Scaling Operations](#scaling-operations)
3. [Backup and Recovery](#backup-and-recovery)
4. [Incident Response](#incident-response)

## Cluster Overview
Production cluster: 9 nodes across 3 regions (US-East, EU-West, AP-South)
- Each region: 3 nodes
- Replication factor: 5
- Backup: Daily to S3

## Scaling Operations

### Adding a Node
1. Provision infrastructure (VM, networking, storage)
2. Generate TLS certificate: cockroach cert create-node --certs-dir=certs node_name
3. Start node: cockroach start --join=existing_node:26257 --certs-dir=certs
4. Verify in DB Console: All ranges should show as HEALTHY
5. Monitor rebalancing: ranges.rebalancing.writes should decrease to zero

### Removing a Node
1. Drain node: cockroach node drain --host=node_name --certs-dir=certs
2. Stop process: cockroach quit --host=node_name --certs-dir=certs
3. Verify all ranges moved: SELECT COUNT(*) FROM crdb_internal.ranges WHERE store_id = node_store_id
4. Remove from infrastructure

## Backup and Recovery

### Full Backup
```
BACKUP INTO 's3://backup-bucket/backup-date/' WITH revision_history;
```

### Point-in-Time Recovery
```
RESTORE FROM 'gs://backup-bucket/backup' AS OF SYSTEM TIME '2024-01-15 14:30:00';
```

## Incident Response

### Cluster Unavailable
1. Check node status: SELECT node_id, is_live FROM crdb_internal.nodes;
2. Verify network connectivity: ping node_addresses
3. Check disk space: df -h
4. Review logs for errors
5. Initiate restore if needed

### High Latency
1. Check active queries: SELECT * FROM crdb_internal.active_transactions;
2. Identify slow queries: SELECT * FROM crdb_internal.node_statement_statistics ORDER BY latency_p99 DESC;
3. Kill long transactions if necessary: CANCEL SESSION 'session_id';
4. Analyze query plans: EXPLAIN ANALYZE SELECT ...;
5. Add indexes if needed
```

### Q130: How do I create effective SLOs and error budgets?

1. Define critical user journeys.
2. Measure latency and error rates for each journey.
3. Set SLO targets: P99 < 100ms, error rate < 0.1%.
4. Track error budget consumption over time.
5. Alert when error budget depleted.
6. Use error budget to guide prioritization.

```python
# SLO tracking and error budget calculation
class SLOTracker:
    def __init__(self, slo_target=0.999, time_window_days=30):
        self.slo_target = slo_target  # 99.9% availability
        self.time_window_seconds = time_window_days * 24 * 3600
        self.allowed_error_budget = (1 - slo_target) * self.time_window_seconds
        self.errors_observed = 0
        self.total_requests = 0
    
    def record_request(self, success):
        self.total_requests += 1
        if not success:
            self.errors_observed += 1
    
    def get_error_budget_remaining(self):
        """Calculate remaining error budget in seconds"""
        error_budget_seconds_consumed = (
            self.errors_observed / self.total_requests * self.time_window_seconds
            if self.total_requests > 0 else 0
        )
        return max(0, self.allowed_error_budget - error_budget_seconds_consumed)
    
    def get_error_budget_percentage(self):
        """Get remaining error budget as percentage"""
        remaining = self.get_error_budget_remaining()
        return (remaining / self.allowed_error_budget) * 100 if self.allowed_error_budget > 0 else 0
    
    def should_accept_risk(self):
        """Determine if safe to deploy changes based on error budget"""
        return self.get_error_budget_percentage() > 20  # Keep 20% buffer

# Usage
slo_tracker = SLOTracker(slo_target=0.999)

for request in incoming_requests:
    try:
        result = execute_query()
        slo_tracker.record_request(success=True)
    except Exception as e:
        slo_tracker.record_request(success=False)
        logging.error(f"Request failed: {e}")

# Before deployment
if slo_tracker.should_accept_risk():
    print("Safe to deploy changes")
else:
    print("Error budget depleted; skip non-critical deployments")
    print(f"Error budget remaining: {slo_tracker.get_error_budget_percentage():.1f}%")
```

---

## SECTION 37: EDGE CASES AND CORNER SCENARIOS

### Q131: How do I handle very large transactions that may exceed available memory?

1. Break into smaller transactions.
2. Monitor SQL memory usage.
3. Increase max-sql-memory if appropriate.
4. Use streaming inserts for bulk loads.
5. Test in staging with production-size data.
6. Implement streaming API for large result sets.

```sql
-- Split large transaction into batches
DO $$
DECLARE
  v_batch_size INT := 10000;
  v_total_rows INT := (SELECT COUNT(*) FROM source_table);
  v_processed INT := 0;
BEGIN
  WHILE v_processed < v_total_rows LOOP
    BEGIN
      INSERT INTO target_table
      SELECT * FROM source_table
      LIMIT v_batch_size OFFSET v_processed;
      
      v_processed := v_processed + v_batch_size;
      RAISE NOTICE 'Processed: %/%', v_processed, v_total_rows;
    EXCEPTION WHEN OTHERS THEN
      RAISE NOTICE 'Error at offset %: %', v_processed, SQLERRM;
      -- Retry logic or skip batch
    END;
  END LOOP;
END$$;

-- Monitor memory usage
SELECT session_id, memory_used_bytes, rows_read
FROM crdb_internal.active_transactions
WHERE memory_used_bytes > 1000000000;  -- > 1GB
```

### Q132: How do I handle extremely high cardinality columns in indexes?

1. Avoid indexing highly unique columns (UUIDs with thousands of values per second).
2. Use filtered indexes for specific value ranges.
3. Partition data to reduce index size.
4. Monitor index bloat.
5. Consider alternative designs.
6. Test index effectiveness before deployment.

```sql
-- Filtered index on specific values
CREATE INDEX idx_status_active ON orders(customer_id)
WHERE status = 'active';

-- Partial index for date ranges
CREATE INDEX idx_recent_orders ON orders(customer_id)
WHERE created_at > now() - interval '30 days';

-- Skip indexing high-cardinality columns
-- Bad: CREATE INDEX idx_request_id ON logs(request_id);  -- Millions of unique values
-- Good: CREATE INDEX idx_user_timestamp ON logs(user_id, timestamp);  -- More selective
```

### Q133: How do I handle clock reset scenarios in distributed clusters?

1. Monitor NTP status on all nodes.
2. Implement alerts for clock jumps.
3. Ensure system clocks synchronized within max-offset.
4. Restart nodes if clock skew exceeds threshold.
5. Monitor for data anomalies post-restart.
6. Document clock reset procedures.

```bash
# Detect clock jumps in logs
grep -i "clock.*jump\|clock.*skew" /path/to/cockroach.log

# Monitor NTP across cluster
for node in node1 node2 node3; do
  echo "=== $node NTP Status ==="
  ssh $node "timedatectl status"
  ssh $node "ntpq -p"
done

# Verify time consistency
cockroach sql --certs-dir=certs <<EOF
SELECT 
  node_id,
  address,
  clock_offset_millis,
  CASE 
    WHEN ABS(clock_offset_millis) > 500 THEN 'WARN: High offset'
    WHEN ABS(clock_offset_millis) > 1000 THEN 'ERROR: Critical offset'
    ELSE 'OK'
  END as status
FROM crdb_internal.node_runtime_info
ORDER BY ABS(clock_offset_millis) DESC;
EOF
```

### Q134: How do I handle gossiping protocol issues in large clusters?

1. Gossip protocol propagates cluster information.
2. Monitor gossip latency and heartbeats.
3. Increase heartbeat interval if many nodes fail to communicate.
4. Check network MTU (1500 bytes default).
5. Verify firewall not blocking gossip (port 26257).
6. Review logs for gossip-related errors.

```sql
-- Monitor gossip health
SELECT 
  node_id,
  COUNT(*) as gossip_count,
  AVG(latency_ms) as avg_gossip_latency
FROM crdb_internal.gossip_metrics
GROUP BY node_id
ORDER BY avg_gossip_latency DESC;

-- Check for gossip failures
SELECT * FROM system.event_log
WHERE event_type LIKE '%gossip%'
ORDER BY timestamp DESC
LIMIT 20;

-- Adjust gossip settings if needed
SET CLUSTER SETTING server.gossip.interval='1s';
SET CLUSTER SETTING server.gossip.check_interval='5m';
```

---

## SECTION 38: SPECIALIZED WORKLOAD PATTERNS

### Q135: How do I handle time-series workloads efficiently?

1. Use partitioned tables by time.
2. Create separate indexes per partition.
3. Archive old partitions to reduce active data.
4. Use consistent batch inserts.
5. Implement downsampling for long-term storage.
6. Monitor partition growth.

```sql
-- Time-series optimized schema
CREATE TABLE metrics (
  timestamp TIMESTAMP NOT NULL,
  metric_name VARCHAR NOT NULL,
  labels JSONB,
  value DECIMAL,
  PRIMARY KEY (timestamp DESC, metric_name)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01')
);

-- High-velocity inserts
INSERT INTO metrics (timestamp, metric_name, value)
SELECT now(), name, value
FROM (VALUES
  ('cpu_usage', 75.5),
  ('memory_usage', 82.3),
  ('disk_io', 45.2)
) AS v(name, value);

-- Downsampling for long-term storage
INSERT INTO metrics_hourly
SELECT 
  DATE_TRUNC('hour', timestamp) as hour,
  metric_name,
  AVG(value) as avg_value,
  MAX(value) as max_value,
  MIN(value) as min_value
FROM metrics
WHERE timestamp < now() - interval '7 days'
GROUP BY DATE_TRUNC('hour', timestamp), metric_name;

-- Archive old partitions
ALTER TABLE metrics DROP PARTITION p_2023_12;
```

### Q136: How do I optimize for analytics workloads vs transactional workloads?

1. Analytics: batch processes, aggregations, sequential scans.
2. Transactional: point lookups, low-latency reads/writes.
3. Separate databases/clusters if workloads conflict.
4. Use read-only replicas for analytics.
5. Implement query optimization separately per workload.
6. Monitor interference between workload types.

```sql
-- Analytics-optimized query with aggressive parallelism
SET CLUSTER SETTING sql.distsql.plan_distribution_mode='distSQL';

SELECT 
  customer_segment,
  product_category,
  SUM(amount) as total_sales,
  COUNT(*) as order_count,
  AVG(amount) as avg_order_value
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > now() - interval '90 days'
GROUP BY customer_segment, product_category
ORDER BY total_sales DESC;

-- Transactional-optimized query with point lookup
SET CLUSTER SETTING sql.distsql.plan_distribution_mode='local';

SELECT * FROM customer_profile
WHERE customer_id = $1;  -- Index lookup, immediate return

-- Separate read replica for analytics
-- Route analytics queries to read-only replica
-- Route transactional queries to primary cluster
```

### Q137: How do I handle graph-like data structures efficiently?

1. Use adjacency list pattern for graph relationships.
2. Index both directions for traversal efficiency.
3. Implement recursive CTEs for graph traversal.
4. Limit recursion depth to prevent performance issues.
5. Cache frequently accessed paths.
6. Consider specialized graph databases for heavy graph workloads.

```sql
-- Graph schema for social network
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  name VARCHAR
);

CREATE TABLE follows (
  follower_id INT,
  followee_id INT,
  created_at TIMESTAMP DEFAULT now(),
  PRIMARY KEY (follower_id, followee_id),
  FOREIGN KEY (follower_id) REFERENCES users(user_id),
  FOREIGN KEY (followee_id) REFERENCES users(user_id)
);

CREATE INDEX idx_follows_reverse ON follows(followee_id, follower_id);

-- Find friends of friends (depth 2)
WITH RECURSIVE friendship AS (
  SELECT follower_id, followee_id, 1 as depth
  FROM follows
  WHERE follower_id = 123
  
  UNION ALL
  
  SELECT f.follower_id, f.followee_id, depth + 1
  FROM follows f
  JOIN friendship fnd ON f.follower_id = fnd.followee_id
  WHERE depth < 2
)
SELECT DISTINCT u.user_id, u.name, depth
FROM friendship
JOIN users u ON followee_id = u.user_id
WHERE depth <= 2
ORDER BY depth, u.name;

-- Limit recursion depth to prevent runaway queries
SET CLUSTER SETTING sql.max_recursion_depth = 10;
```

---

## SECTION 39: ADVANCED TROUBLESHOOTING SCENARIOS

### Q138: How do I diagnose and fix range fragmentation issues?

1. Ranges should be evenly distributed across cluster.
2. Monitor range count per node.
3. Uneven distribution indicates split/merge issues.
4. Identify and fix constraint violations.
5. Manually trigger range merges if needed.
6. Monitor post-fix for return to normal.

```sql
-- Identify range fragmentation
SELECT 
  node_id,
  COUNT(*) as range_count,
  MIN(range_key_count) as min_keys,
  AVG(range_key_count) as avg_keys,
  MAX(range_key_count) as max_keys
FROM crdb_internal.ranges_no_leases
GROUP BY node_id
ORDER BY range_count DESC;

-- Check for overly small ranges
SELECT range_id, table_name, start_key, end_key, key_count
FROM crdb_internal.ranges_no_leases
WHERE key_count < 1000  -- Very small ranges
ORDER BY key_count;

-- Monitor split activity
SELECT * FROM system.event_log
WHERE event_type = 'range_split'
ORDER BY timestamp DESC
LIMIT 20;

-- Trigger manual range merge if needed (caution: affects performance)
ALTER TABLE fragmented_table SPLIT AT (SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY id) FROM table_data);
```

### Q139: How do I handle runaway query situations?

1. Identify long-running queries through monitoring.
2. Analyze query execution plan.
3. Kill query if necessary: CANCEL SESSION.
4. Optimize or refactor query.
5. Implement query timeouts.
6. Monitor for recurrence.

```sql
-- Identify runaway queries
SELECT 
  session_id,
  user_name,
  query,
  elapsed_time,
  rows_read,
  rows_written
FROM crdb_internal.active_transactions
WHERE elapsed_time > interval '5 minutes'
ORDER BY elapsed_time DESC;

-- Get query plan for optimization
EXPLAIN ANALYZE SELECT ... FROM long_running_query;

-- Kill specific session
CANCEL SESSION 'session_id';

-- Set query timeout to prevent in future
ALTER ROLE app_user SET statement_timeout = '5m';

-- Monitor killed queries
SELECT * FROM system.event_log
WHERE event_type = 'cancel'
ORDER BY timestamp DESC;
```

---

## SECTION 40: ENTERPRISE INTEGRATION AND COMPLIANCE

### Q140: How do I integrate CockroachDB with enterprise SSO (SAML/OAuth)?

1. CockroachDB uses local authentication by default.
2. Implement authentication proxy (e.g., pgBouncer) for SAML.
3. Use application-level authentication with SSO.
4. Map SSO users to database roles.
5. Implement audit logging for access.
6. Test integration thoroughly.

```python
# Application-level SSO integration
from flask import Flask, redirect
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
oauth = OAuth(app)

saml = oauth.register(
    'saml',
    client_id='cockroachdb',
    authorize_url='https://idp.example.com/authorize',
)

@app.route('/login')
def login():
    return redirect(saml.authorize_url(
        redirect_uri='/callback'
    ))

@app.route('/callback')
def callback():
    token = saml.authorize_access_token()
    user_info = token.get('userinfo')
    
    # Map SSO user to database role
    db_user = f"sso_{user_info['sub']}"
    
    # Connect to database as SSO user
    import psycopg2
    conn = psycopg2.connect(
        host='localhost',
        user=db_user,
        password=get_sso_password(user_info['sub']),
        database='app_db'
    )
    
    # Audit logging
    audit_log(f"User {user_info['email']} logged in")
    
    return redirect('/')
```

### Q141: How do I implement data lineage and provenance tracking?

1. Create audit tables for each tracked entity.
2. Log all changes with user, timestamp, old/new values.
3. Implement versioning for schema changes.
4. Track data source/origin for regulatory compliance.
5. Use CDC to populate audit tables.
6. Query audit trail for data lineage.

```sql
-- Audit table for tracking changes
CREATE TABLE customer_audit (
  change_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id INT,
  change_type VARCHAR,  -- INSERT, UPDATE, DELETE
  old_values JSONB,
  new_values JSONB,
  changed_by VARCHAR,
  changed_at TIMESTAMP DEFAULT now()
);

-- Trigger for automatic audit logging
CREATE TRIGGER customer_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW
EXECUTE FUNCTION audit_customer_changes();

-- Audit function
CREATE FUNCTION audit_customer_changes() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO customer_audit (customer_id, change_type, new_values, changed_by)
    VALUES (NEW.id, 'INSERT', row_to_json(NEW), current_user);
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO customer_audit (customer_id, change_type, old_values, new_values, changed_by)
    VALUES (NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO customer_audit (customer_id, change_type, old_values, changed_by)
    VALUES (OLD.id, 'DELETE', row_to_json(OLD), current_user);
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Query data lineage
SELECT 
  change_id,
  customer_id,
  change_type,
  changed_by,
  changed_at,
  old_values,
  new_values
FROM customer_audit
WHERE customer_id = 123
ORDER BY changed_at DESC;
```

---

## SECTION 41: ADVANCED PERFORMANCE TUNING

### Q142: How do I implement connection pooling for optimal performance?

1. Applications should use connection pools.
2. PgBouncer or similar for connection multiplexing.
3. Set pool size to match expected concurrent connections.
4. Monitor pool exhaustion and connection leaks.
5. Implement timeouts to prevent hung connections.
6. Test under peak load.

```bash
# PgBouncer configuration
cat <<EOF > /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=26257 user=root password=secret

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 5
reserve_pool_timeout = 3

statement_lifetime = 0
idle_in_transaction_session_timeout = 600
EOF

# Start PgBouncer
systemctl start pgbouncer

# Application connects to PgBouncer
psql -h localhost -p 6432 -U root mydb
```

### Q143: How do I optimize batch processing for maximum throughput?

1. Use batch inserts instead of individual rows.
2. Disable constraints during bulk load.
3. Use IMPORT for external data.
4. Implement parallel inserts across connections.
5. Monitor insert rate.
6. Validate data integrity after batch.

```sql
-- Batch insert (better than single-row inserts)
INSERT INTO users (name, email, created_at)
VALUES 
  ('User1', 'user1@example.com', now()),
  ('User2', 'user2@example.com', now()),
  ('User3', 'user3@example.com', now()),
  ('User4', 'user4@example.com', now()),
  ('User5', 'user5@example.com', now());

-- Bulk import from file
IMPORT DATA FROM 's3://bucket/data.csv'
INTO users (name, email)
WITH delimiter=',', skip='1';

-- Parallel batch inserts from application
import concurrent.futures
import psycopg2

def insert_batch(batch_data):
    conn = psycopg2.connect('dbname=mydb user=root')
    cursor = conn.cursor()
    cursor.executemany(
        "INSERT INTO users (name, email) VALUES (%s, %s)",
        batch_data
    )
    conn.commit()
    cursor.close()
    conn.close()

# Process batches in parallel
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    batches = chunk_data(all_data, batch_size=1000)
    futures = [executor.submit(insert_batch, batch) for batch in batches]
    concurrent.futures.wait(futures)
```

---

## SECTION 42: FINAL COMPREHENSIVE REFERENCE

### Q144: What is production maturity checklist for team?

**TEAM READINESS:**
- [ ] On-call rotation established
- [ ] Incident response team trained
- [ ] Runbooks reviewed and tested
- [ ] Escalation procedures documented
- [ ] Communication plan for outages
- [ ] Team comfortable with database operations

**MONITORING AND ALERTING:**
- [ ] All critical metrics monitored
- [ ] Alerts tested and verified
- [ ] Alert fatigue managed
- [ ] Incident tracking configured
- [ ] Post-incident review process established
- [ ] Metrics dashboards reviewed

**BACKUP AND RECOVERY:**
- [ ] Backup procedures tested
- [ ] Restore procedures tested and timed
- [ ] Backup retention policies enforced
- [ ] Cross-region backups verified
- [ ] Encryption for backups enabled
- [ ] Recovery time objectives (RTO) met

**SECURITY:**
- [ ] TLS enabled for all connections
- [ ] Certificates valid and managed
- [ ] User access controlled and audited
- [ ] Network segmentation implemented
- [ ] Encryption at rest enabled
- [ ] Security scan completed

**CAPACITY AND SCALABILITY:**
- [ ] Current capacity documented
- [ ] Growth projections made
- [ ] Scaling plan documented
- [ ] Load testing completed
- [ ] Performance baseline established
- [ ] Headroom available for growth

**OPERATIONAL PROCEDURES:**
- [ ] Change management process documented
- [ ] Deployment procedures tested
- [ ] Rollback procedures tested
- [ ] Maintenance windows scheduled
- [ ] Downtime windows communicated
- [ ] Emergency contacts listed

**COMPLIANCE:**
- [ ] Compliance requirements documented
- [ ] Data residency validated
- [ ] Audit logging enabled
- [ ] Data retention policies enforced
- [ ] Compliance audit passed
- [ ] Regulatory requirements met

---

## SECTION 43: DEEP-DIVE ARCHITECTURAL PATTERNS

### Q145: How do I implement OLTP vs OLAP separation for optimal performance?

1. OLTP: operational database, low-latency transactional queries.
2. OLAP: analytical database, optimized for aggregations and scans.
3. Use CDC to sync from OLTP to OLAP.
4. OLTP and OLAP can be separate CockroachDB clusters.
5. Implement ETL pipeline for data transformation.
6. Monitor sync lag and data freshness.

```sql
-- OLTP: Optimized for single-row operations
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT,
  product_id INT,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);

-- OLAP: Denormalized for aggregations
CREATE TABLE order_summary (
  customer_id INT,
  product_id INT,
  order_date DATE,
  total_orders INT,
  total_amount DECIMAL,
  avg_order_value DECIMAL
);

-- CDC pipeline (populate OLAP from OLTP)
CREATE CHANGEFEED FOR TABLE orders
INTO 'kafka://broker:9092/orders-stream'
WITH format='json', resolved='10s';

-- Analytics query on OLAP
SELECT 
  customer_id,
  SUM(total_orders) as lifetime_orders,
  SUM(total_amount) as lifetime_revenue,
  AVG(avg_order_value) as avg_order_value
FROM order_summary
GROUP BY customer_id
ORDER BY lifetime_revenue DESC;
```

---

## SECTION 44: ADVANCED CASE STUDIES AND SOLUTIONS

### Q146: Case Study: Multi-region expansion for fintech platform (zero data loss)

**Scenario:** Financial platform expanding from single region to multi-region for compliance and resilience.

**Challenges:**
- RPO = 0 (zero data loss tolerated)
- RTO < 5 minutes (fast failover required)
- Data residency compliance (specific regions)
- Audit trail integrity
- Consistency guarantees

**Solution Architecture:**

```
Physical cluster replication (primary cluster -> standby cluster)
├─ Primary cluster: US-East (3 nodes)
├─ Standby cluster: EU-West (3 nodes)
├─ Standby cluster: AP-South (3 nodes)
└─ Synchronous replication (zero lag)

Change data capture (audit trail)
├─ All transactions logged to audit table
├─ CDC streams to Kafka
├─ Audit archival to separate backup cluster
└─ Compliance queries on immutable audit data

Failover procedures
├─ Automated health monitoring
├─ 30-second detection of primary failure
├─ Automatic promotion of standby
├─ DNS switchover
└─ Application reconnection
```

**Implementation:**

```sql
-- Enable physical replication (synchronous)
ALTER CLUSTER PRIMARY SET REPLICATION FACTOR = 2;

-- Configure dual-cluster setup
-- Cluster 1 (Primary): transactions flow here
-- Cluster 2 (Standby): receives all changes synchronously

-- Audit table with immutable data
CREATE TABLE transactions (
  transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id INT NOT NULL,
  amount DECIMAL NOT NULL,
  type VARCHAR NOT NULL,
  status VARCHAR DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT now(),
  UNIQUE(account_id, created_at, transaction_id)
);

-- Immutable audit log
CREATE TABLE transaction_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL,
  event_type VARCHAR NOT NULL,
  event_data JSONB NOT NULL,
  recorded_at TIMESTAMP DEFAULT now()
);

-- CDC for compliance archival
CREATE CHANGEFEED FOR TABLE transactions, transaction_audit
INTO 'kafka://broker:9092/audit-stream'
WITH format='json', envelope='upsert';

-- Failover monitoring
CREATE VIEW cluster_health AS
SELECT 
  node_id,
  is_live,
  CASE 
    WHEN is_live THEN 'HEALTHY'
    ELSE 'DOWN'
  END as status
FROM crdb_internal.nodes
ORDER BY node_id;

-- Zero-data-loss verification
SELECT 
  table_name,
  COUNT(*) as total_rows,
  MAX(created_at) as latest_transaction
FROM (
  SELECT 'transactions' as table_name, COUNT(*) as cnt, MAX(created_at) FROM transactions
  UNION ALL
  SELECT 'transaction_audit', COUNT(*), MAX(recorded_at) FROM transaction_audit
)
GROUP BY table_name;
```

**Results:**
- RPO: 0 (synchronous replication)
- RTO: < 2 minutes (automatic failover)
- Data consistency: verified continuously
- Audit trail: complete and tamper-proof
- Compliance: validated by auditors

---

## SECTION 45: OPERATIONAL FRAMEWORKS AND TOOLING

### Q147: How do I implement CI/CD for database schema changes?

1. Version control schema in Git.
2. Automated testing for schema compatibility.
3. Automated migration deployment.
4. Rollback procedures for failed migrations.
5. Coordination with application deployments.
6. Testing in staging before production.

```bash
#!/bin/bash
# Schema migration CI/CD pipeline

# 1. Validate schema changes
psql -h localhost -d test_db -f schema_changes.sql --dry-run
if [ $? -ne 0 ]; then
  echo "Schema validation failed"
  exit 1
fi

# 2. Test on staging
psql -h staging-db -d test_db -f schema_changes.sql
if [ $? -ne 0 ]; then
  echo "Staging migration failed"
  exit 1
fi

# 3. Run compatibility tests
pytest schema_compatibility_tests.py
if [ $? -ne 0 ]; then
  echo "Compatibility tests failed"
  exit 1
fi

# 4. Production deployment with rollback
BACKUP_ID=$(date +%s)
cockroach sql --certs-dir=certs -c "BACKUP INTO 's3://backup/$BACKUP_ID/'"

if cockroach sql --certs-dir=certs -f schema_changes.sql; then
  echo "Production migration successful"
  
  # Run post-migration verification
  pytest post_migration_tests.py
  if [ $? -ne 0 ]; then
    echo "Post-migration tests failed, initiating rollback"
    cockroach sql --certs-dir=certs -c "RESTORE FROM 's3://backup/$BACKUP_ID/'"
    exit 1
  fi
else
  echo "Production migration failed"
  exit 1
fi
```

---

## SECTION 46: FINAL OPERATIONAL EXCELLENCE FRAMEWORK

### Q148: How do I build self-healing database infrastructure?

1. Implement automated recovery procedures.
2. Detect issues before impacting applications.
3. Automatically restart failed components.
4. Self-healing for common failure modes.
5. Escalate to human when needed.
6. Learn from recovered incidents.

```python
# Self-healing database infrastructure
import time
import subprocess
import logging

class SelfHealingDatabaseManager:
    def __init__(self, nodes):
        self.nodes = nodes
        self.logger = logging.getLogger(__name__)
    
    def health_check(self, node):
        """Check if node is healthy"""
        try:
            result = subprocess.run(
                f"cockroach sql --host={node} -c 'SELECT 1'",
                capture_output=True,
                timeout=5
            )
            return result.returncode == 0
        except Exception as e:
            self.logger.error(f"Health check failed for {node}: {e}")
            return False
    
    def restart_node(self, node):
        """Restart failed node"""
        try:
            self.logger.info(f"Restarting node {node}")
            subprocess.run(f"ssh {node} 'systemctl restart cockroachdb'")
            
            # Wait for node to be ready
            for attempt in range(30):
                if self.health_check(node):
                    self.logger.info(f"Node {node} recovered")
                    return True
                time.sleep(1)
            
            self.logger.error(f"Failed to recover node {node}")
            return False
        except Exception as e:
            self.logger.error(f"Restart failed for {node}: {e}")
            return False
    
    def drain_node(self, node):
        """Gracefully drain node before maintenance"""
        try:
            subprocess.run(
                f"cockroach node drain --host={node} --certs-dir=certs",
                timeout=300
            )
            return True
        except Exception as e:
            self.logger.error(f"Drain failed for {node}: {e}")
            return False
    
    def monitor_cluster(self):
        """Continuous monitoring and auto-healing"""
        while True:
            for node in self.nodes:
                if not self.health_check(node):
                    self.logger.warning(f"Node {node} unhealthy, attempting recovery")
                    if not self.restart_node(node):
                        self.logger.error(f"Self-healing failed for {node}, escalating")
                        # Send alert to PagerDuty
                        send_alert(f"Node {node} requires manual intervention")
            
            time.sleep(10)  # Check every 10 seconds

# Usage
manager = SelfHealingDatabaseManager(['node1', 'node2', 'node3'])
manager.monitor_cluster()
```

---

## SECTION 47: SPECIALIZED INTEGRATIONS AND CONNECTORS

### Q149: How do I integrate CockroachDB with data lake/warehouse platforms?

1. Use CDC to stream real-time changes to data lake.
2. Scheduled batch exports for historical data.
3. Implement schemas in data warehouse matching CockroachDB.
4. Handle schema evolution in both systems.
5. Monitor sync latency and completeness.
6. Implement fallback queries for data validation.

```sql
-- Real-time CDC to data lake
CREATE CHANGEFEED FOR TABLE orders, customers, products
INTO 's3://data-lake/raw/orders-stream/'
WITH format='json', envelope='upsert', resolved='10s';

-- Scheduled batch export to warehouse
CREATE SCHEDULE warehouse_export FOR 
  EXPORT (SELECT * FROM analytics_table)
  TO 's3://warehouse-bucket/daily-export-{date}/'
  RECURRING EVERY 1 day;

-- Schema in data warehouse (Snowflake example)
CREATE TABLE orders_from_crdb (
  order_id INT,
  customer_id INT,
  amount DECIMAL,
  status VARCHAR,
  created_at TIMESTAMP
)
AS
SELECT * FROM 's3://data-lake/orders-export/orders.parquet';

-- Validate sync completeness
SELECT 
  'CockroachDB' as source,
  COUNT(*) as row_count,
  MAX(created_at) as latest_record
FROM orders
UNION ALL
SELECT 
  'Data Warehouse',
  COUNT(*),
  MAX(created_at)
FROM warehouse.orders_from_crdb;
```

### Q150: How do I implement real-time analytics with CockroachDB?

1. Use CDC for real-time data pipeline.
2. Stream to Apache Kafka for event processing.
3. Implement materialized views for common aggregations.
4. Update views incrementally rather than recalculating.
5. Serve analytics via REST API or BI tool.
6. Monitor end-to-end latency.

```sql
-- Real-time CDC to Kafka
CREATE CHANGEFEED FOR TABLE events
INTO 'kafka://broker:9092/events-stream'
WITH format='json', resolved='5s';

-- Materialized view for analytics
CREATE MATERIALIZED VIEW events_by_hour AS
SELECT 
  DATE_TRUNC('hour', event_time) as hour,
  event_type,
  COUNT(*) as count,
  COUNT(DISTINCT user_id) as unique_users
FROM events
WHERE event_time > now() - interval '24 hours'
GROUP BY DATE_TRUNC('hour', event_time), event_type;

-- Refresh schedule for materialized view
CREATE SCHEDULE refresh_analytics FOR 
  REFRESH MATERIALIZED VIEW events_by_hour
  RECURRING EVERY 5 minutes;

-- Real-time metrics endpoint (application code)
@app.route('/api/analytics/events')
def get_event_analytics():
    result = db.query("""
      SELECT 
        hour,
        event_type,
        count,
        unique_users
      FROM events_by_hour
      WHERE hour > now() - interval '24 hours'
      ORDER BY hour DESC
    """)
    return jsonify(result)
```

---

## SECTION 48: ADVANCED AUTOMATION AND ORCHESTRATION

### Q151: How do I implement Kubernetes-native CockroachDB deployment automation?

1. Use CockroachDB Kubernetes Operator.
2. Define cluster configuration in custom resource.
3. Operator handles provisioning, upgrades, scaling.
4. Use PersistentVolumes for data storage.
5. Implement monitoring via Prometheus ServiceMonitor.
6. Automate backup to cloud storage.

```yaml
# Kubernetes CockroachDB cluster manifest
apiVersion: cockroachdb.cockroachlabs.com/v1beta1
kind: CockroachDBCluster
metadata:
  name: production-cluster
  namespace: cockroachdb
spec:
  nodes: 3
  image:
    name: cockroachdb/cockroach:v24.2.0
  
  storage:
    persistentVolume:
      size: 100Gi
      storageClass: fast-ssd
  
  resources:
    requests:
      cpu: 4
      memory: 8Gi
    limits:
      cpu: 8
      memory: 16Gi
  
  tlsEnabled: true
  
  ingress:
    enabled: true
    className: nginx
    hosts:
      - host: cockroachdb.example.com
        paths:
          - path: /
            pathType: Prefix
  
  serviceMonitor:
    enabled: true
    interval: 30s

---
# Backup configuration
apiVersion: cockroachdb.cockroachlabs.com/v1beta1
kind: CockroachDBBackup
metadata:
  name: production-backup
spec:
  clusterName: production-cluster
  destination: s3://backup-bucket/production/
  schedule: "0 2 * * *"  # Daily at 2 AM
  retention: 30d
```

---

## SECTION 49: COMPREHENSIVE OPERATIONAL CHECKLISTS

### Q152: Daily Operations Checklist

```bash
#!/bin/bash
# Daily CockroachDB Operations Checklist

echo "=== Daily Operations Checklist ==="
DATE=$(date +%Y-%m-%d)

# 1. Health Check
echo "1. Checking cluster health..."
cockroach sql --certs-dir=certs <<EOF
SELECT 
  'Node Status' as check,
  COUNT(*) as healthy,
  COUNT(*) FILTER (WHERE is_live = false) as unhealthy
FROM crdb_internal.nodes;
EOF

# 2. Monitoring Verification
echo "2. Verifying monitoring systems..."
curl -s http://prometheus:9090/api/v1/targets | grep -q '"health":"up"' && echo "✓ Prometheus operational" || echo "✗ Prometheus issues"

# 3. Backup Verification
echo "3. Checking latest backup..."
aws s3 ls s3://backup-bucket/ --recursive --summarize | tail -5

# 4. Disk Space Check
echo "4. Checking disk usage..."
cockroach sql --certs-dir=certs <<EOF
SELECT 
  store_id,
  ROUND(capacity_used::float / capacity_available, 2) * 100 as usage_percent,
  CASE WHEN ROUND(capacity_used::float / capacity_available, 2) > 0.85 THEN 'WARNING'
       ELSE 'OK' END as status
FROM crdb_internal.stores
ORDER BY usage_percent DESC;
EOF

# 5. Query Performance
echo "5. Checking slow queries..."
cockroach sql --certs-dir=certs <<EOF
SELECT 
  query,
  COUNT(*) as count,
  ROUND(AVG(latency)::numeric, 0) as avg_latency_ms
FROM crdb_internal.node_statement_statistics
WHERE latency > 1000  -- > 1 second
GROUP BY query
ORDER BY count DESC
LIMIT 5;
EOF

echo "=== Daily Checklist Complete ==="
```

---

## SECTION 50: COMPLETE OPERATIONAL RUNBOOKS

### Q153: Runbook: Emergency Cluster Failover

```markdown
# Emergency Cluster Failover Runbook

## Trigger Conditions
- Primary cluster offline > 5 minutes
- Quorum lost (majority nodes down)
- Network partition between regions

## Pre-Failover Verification
1. Confirm primary cluster unrecoverable
2. Verify standby cluster healthy (all nodes live)
3. Confirm data replication current (< 1 second lag)

## Failover Steps
1. Page on-call database engineer
2. Stop write traffic to primary cluster
3. Verify no in-flight transactions
4. Promote standby to primary:
   ```sql
   ALTER CLUSTER SET REPLICATION FACTOR = 1;
   ```
5. Update DNS to point to standby cluster IP
6. Enable write traffic to new primary
7. Monitor for errors and rollback if necessary

## Post-Failover
1. Investigate primary cluster failure cause
2. Document incident timeline
3. Plan recovery of failed primary
4. Restore replication when original primary recovered
5. Post-incident review within 24 hours

## Estimated Duration
- Detection: 5 minutes
- Execution: 15 minutes
- Validation: 10 minutes
- Total RTO: 30 minutes
```

---

## SECTION 51: FINAL SUMMARY AND CLOSING RECOMMENDATIONS

### Q154: Strategic Database Architecture Recommendations

**For startup/early stage:**
- Single-region CockroachDB cluster (3 nodes)
- Basic monitoring with Prometheus/Grafana
- Daily backups to S3
- Application connection pooling
- Estimated cost: $500-1000/month

**For growth stage (1-10M records):**
- Multi-region for resilience (2 regions, 3 nodes each)
- Advanced monitoring with alerting
- Incremental backups + point-in-time recovery
- Read replicas for analytics
- Advanced RBAC and audit logging
- Estimated cost: $2000-5000/month

**For enterprise scale (100M+ records):**
- Multi-region globally distributed (3+ regions)
- Multi-tenant isolation
- Specialized OLTP/OLAP separation
- Real-time CDC to data lake
- Compliance and data governance automation
- Dedicated DBA team
- Estimated cost: $10000+/month

### Q155: Key Success Factors for Production CockroachDB

1. **Planning**: Thorough capacity planning and load testing before launch
2. **Team Expertise**: Invest in training team on distributed database concepts
3. **Monitoring**: Proactive monitoring beats reactive firefighting
4. **Automation**: Automate repetitive operations to reduce manual errors
5. **Testing**: Regularly test backup/restore and failover procedures
6. **Documentation**: Keep runbooks and architecture documentation updated
7. **Communication**: Clear escalation paths and incident communication
8. **Continuous Learning**: Share lessons learned across team

---

## SECTION 52: ADVANCED TROUBLESHOOTING DEEP-DIVES

### Q156: How do I debug replication consensus issues?

1. Monitor raft consensus through metrics
2. Check leader status and elections
3. Identify split-brain scenarios
4. Diagnose network partitions
5. Implement recovery procedures
6. Prevent future occurrences

```sql
-- Monitor raft consensus status
SELECT 
  node_id,
  SUM(CASE WHEN raft.heartbeats.pending > 0 THEN 1 ELSE 0 END) as pending_heartbeats,
  COUNT(*) as total_ranges
FROM crdb_internal.node_metrics
GROUP BY node_id
ORDER BY pending_heartbeats DESC;

-- Check for leadership issues
SELECT 
  range_id,
  lease_holder,
  leader_node,
  CASE WHEN lease_holder != leader_node THEN 'MISMATCH' ELSE 'OK' END as status
FROM crdb_internal.ranges
WHERE lease_holder != leader_node
LIMIT 20;

-- Detect split-brain (should not occur in CockroachDB with proper clock sync)
SELECT 
  node_id,
  COUNT(*) as leader_count
FROM crdb_internal.ranges_no_leases
WHERE is_leader = true
GROUP BY node_id
HAVING COUNT(*) > (SELECT COUNT(*) FROM crdb_internal.ranges) / 2;
```

---

## SECTION 53: PERFORMANCE TUNING MATRICES

### Q157: Decision matrix: When to scale horizontally vs vertically?

**Scale Horizontally When:**
- Need fault tolerance across zones/regions
- Want to distribute read load
- Plan to exceed single-node capacity
- Have <10 TB total data
- Can tolerate slight latency increase from replication

**Scale Vertically When:**
- Performance is CPU-bound
- Increase cache size improves hit ratio
- Reduce network round trips
- Temporary measure before horizontal scaling
- <3 node cluster causing operational complexity

**Recommended Matrix:**
```
Data Size         | Nodes | Scaling Strategy
<10GB            | 1     | Start single-node, scale horizontally before 10GB
10-100GB         | 3     | Baseline multi-region
100GB-1TB        | 5-9   | Horizontal scaling for fault tolerance
1-10TB           | 9-15  | Multi-region with read replicas
10TB+            | 15+   | Sharding + multi-region
```

---

## SECTION 54: COMPLIANCE AND REGULATORY FRAMEWORKS

### Q158: GDPR Compliance Checklist for CockroachDB

```bash
#!/bin/bash
# GDPR Compliance Verification Script

echo "=== GDPR Compliance Checklist ==="

# 1. Data Residency Verification
cockroach sql --certs-dir=certs <<EOF
-- Verify all EU customer data in EU regions only
SELECT 
  table_name,
  COUNT(*) as rows_outside_eu
FROM crdb_internal.ranges_no_leases r
LEFT JOIN system.zones z ON r.table_name = z.table_name
WHERE crdb_internal.get_node_locality(r.replica_node_id) -> 'region' NOT LIKE '%eu%'
  AND table_name IN ('customers', 'customer_data', 'personal_info')
GROUP BY table_name
HAVING COUNT(*) > 0;
EOF

# 2. Encryption Verification
echo "Checking encryption..."
cockroach sql --certs-dir=certs -c "
  SELECT setting FROM system.settings 
  WHERE name = 'server.encryption_at_rest.enabled';"

# 3. Audit Logging Verification
echo "Verifying audit logging..."
cockroach sql --certs-dir=certs -c "
  SELECT COUNT(*) FROM system.event_log 
  WHERE timestamp > now() - interval '1 day';"

# 4. Data Deletion Verification
echo "Testing GDPR right-to-be-forgotten..."
# Create test data, delete, verify deletion
TEST_ID=$(uuidgen)
cockroach sql --certs-dir=certs <<EOF
  INSERT INTO customers (customer_id, email) VALUES ('$TEST_ID', 'test@example.com');
  DELETE FROM customers WHERE customer_id = '$TEST_ID';
  SELECT COUNT(*) as remaining FROM customers WHERE customer_id = '$TEST_ID';
EOF

# 5. Backup Location Verification
echo "Checking backup locations..."
aws s3 ls s3://backup-bucket/ | grep "eu" && echo "✓ EU backups found" || echo "✗ No EU backups"

echo "=== Compliance Check Complete ==="
```

---

## SECTION 55: OPERATIONAL EFFICIENCY MATRICES

### Q159: Operational Cost Optimization Matrix

```
Cost Category          | Current Approach | Optimization Opportunity
Compute               | 24/7 full nodes  | Downsize off-peak, auto-scale
Storage               | All SSD          | Tiered storage (SSD/HDD)
Replication           | 5 replicas       | Reduce to 3 if latency OK
Backup                | Hourly retention | Tiered retention policy
Cross-region traffic  | Always replicate | Regional read replicas only
Monitoring            | High-resolution  | Adaptive resolution sampling

Example savings: 30-50% by combining optimizations
```

---

## SECTION 56: FINAL STRATEGIC QUESTIONS AND GUIDANCE

### Q160: What is the path to 100% availability SLA with CockroachDB?

1. **Multi-region replication**: Distribute across 3+ regions for fault tolerance
2. **Automated failover**: < 5 minute RTO via DNS switchover
3. **Health monitoring**: Real-time detection of failures
4. **Circuit breakers**: Graceful degradation when issues detected
5. **Load shedding**: Drop non-critical requests during failures
6. **Rate limiting**: Prevent cascading failures
7. **Chaos engineering**: Regularly test failure scenarios

Implementation achieves:
- 99.95% uptime (22 minutes/month downtime)
- Sub-second failover detection
- < 5 minute recovery time
- Data consistency maintained
- Zero data loss with synchronous replication
- Cost: requires 3+ regions and continuous operational attention

---

You're right. Let me continue from Q161 onwards to complete the remaining FAQs to reach 250+.

---

## SECTION 57: ADVANCED QUERY EXECUTION PATTERNS

### Q161: How do I implement query result caching for repeated queries?

1. Cache frequently executed queries in application layer.
2. Invalidate cache when underlying data changes.
3. Use TTL-based expiration for eventual consistency.
4. Monitor cache hit rates to validate effectiveness.
5. Implement cache warmup on application startup.
6. Consider Redis or Memcached for distributed caching.

```python
from functools import wraps
import time
import hashlib

class QueryCache:
    def __init__(self, ttl_seconds=300):
        self.cache = {}
        self.ttl = ttl_seconds
    
    def cache_query(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key from query and parameters
            cache_key = hashlib.md5(
                f"{func.__name__}:{args}:{kwargs}".encode()
            ).hexdigest()
            
            # Check cache
            if cache_key in self.cache:
                cached_value, cached_time = self.cache[cache_key]
                if time.time() - cached_time < self.ttl:
                    return cached_value
            
            # Execute query
            result = func(*args, **kwargs)
            
            # Store in cache
            self.cache[cache_key] = (result, time.time())
            return result
        
        return wrapper
    
    def invalidate(self, pattern=None):
        """Invalidate cache entries matching pattern"""
        if pattern is None:
            self.cache.clear()
        else:
            self.cache = {
                k: v for k, v in self.cache.items()
                if pattern not in k
            }

query_cache = QueryCache(ttl_seconds=300)

@query_cache.cache_query
def get_customer_orders(customer_id):
    return db.query("SELECT * FROM orders WHERE customer_id = ?", customer_id)

# Invalidate when data changes
def create_order(customer_id, order_data):
    db.execute("INSERT INTO orders VALUES (?)", order_data)
    query_cache.invalidate(f"get_customer_orders:{customer_id}")
```

### Q162: How do I handle pagination efficiently for large result sets?

1. Use cursor-based pagination (keyset pagination) for efficiency.
2. Avoid OFFSET which requires reading skipped rows.
3. Track last-seen key for next page request.
4. Index on pagination column for performance.
5. Implement timeout to prevent runaway queries.
6. Test with production-scale datasets.

```sql
-- Cursor-based pagination (efficient)
SELECT * FROM users
WHERE id > $1  -- Last ID from previous page
ORDER BY id
LIMIT 20;

-- Client passes last_id=1234 for next page
-- Query: SELECT * FROM users WHERE id > 1234 ORDER BY id LIMIT 20

-- Create index for efficient pagination
CREATE INDEX idx_users_id ON users(id);

-- Application implementation
def get_users_page(last_id=None, page_size=20):
    if last_id is None:
        query = "SELECT * FROM users ORDER BY id LIMIT ?"
        return db.query(query, page_size)
    else:
        query = "SELECT * FROM users WHERE id > ? ORDER BY id LIMIT ?"
        return db.query(query, last_id, page_size)

# Usage
page1 = get_users_page(page_size=20)
last_id = page1[-1]['id']
page2 = get_users_page(last_id=last_id, page_size=20)
```

### Q163: How do I implement result set filtering without full table scans?

1. Create indexes on frequently filtered columns.
2. Use selective indexes for specific value ranges.
3. Combine multiple indexes for complex filters.
4. Monitor filter effectiveness through explain plans.
5. Implement column statistics for optimizer accuracy.
6. Analyze and update statistics regularly.

```sql
-- Create indexes for common filters
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_date ON orders(created_at DESC);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at DESC);

-- Selective index for specific values
CREATE INDEX idx_active_orders ON orders(customer_id)
WHERE status = 'active';

-- Analyze and update statistics
ANALYZE TABLE orders;

-- Verify index usage with EXPLAIN
EXPLAIN SELECT * FROM orders
WHERE customer_id = 123 AND status = 'completed'
ORDER BY created_at DESC;

-- Monitor index effectiveness
SELECT 
  index_name,
  seq_scans,
  idx_scans,
  CASE 
    WHEN seq_scans > idx_scans THEN 'UNDERUTILIZED'
    ELSE 'EFFECTIVE'
  END as status
FROM crdb_internal.index_stats
WHERE table_name = 'orders'
ORDER BY seq_scans DESC;
```

---

## SECTION 58: ADVANCED TRANSACTION MANAGEMENT

### Q164: How do I handle distributed transaction deadlocks?

1. Deadlocks occur when transactions wait on each other.
2. CockroachDB automatically detects and resolves via restart.
3. Implement retry logic with exponential backoff in application.
4. Monitor deadlock frequency; high rates indicate schema issues.
5. Redesign queries or access patterns to reduce conflicts.
6. Test deadlock scenarios in staging.

```sql
-- Monitor deadlocks
SELECT * FROM system.event_log
WHERE event_type = 'transaction_abort'
  AND error LIKE '%deadlock%'
ORDER BY timestamp DESC
LIMIT 20;

-- Example deadlock scenario
-- Transaction 1: Lock order 1, then lock inventory 1
-- Transaction 2: Lock inventory 1, then lock order 1
-- Result: Deadlock

-- Application-level retry logic
def execute_with_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except DuplicateKeyError:
            raise  # Not a deadlock, fail immediately
        except Exception as e:
            if 'deadlock' in str(e).lower():
                if attempt < max_retries - 1:
                    wait_time = 2 ** attempt + random.uniform(0, 1)
                    time.sleep(wait_time)
                    continue
            raise

# Usage
def transfer_funds(from_account, to_account, amount):
    def do_transfer():
        BEGIN
        SELECT * FROM accounts WHERE id = from_account FOR UPDATE
        SELECT * FROM accounts WHERE id = to_account FOR UPDATE
        UPDATE accounts SET balance = balance - amount WHERE id = from_account
        UPDATE accounts SET balance = balance + amount WHERE id = to_account
        COMMIT
    
    return execute_with_retry(do_transfer)
```

### Q165: How do I implement optimistic locking for concurrent updates without explicit locks?

1. Add version column to table.
2. Check version in WHERE clause during update.
3. If version mismatch, operation fails and application retries.
4. Reduces lock contention compared to pessimistic locking.
5. Monitor conflict rates to ensure acceptable performance.
6. Suitable for read-heavy workloads with occasional writes.

```sql
CREATE TABLE inventory (
  product_id INT PRIMARY KEY,
  quantity INT,
  version INT DEFAULT 1,
  updated_at TIMESTAMP DEFAULT now()
);

-- Update with version check
UPDATE inventory
SET quantity = quantity - ?,
    version = version + 1,
    updated_at = now()
WHERE product_id = ? AND version = ?;

-- Check if update succeeded
-- If rows_affected = 0, version mismatch detected

-- Application retry logic
def update_inventory_optimistic(product_id, quantity_change, max_retries=3):
    for attempt in range(max_retries):
        # Get current state
        current = db.query_one(
            "SELECT quantity, version FROM inventory WHERE product_id = ?",
            product_id
        )
        
        # Attempt update
        result = db.execute(
            "UPDATE inventory SET quantity = ?, version = version + 1 "
            "WHERE product_id = ? AND version = ?",
            current['quantity'] + quantity_change,
            product_id,
            current['version']
        )
        
        if result.rows_affected > 0:
            return True
        
        # Conflict detected, retry
        time.sleep(0.01 * (2 ** attempt))
    
    raise UpdateFailedError("Failed after retries")
```

---

## SECTION 59: ADVANCED DATA MODELING

### Q166: How do I model hierarchical data efficiently in CockroachDB?

1. Use adjacency list: parent_id column for hierarchy.
2. Use nested sets for efficient range queries.
3. Use materialized path for breadcrumb queries.
4. Choose based on query patterns and update frequency.
5. Index appropriately for performance.
6. Monitor query performance for each pattern.

```sql
-- Adjacency List (simple, but slow for deep hierarchies)
CREATE TABLE categories (
  category_id INT PRIMARY KEY,
  parent_id INT REFERENCES categories(category_id),
  name VARCHAR,
  description TEXT
);

-- Query parent chain
WITH RECURSIVE category_path AS (
  SELECT category_id, parent_id, name, 1 as depth
  FROM categories
  WHERE category_id = 5
  
  UNION ALL
  
  SELECT c.category_id, c.parent_id, c.name, depth + 1
  FROM categories c
  JOIN category_path cp ON c.category_id = cp.parent_id
)
SELECT * FROM category_path ORDER BY depth;

-- Nested Sets (efficient for range queries)
CREATE TABLE categories_nested (
  category_id INT PRIMARY KEY,
  name VARCHAR,
  left_bound INT,
  right_bound INT
);

-- Query subtree
SELECT * FROM categories_nested
WHERE left_bound >= (SELECT left_bound FROM categories_nested WHERE category_id = 5)
  AND right_bound <= (SELECT right_bound FROM categories_nested WHERE category_id = 5);

-- Materialized Path (good balance)
CREATE TABLE categories_path (
  category_id INT PRIMARY KEY,
  name VARCHAR,
  path VARCHAR  -- '1/5/12/45'
);

CREATE INDEX idx_path ON categories_path(path);

-- Query subtree by path
SELECT * FROM categories_path
WHERE path LIKE '1/5/12/%' OR path = '1/5/12';
```

### Q167: How do I handle self-referencing foreign keys and avoid cycles?

1. Self-referencing FK common for hierarchies (employee supervisor).
2. Implement application-level cycle detection.
3. Use triggers to prevent invalid updates.
4. Validate hierarchy integrity on inserts.
5. Monitor for orphaned records.
6. Document cascade rules clearly.

```sql
-- Self-referencing foreign key
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  name VARCHAR,
  supervisor_id INT REFERENCES employees(employee_id),
  hired_at TIMESTAMP DEFAULT now()
);

-- Prevent cycles with trigger
CREATE FUNCTION check_no_cycles() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.supervisor_id IS NOT NULL THEN
    -- Check if NEW.supervisor_id reports to NEW.employee_id (cycle)
    IF EXISTS (
      WITH RECURSIVE chain AS (
        SELECT supervisor_id FROM employees WHERE employee_id = NEW.supervisor_id
        UNION ALL
        SELECT supervisor_id FROM employees e
        JOIN chain ON e.employee_id = chain.supervisor_id
      )
      SELECT 1 FROM chain WHERE supervisor_id = NEW.employee_id
    ) THEN
      RAISE EXCEPTION 'Cycle detected in employee hierarchy';
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employee_cycle_check
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION check_no_cycles();

-- Query reporting chain
WITH RECURSIVE reporting_chain AS (
  SELECT employee_id, name, supervisor_id, 1 as level
  FROM employees
  WHERE employee_id = 100
  
  UNION ALL
  
  SELECT e.employee_id, e.name, e.supervisor_id, level + 1
  FROM employees e
  JOIN reporting_chain rc ON e.employee_id = rc.supervisor_id
)
SELECT * FROM reporting_chain;
```

### Q168: How do I implement soft deletes vs hard deletes efficiently?

1. Soft delete: mark deleted_at timestamp, data remains.
2. Hard delete: physically remove data.
3. Soft deletes simplify recovery but bloat tables.
4. Hard deletes reclaim storage but prevent recovery.
5. Choose based on compliance/recovery requirements.
6. Monitor soft-delete bloat over time.

```sql
-- Soft delete approach
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  email VARCHAR,
  created_at TIMESTAMP DEFAULT now(),
  deleted_at TIMESTAMP  -- NULL means active
);

CREATE INDEX idx_active_users ON users(user_id) WHERE deleted_at IS NULL;

-- Query only active users
SELECT * FROM users WHERE deleted_at IS NULL;

-- Soft delete operation
UPDATE users SET deleted_at = now() WHERE user_id = 123;

-- Recovery from soft delete
UPDATE users SET deleted_at = NULL WHERE user_id = 123;

-- Hard delete (permanent)
DELETE FROM users WHERE deleted_at IS NOT NULL AND deleted_at < now() - interval '30 days';

-- Monitor soft-delete bloat
SELECT 
  COUNT(*) FILTER (WHERE deleted_at IS NULL) as active_count,
  COUNT(*) FILTER (WHERE deleted_at IS NOT NULL) as deleted_count,
  COUNT(*) as total_count,
  ROUND(100.0 * COUNT(*) FILTER (WHERE deleted_at IS NOT NULL) / COUNT(*), 2) as deleted_percent
FROM users;
```

---

## SECTION 60: ADVANCED INDEXING STRATEGIES

### Q169: How do I choose between single-column and multi-column indexes?

1. Single-column: simple, low write overhead, good for selective filters.
2. Multi-column: covers common query patterns, reduces table access.
3. Order matters in multi-column indexes (leading column most selective).
4. Monitor index usage; remove unused indexes.
5. Consider index size and write impact.
6. Test both approaches in staging.

```sql
-- Single-column indexes (for independent filters)
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_customer ON orders(customer_id);
CREATE INDEX idx_date ON orders(created_at);

-- Multi-column index (for common query pattern)
-- Query: WHERE customer_id = 123 AND status = 'completed' ORDER BY created_at DESC
CREATE INDEX idx_orders_composite ON orders(customer_id, status, created_at DESC);

-- Verify index usage
EXPLAIN SELECT * FROM orders
WHERE customer_id = 123 AND status = 'completed'
ORDER BY created_at DESC;

-- Monitor index effectiveness
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan as index_scans,
  idx_tup_read as tuples_read,
  idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Remove unused indexes (be careful)
DROP INDEX idx_unused_index;
```

### Q170: How do I implement covering indexes for query optimization?

1. Covering index includes all columns needed by query.
2. Avoids table access, improves performance.
3. Trade-off: larger index, slower writes.
4. Best for read-heavy workloads.
5. Monitor covering index effectiveness.
6. Update statistics for accurate query planning.

```sql
-- Covering index includes non-key columns
-- Query: SELECT customer_id, total_amount FROM orders WHERE status = 'completed'
CREATE INDEX idx_orders_covering ON orders(status) INCLUDE (customer_id, total_amount);

-- Query optimizer chooses covering index
EXPLAIN SELECT customer_id, total_amount
FROM orders
WHERE status = 'completed';
-- Result: IndexScan (no table access needed)

-- Without covering index (requires table access)
-- Result: IndexScan -> TableScan

-- Monitor covering index size and effectiveness
SELECT 
  index_name,
  pg_size_pretty(pg_relation_size('index_name'::regclass)) as index_size,
  idx_scan as scans
FROM pg_stat_user_indexes
WHERE index_name LIKE 'idx_orders%';
```

---

## SECTION 61: ADVANCED SECURITY AND AUDIT

### Q171: How do I implement column-level encryption for sensitive data?

1. CockroachDB encryption at rest (cluster-level).
2. Application-level encryption for column-level secrecy.
3. Encrypt before insertion, decrypt after retrieval.
4. Manage encryption keys separately from data.
5. Monitor key access and rotation.
6. Test encryption/decryption performance impact.

```python
from cryptography.fernet import Fernet
import base64

class ColumnEncryption:
    def __init__(self, master_key):
        self.cipher = Fernet(master_key)
    
    def encrypt_value(self, plaintext):
        """Encrypt sensitive data"""
        return self.cipher.encrypt(plaintext.encode()).decode()
    
    def decrypt_value(self, ciphertext):
        """Decrypt sensitive data"""
        return self.cipher.decrypt(ciphertext.encode()).decode()

# Usage
encryption = ColumnEncryption(b'your-encryption-key')

# Encrypt before insert
encrypted_ssn = encryption.encrypt_value('123-45-6789')
db.execute("INSERT INTO customers (ssn) VALUES (?)", encrypted_ssn)

# Decrypt after retrieval
encrypted_data = db.query_one("SELECT ssn FROM customers WHERE id = 1")
decrypted_ssn = encryption.decrypt_value(encrypted_data['ssn'])

# SQL-level transparent encryption
CREATE TABLE customers_secure (
  customer_id INT PRIMARY KEY,
  email VARCHAR,
  ssn_encrypted VARCHAR,  -- Encrypted at application level
  phone_encrypted VARCHAR  -- Encrypted at application level
);

-- Never store plaintext sensitive data
-- Always encrypt before insert
-- Always decrypt after retrieval
```

### Q172: How do I audit data access and track who accessed what data?

1. Implement audit triggers on sensitive tables.
2. Log SELECT queries on sensitive tables (application-level).
3. Create immutable audit log.
4. Query audit log for data access patterns.
5. Alert on suspicious access patterns.
6. Regular audit log review.

```sql
-- Audit log table
CREATE TABLE audit_log (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  table_name VARCHAR,
  operation VARCHAR,  -- SELECT, INSERT, UPDATE, DELETE
  user_name VARCHAR,
  record_id INT,
  old_values JSONB,
  new_values JSONB,
  accessed_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_audit_table_time ON audit_log(table_name, accessed_at DESC);
CREATE INDEX idx_audit_user ON audit_log(user_name, accessed_at DESC);

-- Audit trigger for INSERT/UPDATE/DELETE
CREATE FUNCTION audit_data_changes() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (table_name, operation, user_name, record_id, new_values)
    VALUES (TG_TABLE_NAME, 'INSERT', current_user, NEW.id, row_to_json(NEW));
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (table_name, operation, user_name, record_id, old_values, new_values)
    VALUES (TG_TABLE_NAME, 'UPDATE', current_user, NEW.id, row_to_json(OLD), row_to_json(NEW));
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (table_name, operation, user_name, record_id, old_values)
    VALUES (TG_TABLE_NAME, 'DELETE', current_user, OLD.id, row_to_json(OLD));
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sensitive_table_audit
AFTER INSERT OR UPDATE OR DELETE ON sensitive_table
FOR EACH ROW EXECUTE FUNCTION audit_data_changes();

-- Application-level audit for SELECT
def audit_select_query(table_name, user_id, filter_conditions):
    db.execute(
        "INSERT INTO audit_log (table_name, operation, user_name, accessed_at) "
        "VALUES (?, ?, ?, now())",
        table_name,
        'SELECT',
        user_id
    )

-- Query audit log
SELECT 
  audit_id,
  user_name,
  operation,
  table_name,
  record_id,
  accessed_at
FROM audit_log
WHERE table_name = 'customers' AND accessed_at > now() - interval '7 days'
ORDER BY accessed_at DESC;

-- Alert on suspicious access
SELECT 
  user_name,
  COUNT(*) as access_count,
  COUNT(DISTINCT table_name) as tables_accessed
FROM audit_log
WHERE accessed_at > now() - interval '1 hour'
GROUP BY user_name
HAVING access_count > 1000;  -- Unusual access pattern
```

---

## SECTION 62: ADVANCED WORKLOAD CHARACTERISTICS

### Q173: How do I identify and optimize for write-heavy workloads?

1. Monitor write throughput and latency metrics.
2. Identify write-intensive tables.
3. Optimize for batch writes.
4. Consider partitioning for parallel inserts.
5. Monitor replication overhead.
6. Balance consistency vs performance.

```sql
-- Monitor write performance
SELECT 
  table_name,
  ranges_writes_per_second,
  avg_write_latency_ms
FROM crdb_internal.table_metrics
WHERE ranges_writes_per_second > 1000
ORDER BY ranges_writes_per_second DESC;

-- Optimize batch writes
INSERT INTO events (timestamp, event_type, data)
SELECT * FROM (VALUES
  (now(), 'click', '{"page": "home"}'),
  (now(), 'click', '{"page": "products"}'),
  (now(), 'view', '{"product": "123"}')
) AS v(timestamp, event_type, data);

-- Partition for parallel writes
CREATE TABLE events_partitioned (
  timestamp TIMESTAMP NOT NULL,
  event_type VARCHAR,
  data JSONB,
  PRIMARY KEY (timestamp DESC, event_type)
) PARTITION BY RANGE (timestamp) (
  PARTITION p_2024_01_01 VALUES FROM ('2024-01-01') TO ('2024-01-02'),
  PARTITION p_2024_01_02 VALUES FROM ('2024-01-02') TO ('2024-01-03')
);

-- Monitor write latency
SELECT 
  node_id,
  sql_exec_latency_p99 as write_latency_p99
FROM crdb_internal.node_metrics
ORDER BY write_latency_p99 DESC;
```

### Q174: How do I handle read-heavy workloads with minimal write latency?

1. Use read replicas for scaling reads.
2. Enable follower reads for local reads.
3. Implement application-level caching.
4. Denormalize data for common queries.
5. Monitor read-to-write ratio.
6. Balance replication overhead with read benefits.

```sql
-- Enable follower reads for read-heavy queries
SET SESSION enable_follower_reads = true;

SELECT * FROM products
WHERE id = 123
AS OF SYSTEM TIME follower_read_timestamp();

-- Create read-optimized view
CREATE VIEW product_summary AS
SELECT 
  p.id,
  p.name,
  p.category,
  COUNT(r.id) as review_count,
  AVG(r.rating) as avg_rating
FROM products p
LEFT JOIN reviews r ON p.id = r.product_id
GROUP BY p.id, p.name, p.category;

-- Application-level caching
import redis

cache = redis.Redis(host='localhost', port=6379)

def get_product(product_id):
    # Check cache first
    cached = cache.get(f'product:{product_id}')
    if cached:
        return json.loads(cached)
    
    # Fetch from database
    product = db.query_one("SELECT * FROM products WHERE id = ?", product_id)
    
    # Store in cache (1 hour TTL)
    cache.setex(f'product:{product_id}', 3600, json.dumps(product))
    
    return product

# Invalidate cache on write
def update_product(product_id, data):
    db.execute("UPDATE products SET ... WHERE id = ?", product_id)
    cache.delete(f'product:{product_id}')
```

---

## SECTION 63: ADVANCED OPERATIONAL PATTERNS

### Q175: How do I implement automatic failover for application layer?

1. Health checks detect database unavailability.
2. DNS or load balancer routes to secondary.
3. Application reconnects to new primary.
4. Monitor failover time and completeness.
5. Test failover regularly.
6. Document failover procedures.

```python
import socket
import dns.resolver
from datetime import datetime

class DatabaseFailover:
    def __init__(self, primary_host, secondary_host, health_check_interval=10):
        self.primary_host = primary_host
        self.secondary_host = secondary_host
        self.health_check_interval = health_check_interval
        self.current_host = primary_host
        self.last_failover = None
    
    def health_check(self, host):
        """Check if database is accessible"""
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(5)
            result = sock.connect_ex((host, 26257))
            sock.close()
            return result == 0
        except Exception as e:
            return False
    
    def failover(self):
        """Switch to secondary database"""
        if self.current_host == self.primary_host:
            self.current_host = self.secondary_host
            self.last_failover = datetime.now()
            self.update_dns()
            self.notify_team()
    
    def failback(self):
        """Switch back to primary database"""
        if self.current_host == self.secondary_host:
            if self.health_check(self.primary_host):
                self.current_host = self.primary_host
                self.update_dns()
                self.notify_team()
    
    def update_dns(self):
        """Update DNS to point to active database"""
        # Update DNS records
        pass
    
    def notify_team(self):
        """Alert team of failover"""
        # Send PagerDuty alert
        pass
    
    def get_connection_string(self):
        """Get current database connection string"""
        return f"postgresql://user@{self.current_host}:26257/db"

# Monitor and failover
failover_manager = DatabaseFailover('primary.example.com', 'secondary.example.com')

while True:
    if not failover_manager.health_check(failover_manager.current_host):
        failover_manager.failover()
    else:
        failover_manager.failback()
    
    time.sleep(failover_manager.health_check_interval)
```

### Q176: How do I implement automatic scaling based on load?

1. Monitor CPU, memory, and throughput metrics.
2. Define scaling thresholds and policies.
3. Automate node addition when exceeded.
4. Automate node removal when underutilized.
5. Prevent scaling thrashing with cooldown periods.
6. Test scaling procedures in staging.

```python
import time
from prometheus_client import query_prometheus

class AutoScaler:
    def __init__(self, cluster_name, min_nodes=3, max_nodes=20):
        self.cluster_name = cluster_name
        self.min_nodes = min_nodes
        self.max_nodes = max_nodes
        self.scale_cooldown = 300  # 5 minutes
        self.last_scale_time = 0
    
    def get_current_metrics(self):
        """Fetch cluster metrics from Prometheus"""
        cpu_usage = query_prometheus(
            'avg(node_cpu_usage) for cluster'
        )
        memory_usage = query_prometheus(
            'avg(node_memory_usage / node_memory_total)'
        )
        throughput = query_prometheus(
            'rate(sql_exec_success[5m])'
        )
        return {
            'cpu_usage': cpu_usage,
            'memory_usage': memory_usage,
            'throughput': throughput
        }
    
    def should_scale_up(self, metrics):
        """Determine if cluster should scale up"""
        return (
            metrics['cpu_usage'] > 0.80 or
            metrics['memory_usage'] > 0.85 or
            metrics['throughput'] > 50000  # queries per second
        )
    
    def should_scale_down(self):
        """Determine if cluster should scale down"""
        metrics = self.get_current_metrics()
        return (
            metrics['cpu_usage'] < 0.30 and
            metrics['memory_usage'] < 0.40 and
            metrics['throughput'] < 5000
        )
    
    def add_node(self):
        """Add new node to cluster"""
        current_nodes = self.get_node_count()
        if current_nodes < self.max_nodes:
            # API call to cloud provider to add node
            self.provision_node(f"node{current_nodes + 1}")
    
    def remove_node(self):
        """Remove node from cluster"""
        current_nodes = self.get_node_count()
        if current_nodes > self.min_nodes:
            # Drain and remove node
            self.drain_and_remove_node(f"node{current_nodes}")
    
    def auto_scale(self):
        """Main scaling loop"""
        while True:
            if time.time() - self.last_scale_time < self.scale_cooldown:
                time.sleep(10)
                continue
            
            metrics = self.get_current_metrics()
            
            if self.should_scale_up(metrics):
                self.add_node()
                self.last_scale_time = time.time()
            
            elif self.should_scale_down():
                self.remove_node()
                self.last_scale_time = time.time()
            
            time.sleep(30)

# Start auto-scaling
scaler = AutoScaler('production-cluster')
scaler.auto_scale()
```

---

## SECTION 64: ADVANCED MONITORING DEEP DIVE

### Q177: How do I implement custom metrics and alerting for business KPIs?

1. Track business metrics alongside technical metrics.
2. Alert on KPI thresholds, not just infrastructure.
3. Correlate business and technical metrics.
4. Implement dashboards for different audiences.
5. Review metrics regularly for accuracy.
6. Document metric definitions clearly.

```python
from prometheus_client import Counter, Histogram, Gauge

# Business metrics
orders_total = Counter(
    'orders_total',
    'Total orders processed',
    ['status', 'region']
)

order_value = Histogram(
    'order_value_dollars',
    'Distribution of order values',
    buckets=(10, 50, 100, 500, 1000, 5000)
)

customer_lifetime_value = Gauge(
    'customer_lifetime_value_dollars',
    'Customer lifetime value',
    ['segment']
)

# Record metrics
def process_order(order_data):
    order_total = order_data['amount']
    region = order_data['region']
    
    # Record transaction
    orders_total.labels(status='completed', region=region).inc()
    order_value.observe(order_total)
    
    # Update customer CLV
    customer_id = order_data['customer_id']
    clv = calculate_customer_lifetime_value(customer_id)
    customer_lifetime_value.labels(segment='high_value').set(clv)

# Alerting rules (in Prometheus)
# groups:
#   - name: business_metrics
#     rules:
#       - alert: LowOrdersPerMinute
#         expr: rate(orders_total[5m]) < 1
#         for: 15m
#         annotations:
#           summary: "Order rate dropped below 1/min"
#
#       - alert: HighOrderValueVariance
#         expr: stddev(order_value) > 1000
#         annotations:
#           summary: "Unusual order value distribution"
```

### Q178: How do I diagnose performance issues using flame graphs?

1. Collect CPU profiling data from CockroachDB.
2. Generate flame graphs for visualization.
3. Identify hot functions and bottlenecks.
4. Correlate flame graphs with query execution.
5. Monitor changes after optimizations.
6. Archive flame graphs for trend analysis.

```bash
#!/bin/bash
# Generate CPU flame graph

# 1. Collect CPU profile (30 seconds)
pprof_url="http://localhost:8080/debug/pprof/profile?seconds=30"
curl "$pprof_url" > cpu.prof

# 2. Convert to flame graph format
go-torch --file=cpu.prof --output=flame.svg

# 3. Analyze with flamegraph.pl
perf record -F 99 -p $(pgrep -f cockroach) -- sleep 30
perf script > out.perf
stackcollapse-perf.pl out.perf > out.folded
flamegraph.pl out.folded > flame.svg

# 4. View and analyze
# Open flame.svg in browser
# Look for wide, tall bars indicating hot functions

# 5. Correlate with queries
# Identify if flame graph aligns with expected query execution pattern
# Use EXPLAIN ANALYZE to validate

# 6. Apply optimizations
# Add indexes, denormalize, refactor queries
# Re-profile to verify improvements
```

---

## SECTION 65: SPECIALIZED DEPLOYMENT SCENARIOS

### Q179: How do I deploy CockroachDB in air-gapped (offline) environment?

1. Pre-download all binaries, dependencies, and images.
2. Transfer via approved secure channel.
3. Configure cluster without internet access.
4. Implement local backup/restore without cloud storage.
5. Monitor for connectivity issues.
6. Document offline procedures.

```bash
#!/bin/bash
# Air-gapped deployment

# 1. Download on connected system
cd /tmp/offline-download
wget https://binaries.cockroachdb.com/cockroach-v24.2.0.linux-amd64.tgz
wget https://github.com/prometheus/prometheus/releases/download/v2.40.0/prometheus-2.40.0.linux-amd64.tar.gz

# 2. Create offline package
tar czf cockroachdb-offline-package.tar.gz \
  cockroach-v24.2.0.linux-amd64.tgz \
  prometheus-2.40.0.linux-amd64.tar.gz \
  README_OFFLINE.md

# 3. Transfer via secure channel (USB, approved transfer)
# scp cockroachdb-offline-package.tar.gz user@offline-server:

# 4. Extract on offline system
tar xzf cockroachdb-offline-package.tar.gz
tar xzf cockroach-v24.2.0.linux-amd64.tgz -C /usr/local/bin/
tar xzf prometheus-2.40.0.linux-amd64.tar.gz -C /opt/

# 5. Start cluster (no internet required)
cockroach start --certs-dir=certs --listen-addr=0.0.0.0:26257

# 6. Local backup to NFS/network storage
cockroach sql --certs-dir=certs <<EOF
BACKUP INTO 'file:///mnt/nfs-backup/backup-2024-01-15';
EOF

# 7. Verify cluster operational without internet
cockroach sql --certs-dir=certs -c "SELECT 1;"
```

### Q180: How do I handle CockroachDB deployment on resource-constrained environments?

1. Reduce cache sizes: --cache=512MB for limited RAM.
2. Minimize replication factor: 3 nodes minimum.
3. Use smaller block sizes for limited storage.
4. Limit connections per node.
5. Monitor resource exhaustion carefully.
6. Plan for eventual upgrade when possible.

```bash
# CockroachDB on resource-constrained environment
cockroach start \
  --cache=512MB \
  --max-sql-memory=256MB \
  --listen-addr=0.0.0.0:26257 \
  --store=/data/cockroach \
  --certs-dir=certs

# SQL configuration for limited resources
cockroach sql --certs-dir=certs <<EOF
-- Limit concurrent connections
ALTER ROLE app_user CONNECTION LIMIT 50;

-- Reduce statement timeout
SET CLUSTER SETTING sql.defaults.statement_timeout = '30s';

-- Disable expensive operations
SET CLUSTER SETTING sql.mutations.max_row_size_err = '10MB';
EOF

# Monitor resource usage
top -p $(pgrep -f cockroach)
df -h /data/cockroach
free -h
```

---

## SECTION 66: FINAL ADVANCED TOPICS

### Q181: How do I implement disaster recovery for ransomware scenarios?

1. Maintain offline backup copies (immutable storage).
2. Implement air-gapped backup environment.
3. Regular restore testing from offline backups.
4. Version backups for point-in-time recovery.
5. Monitor backup integrity checksums.
6. Plan recovery procedures for multiple failure scenarios.

```bash
#!/bin/bash
# Ransomware disaster recovery

# 1. Immutable backup to offline storage
# Use WORM (Write Once Read Many) storage
cockroach sql --certs-dir=certs <<EOF
BACKUP INTO 's3://immutable-bucket/backup-2024-01-15'
  WITH S3_STORAGE_CLASS='GLACIER';  -- Archive storage
EOF

# 2. Regular integrity verification
# Calculate checksum of backup
aws s3 sync s3://immutable-bucket/backup-2024-01-15 /tmp/backup-check
find /tmp/backup-check -type f -exec sha256sum {} \; > backup-checksums.txt

# 3. Test restore from offline backup
# 1. Provision clean infrastructure
# 2. Download backup from offline storage
# 3. Restore cluster
# 4. Validate data integrity
# 5. Test application connectivity

# 4. Ransomware detection
# Monitor for unusual database access patterns
cockroach sql --certs-dir=certs <<EOF
SELECT 
  user_name,
  COUNT(*) as access_count,
  MAX(timestamp) as last_access
FROM system.event_log
WHERE timestamp > now() - interval '1 hour'
GROUP BY user_name
HAVING COUNT(*) > 10000;  -- Unusual activity
EOF

# 5. Incident response
# 1. Detect ransomware activity (suspicious bulk operations)
# 2. Isolate affected database from network
# 3. Preserve evidence (logs, metrics)
# 4. Initiate restore from verified backup
# 5. Restore to offline infrastructure first
# 6. Validate before reconnecting to network
```

### Q182: How do I implement continuous compliance validation?

1. Automated compliance checks scheduled regularly.
2. Verify data residency, encryption, audit logging.
3. Generate compliance reports automatically.
4. Alert on compliance violations.
5. Document compliance validation procedures.
6. Archive compliance reports for audits.

```sql
-- Automated compliance validation
CREATE TABLE compliance_status (
  check_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  check_name VARCHAR,
  check_status VARCHAR,  -- PASS, FAIL
  check_details JSONB,
  checked_at TIMESTAMP DEFAULT now()
);

-- Scheduled compliance job
CREATE SCHEDULE compliance_check FOR 
  (INSERT INTO compliance_status (check_name, check_status, check_details)
   SELECT 
     'Encryption at Rest' as check_name,
     CASE WHEN (SELECT setting FROM system.settings 
               WHERE name = 'server.encryption_at_rest.enabled') = 'true' 
       THEN 'PASS' ELSE 'FAIL' END,
     jsonb_build_object(
       'setting', 'server.encryption_at_rest.enabled',
       'checked_at', now()
     ))
  RECURRING EVERY 1 day;

-- Data residency check
INSERT INTO compliance_status (check_name, check_status, check_details)
SELECT 
  'Data Residency EU' as check_name,
  CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END,
  jsonb_build_object(
    'non_compliant_ranges', COUNT(*),
    'checked_at', now()
  )
FROM crdb_internal.ranges_no_leases
WHERE table_name IN ('customer_data', 'personal_info')
  AND crdb_internal.get_node_locality(replica_node_id) -> 'region' NOT LIKE '%eu%';

-- Audit logging verification
INSERT INTO compliance_status (check_name, check_status, check_details)
SELECT 
  'Audit Logging Active' as check_name,
  CASE WHEN COUNT(*) > 0 THEN 'PASS' ELSE 'FAIL' END,
  jsonb_build_object(
    'recent_audit_events', COUNT(*),
    'checked_at', now()
  )
FROM system.event_log
WHERE timestamp > now() - interval '1 hour';

-- Generate compliance report
SELECT 
  DATE_TRUNC('day', checked_at) as check_date,
  check_name,
  check_status,
  COUNT(*) as check_count
FROM compliance_status
WHERE checked_at > now() - interval '30 days'
GROUP BY DATE_TRUNC('day', checked_at), check_name, check_status
ORDER BY check_date DESC, check_name;
```

### Q183: How do I implement predictive maintenance for database cluster?

1. Monitor hardware health metrics proactively.
2. Predict disk failures from SMART data.
3. Forecast CPU/memory utilization trends.
4. Schedule maintenance before failures.
5. Implement automatic remediation where safe.
6. Document maintenance history.

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from datetime import datetime, timedelta

class PredictiveMaintenanceManager:
    def __init__(self, lookback_days=30):
        self.lookback_days = lookback_days
        self.model = LinearRegression()
    
    def predict_disk_failure(self, disk_metrics):
        """Predict disk failure based on SMART data"""
        # Collect historical SMART data
        # Parse reallocated_sectors, seek_error_rate, etc.
        
        # Train model on historical data
        X = []  # Time in days
        y = []  # Error rate
        
        for i, metric in enumerate(disk_metrics):
            X.append([i])  # Day index
            y.append(metric['error_rate'])
        
        self.model.fit(X, y)
        
        # Predict next 30 days
        future_days = np.arange(len(disk_metrics), len(disk_metrics) + 30).reshape(-1, 1)
        predictions = self.model.predict(future_days)
        
        # Alert if error rate exceeds threshold
        failure_threshold = 1000
        if np.max(predictions) > failure_threshold:
            return {
                'risk': 'HIGH',
                'predicted_days_to_failure': np.argmax(predictions > failure_threshold),
                'recommended_action': 'Schedule disk replacement'
            }
        
        return {'risk': 'LOW'}
    
    def predict_capacity_exhaustion(self, capacity_metrics):
        """Predict when cluster capacity will be exhausted"""
        X = []
        y = []
        
        for i, metric in enumerate(capacity_metrics):
            X.append([i])
            y.append(metric['used_percent'])
        
        self.model.fit(X, y)
        
        # Predict when 90% capacity will be reached
        threshold = 90
        future_days = np.arange(len(capacity_metrics), len(capacity_metrics) + 90).reshape(-1, 1)
        predictions = self.model.predict(future_days)
        
        if np.any(predictions > threshold):
            days_to_full = np.argmax(predictions > threshold)
            return {
                'capacity_risk': 'HIGH',
                'days_to_full': days_to_full,
                'recommended_action': f'Scale cluster in {days_to_full - 7} days'
            }
        
        return {'capacity_risk': 'LOW'}

# Usage
maintenance_manager = PredictiveMaintenanceManager()

# Get disk metrics from monitoring system
disk_metrics = fetch_disk_metrics(lookback_days=30)
disk_prediction = maintenance_manager.predict_disk_failure(disk_metrics)

if disk_prediction['risk'] == 'HIGH':
    schedule_maintenance(f"Replace disk - {disk_prediction['recommended_action']}")

# Get capacity metrics
capacity_metrics = fetch_capacity_metrics(lookback_days=30)
capacity_prediction = maintenance_manager.predict_capacity_exhaustion(capacity_metrics)

if capacity_prediction['capacity_risk'] == 'HIGH':
    schedule_capacity_planning(capacity_prediction['recommended_action'])
```

### Q184: How do I implement cost optimization strategies for CockroachDB?

1. Right-size cluster for actual workload.
2. Use spot/preemptible instances where appropriate.
3. Implement reserved capacity for predictable load.
4. Schedule nodes down during off-peak hours.
5. Compress backups to reduce storage costs.
6. Monitor cost per transaction.

```python
class CostOptimizationManager:
    def __init__(self, cloud_provider='aws'):
        self.cloud = cloud_provider
        self.cost_history = []
    
    def calculate_current_cost(self, cluster_config):
        """Calculate monthly cluster cost"""
        # Example AWS pricing (as of 2024)
        on_demand_hourly = 1.00  # per vCPU-hour
        storage_monthly = 0.023  # per GB-month
        
        node_count = cluster_config['nodes']
        vcpu_per_node = cluster_config['vcpu']
        storage_gb = cluster_config['storage_gb']
        
        # Calculate compute cost
        compute_cost_monthly = node_count * vcpu_per_node * on_demand_hourly * 730  # 730 hours/month
        
        # Calculate storage cost
        storage_cost_monthly = storage_gb * storage_monthly
        
        # Add replication overhead (roughly 3x for 5 replicas)
        replication_multiplier = cluster_config.get('replication_factor', 5) / 3
        total_cost = (compute_cost_monthly + storage_cost_monthly) * replication_multiplier
        
        return {
            'compute_cost': compute_cost_monthly,
            'storage_cost': storage_cost_monthly,
            'total_monthly': total_cost,
            'cost_per_hour': total_cost / 730
        }
    
    def recommend_optimizations(self, current_cost, usage_metrics):
        """Recommend cost optimizations"""
        recommendations = []
        
        # Check for over-provisioned CPU
        if usage_metrics['cpu_utilization'] < 30:
            recommendations.append({
                'type': 'REDUCE_VCPU',
                'potential_savings': current_cost['compute_cost'] * 0.3,
                'action': 'Reduce vCPU per node by 30%'
            })
        
        # Check for over-provisioned storage
        if usage_metrics['storage_utilization'] < 50:
            recommendations.append({
                'type': 'REDUCE_STORAGE',
                'potential_savings': current_cost['storage_cost'] * 0.3,
                'action': 'Archive old data to reduce storage size'
            })
        
        # Check for spot instance eligibility
        if usage_metrics['workload_interruption_tolerant']:
            recommendations.append({
                'type': 'USE_SPOT_INSTANCES',
                'potential_savings': current_cost['compute_cost'] * 0.7,
                'action': 'Use spot instances for non-critical workloads'
            })
        
        return recommendations

# Usage
cost_manager = CostOptimizationManager('aws')

current_config = {
    'nodes': 5,
    'vcpu': 8,
    'storage_gb': 500,
    'replication_factor': 5
}

current_cost = cost_manager.calculate_current_cost(current_config)

usage_metrics = {
    'cpu_utilization': 25,
    'storage_utilization': 40,
    'workload_interruption_tolerant': False
}

recommendations = cost_manager.recommend_optimizations(current_cost, usage_metrics)

print(f"Current monthly cost: ${current_cost['total_monthly']:.2f}")
for rec in recommendations:
    print(f"- {rec['action']}: Save ${rec['potential_savings']:.2f}/month")
```

### Q185: How do I handle distributed tracing across microservices using CockroachDB?

1. Propagate trace IDs through all services.
2. Log trace IDs in application and database logs.
3. Correlate database queries with application traces.
4. Visualize end-to-end request flow.
5. Identify bottlenecks across services.
6. Implement distributed context propagation.

```python
import opentelemetry.trace as trace
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
import psycopg2

# Initialize Jaeger tracing
jaeger_exporter = JaegerExporter(
    agent_host_name='localhost',
    agent_port=6831,
)

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

class DistributedTracer:
    def __init__(self):
        self.tracer = tracer
    
    def trace_database_query(self, query, params=None):
        """Trace database query execution"""
        with self.tracer.start_as_current_span("db.query") as span:
            span.set_attribute("db.statement", query)
            span.set_attribute("db.system", "cockroachdb")
            
            # Get current trace ID
            trace_id = span.get_span_context().trace_id
            
            try:
                # Execute query with trace ID in context
                conn = psycopg2.connect('dbname=mydb user=root')
                cursor = conn.cursor()
                cursor.execute(query, params or ())
                result = cursor.fetchall()
                
                span.set_attribute("db.rows", len(result))
                span.set_status(trace.Status(trace.StatusCode.OK))
                
                return result
            
            except Exception as e:
                span.set_attribute("error", True)
                span.set_attribute("error.message", str(e))
                raise

# Usage in microservice
def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        
        # Trace database call
        distributed_tracer = DistributedTracer()
        
        # Query customer
        customer = distributed_tracer.trace_database_query(
            "SELECT * FROM customers WHERE id = ?",
            [order_id]
        )
        
        # Query inventory
        inventory = distributed_tracer.trace_database_query(
            "SELECT * FROM inventory WHERE product_id = ?",
            [order_id]
        )
        
        # Process order
        with tracer.start_as_current_span("payment_processing") as payment_span:
            # Call payment service
            payment_result = call_payment_service(order_id)
            payment_span.set_attribute("payment.status", payment_result['status'])

# Trace context propagation to other services
def call_downstream_service(service_url, data):
    """Call downstream service with trace context"""
    headers = {}
    
    # Propagate trace context
    trace.get_tracer(__name__).get_current_span().get_span_context()
    
    # Add trace headers to request
    headers['X-Trace-ID'] = trace_id
    headers['X-Span-ID'] = span_id
    
    return requests.post(service_url, json=data, headers=headers)
```

---

# SECTION 67: ADVANCED QUERY PATTERNS AND OPTIMIZATION

### Q186: How do I implement query result streaming to handle large datasets efficiently?

1. Stream results instead of loading into memory.
2. Process rows as they arrive from database.
3. Implement pagination for API responses.
4. Monitor streaming query resource usage.
5. Set timeouts to prevent runaway queries.
6. Test streaming performance with large datasets.

```python
import psycopg2
from psycopg2.extras import RealDictCursor

class StreamingQueryProcessor:
    def __init__(self, batch_size=1000):
        self.batch_size = batch_size
    
    def stream_large_query(self, query, params=None, callback=None):
        """Stream query results without loading into memory"""
        conn = psycopg2.connect('dbname=mydb user=root')
        
        # Use server-side cursor for streaming
        cursor = conn.cursor('streaming_cursor', cursor_factory=RealDictCursor)
        cursor.itersize = self.batch_size
        
        try:
            cursor.execute(query, params or ())
            
            # Process results in batches
            batch = []
            for row in cursor:
                batch.append(row)
                
                if len(batch) >= self.batch_size:
                    if callback:
                        callback(batch)
                    else:
                        yield batch
                    batch = []
            
            # Process remaining rows
            if batch:
                if callback:
                    callback(batch)
                else:
                    yield batch
        
        finally:
            cursor.close()
            conn.close()

# Usage: Stream large export
def export_large_table_to_s3():
    processor = StreamingQueryProcessor(batch_size=5000)
    
    def write_batch_to_s3(batch):
        """Write batch to S3"""
        csv_data = convert_to_csv(batch)
        s3_client.put_object(
            Bucket='export-bucket',
            Key=f'export/batch_{uuid.uuid4()}.csv',
            Body=csv_data
        )
        print(f"Exported {len(batch)} rows")
    
    # Stream results directly to S3
    processor.stream_large_query(
        "SELECT * FROM large_table WHERE created_at > ?",
        [datetime.now() - timedelta(days=30)],
        callback=write_batch_to_s3
    )

# REST API with streaming
@app.route('/api/export/orders')
def export_orders():
    def generate():
        processor = StreamingQueryProcessor()
        
        for batch in processor.stream_large_query(
            "SELECT * FROM orders WHERE created_at > ? ORDER BY id",
            [request.args.get('since', '2024-01-01')]
        ):
            for row in batch:
                yield json.dumps(row) + '\n'
    
    return Response(generate(), mimetype='application/x-ndjson')
```

### Q187: How do I optimize GROUP BY and aggregation queries?

1. Add indexes on grouping columns.
2. Use covering indexes to avoid table access.
3. Denormalize frequently aggregated data.
4. Partition large tables before aggregating.
5. Use incremental aggregation for real-time dashboards.
6. Monitor aggregation query performance.

```sql
-- Index for GROUP BY optimization
CREATE INDEX idx_orders_status_amount ON orders(status, amount);

-- Optimize aggregation with index
EXPLAIN SELECT 
  status,
  COUNT(*) as order_count,
  SUM(amount) as total_amount,
  AVG(amount) as avg_amount
FROM orders
WHERE created_at > now() - interval '30 days'
GROUP BY status
ORDER BY total_amount DESC;

-- Materialized view for common aggregations
CREATE MATERIALIZED VIEW order_summary_daily AS
SELECT 
  DATE_TRUNC('day', created_at) as order_date,
  status,
  COUNT(*) as order_count,
  SUM(amount) as total_amount,
  COUNT(DISTINCT customer_id) as unique_customers
FROM orders
GROUP BY DATE_TRUNC('day', created_at), status;

-- Refresh schedule
CREATE SCHEDULE refresh_order_summary FOR 
  REFRESH MATERIALIZED VIEW order_summary_daily
  RECURRING EVERY 1 hour;

-- Query materialized view (much faster)
SELECT order_date, status, order_count, total_amount
FROM order_summary_daily
WHERE order_date > now() - interval '7 days'
ORDER BY order_date DESC;

-- Incremental aggregation for real-time metrics
CREATE TABLE daily_metrics (
  metric_date DATE PRIMARY KEY,
  total_orders INT,
  total_revenue DECIMAL,
  unique_customers INT,
  UNIQUE(metric_date)
);

-- Update incrementally instead of full recalculation
UPDATE daily_metrics
SET total_orders = (SELECT COUNT(*) FROM orders WHERE DATE_TRUNC('day', created_at) = CURRENT_DATE),
    total_revenue = (SELECT SUM(amount) FROM orders WHERE DATE_TRUNC('day', created_at) = CURRENT_DATE)
WHERE metric_date = CURRENT_DATE;
```

### Q188: How do I implement efficient DISTINCT queries for large datasets?

1. Use indexes on DISTINCT columns.
2. Partition data to reduce distinct values per partition.
3. Use DISTINCT ON for partial distinctness.
4. Implement sampling for approximate distinct counts.
5. Monitor DISTINCT query performance.
6. Use HyperLogLog approximation when appropriate.

```sql
-- Efficient DISTINCT with index
CREATE INDEX idx_customers_city ON customers(city);

-- DISTINCT query using index
SELECT DISTINCT city FROM customers WHERE created_at > now() - interval '30 days';

-- DISTINCT ON for partial distinctness
SELECT DISTINCT ON (customer_id) customer_id, order_date, amount
FROM orders
ORDER BY customer_id, order_date DESC;  -- Latest order per customer

-- Approximate distinct count (HyperLogLog)
SELECT 
  APPROX_DISTINCT(customer_id) as estimated_unique_customers,
  COUNT(DISTINCT customer_id) as exact_unique_customers
FROM orders
WHERE created_at > now() - interval '30 days';

-- Optimize with GROUP BY instead of DISTINCT
-- SLOW: SELECT DISTINCT customer_id FROM orders;
-- FAST: SELECT customer_id FROM orders GROUP BY customer_id;

-- Monitor DISTINCT performance
EXPLAIN SELECT DISTINCT city FROM customers;
```

### Q189: How do I optimize JOIN operations for large tables?

1. Identify join order: filter first, then join.
2. Create indexes on join columns.
3. Use hash joins for large tables.
4. Avoid implicit conversions in join predicates.
5. Monitor join performance metrics.
6. Consider denormalization for frequently joined tables.

```sql
-- Join order optimization (filter before joining)
-- SLOW: Large table joins before filtering
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > now() - interval '30 days'
  AND c.country = 'US';

-- FAST: Filter first, then join
WITH recent_orders AS (
  SELECT * FROM orders WHERE created_at > now() - interval '30 days'
),
us_customers AS (
  SELECT * FROM customers WHERE country = 'US'
)
SELECT ro.*, uc.name
FROM recent_orders ro
JOIN us_customers uc ON ro.customer_id = uc.id;

-- Create indexes for join columns
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_customers_id ON customers(id);

-- Join statistics
EXPLAIN ANALYZE
SELECT o.*, c.name, p.name as product_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE o.created_at > now() - interval '7 days';

-- Denormalize frequently joined data
CREATE MATERIALIZED VIEW order_details_denormalized AS
SELECT 
  o.id as order_id,
  o.customer_id,
  c.name as customer_name,
  c.email,
  o.product_id,
  p.name as product_name,
  p.price,
  o.amount,
  o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id;

-- Query denormalized view (much faster)
SELECT order_id, customer_name, product_name, amount
FROM order_details_denormalized
WHERE created_at > now() - interval '7 days';
```

### Q190: How do I optimize UNION and UNION ALL operations?

1. Use UNION ALL when duplicates acceptable (faster).
2. Add indexes on union columns.
3. Push filters to individual SELECT statements.
4. Partition data to reduce union scope.
5. Monitor union performance.
6. Consider materialized view for repeated unions.

```sql
-- Inefficient UNION (filters duplicates)
SELECT product_id FROM current_orders
UNION
SELECT product_id FROM archived_orders
WHERE archived_at > now() - interval '1 year';

-- Efficient UNION ALL (no duplicate filtering)
SELECT product_id FROM current_orders
UNION ALL
SELECT product_id FROM archived_orders
WHERE archived_at > now() - interval '1 year';

-- Push filters to individual queries
SELECT product_id FROM orders WHERE status = 'completed'
UNION ALL
SELECT product_id FROM archived_orders WHERE status = 'completed';

-- Materialized view for repeated unions
CREATE MATERIALIZED VIEW all_orders_combined AS
SELECT id, customer_id, product_id, amount, created_at, 'active' as source
FROM orders
UNION ALL
SELECT id, customer_id, product_id, amount, created_at, 'archived'
FROM archived_orders;

-- Query materialized view
SELECT product_id, COUNT(*) as order_count
FROM all_orders_combined
GROUP BY product_id
ORDER BY order_count DESC;
```

---

## SECTION 68: ADVANCED ANALYTICS AND REPORTING

### Q191: How do I implement data warehouse patterns with CockroachDB?

1. Separate OLTP and OLAP systems.
2. Use CDC to replicate OLTP data to warehouse.
3. Implement star schema for analytics.
4. Use facts and dimensions tables.
5. Denormalize for query performance.
6. Implement slowly changing dimensions (SCD).

```sql
-- Star schema for data warehouse
-- Fact table (large, contains foreign keys to dimensions)
CREATE TABLE fact_orders (
  order_id INT,
  customer_id INT,
  product_id INT,
  order_date_id INT,  -- Reference to date dimension
  order_amount DECIMAL,
  quantity INT,
  discount_percent DECIMAL
);

-- Dimension tables (small, descriptive)
CREATE TABLE dim_customers (
  customer_id INT PRIMARY KEY,
  name VARCHAR,
  city VARCHAR,
  country VARCHAR,
  segment VARCHAR,
  created_at TIMESTAMP
);

CREATE TABLE dim_products (
  product_id INT PRIMARY KEY,
  name VARCHAR,
  category VARCHAR,
  price DECIMAL,
  supplier_id INT
);

CREATE TABLE dim_dates (
  date_id INT PRIMARY KEY,  -- YYYYMMDD format
  full_date DATE,
  year INT,
  month INT,
  day INT,
  quarter INT,
  day_of_week VARCHAR,
  is_holiday BOOLEAN
);

-- Indexes for warehouse queries
CREATE INDEX idx_fact_customer ON fact_orders(customer_id);
CREATE INDEX idx_fact_product ON fact_orders(product_id);
CREATE INDEX idx_fact_date ON fact_orders(order_date_id);

-- Aggregate fact table for common queries
CREATE TABLE fact_orders_daily_summary (
  order_date_id INT,
  customer_segment VARCHAR,
  product_category VARCHAR,
  order_count INT,
  total_revenue DECIMAL,
  avg_order_value DECIMAL
);

-- Slowly Changing Dimension (Type 2 - track history)
CREATE TABLE dim_customers_scd (
  customer_id INT,
  customer_name VARCHAR,
  city VARCHAR,
  country VARCHAR,
  effective_from DATE,
  effective_to DATE,
  is_current BOOLEAN
);

-- Update SCD when customer changes
UPDATE dim_customers_scd
SET effective_to = CURRENT_DATE - 1,
    is_current = false
WHERE customer_id = 123 AND is_current = true;

INSERT INTO dim_customers_scd
VALUES (123, 'Updated Name', 'New City', 'Country', CURRENT_DATE, '9999-12-31', true);

-- Analytics query on star schema
SELECT 
  dc.segment,
  dp.category,
  COUNT(fo.order_id) as order_count,
  SUM(fo.order_amount) as total_revenue,
  AVG(fo.order_amount) as avg_order_value
FROM fact_orders fo
JOIN dim_customers dc ON fo.customer_id = dc.customer_id
JOIN dim_products dp ON fo.product_id = dp.product_id
WHERE fo.order_date_id >= 20240101
GROUP BY dc.segment, dp.category
ORDER BY total_revenue DESC;
```

### Q192: How do I implement real-time dashboards with CockroachDB?

1. Use materialized views for pre-computed metrics.
2. Implement incremental refresh schedules.
3. Use CDC for real-time updates.
4. Cache dashboard results with short TTL.
5. Implement drill-down capabilities.
6. Monitor dashboard query performance.

```sql
-- Real-time metrics view
CREATE MATERIALIZED VIEW dashboard_metrics AS
SELECT 
  NOW() as last_updated,
  (SELECT COUNT(*) FROM orders WHERE created_at > now() - interval '1 day') as orders_today,
  (SELECT SUM(amount) FROM orders WHERE created_at > now() - interval '1 day') as revenue_today,
  (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE created_at > now() - interval '1 day') as customers_today,
  (SELECT AVG(amount) FROM orders WHERE created_at > now() - interval '1 day') as avg_order_value_today;

-- Refresh frequently for real-time data
CREATE SCHEDULE refresh_dashboard FOR 
  REFRESH MATERIALIZED VIEW dashboard_metrics
  RECURRING EVERY 1 minute;

-- Time-series data for charts
CREATE MATERIALIZED VIEW revenue_by_hour AS
SELECT 
  DATE_TRUNC('hour', created_at) as hour,
  COUNT(*) as order_count,
  SUM(amount) as revenue,
  COUNT(DISTINCT customer_id) as unique_customers
FROM orders
WHERE created_at > now() - interval '7 days'
GROUP BY DATE_TRUNC('hour', created_at);

-- Create index for efficient refresh
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Application dashboard API
@app.route('/api/dashboard')
def get_dashboard():
    # Get cached dashboard data
    cache_key = 'dashboard_metrics'
    cached = cache.get(cache_key)
    
    if not cached:
        result = db.query_one("SELECT * FROM dashboard_metrics")
        cache.setex(cache_key, 60, json.dumps(result))
    else:
        result = json.loads(cached)
    
    return jsonify(result)

# Drill-down capability
@app.route('/api/dashboard/orders/detail')
def get_orders_detail():
    hour = request.args.get('hour')
    
    orders = db.query("""
      SELECT id, customer_id, amount, created_at
      FROM orders
      WHERE DATE_TRUNC('hour', created_at) = ?::timestamp
      ORDER BY created_at DESC
    """, hour)
    
    return jsonify(orders)
```

### Q193: How do I implement dimensional analysis and drill-down reports?

1. Design hierarchical dimensions (region -> country -> city).
2. Support drill-down by enabling multi-level navigation.
3. Implement breadcrumb navigation showing drill path.
4. Cache dimension hierarchies.
5. Monitor drill-down performance.
6. Document drill-down paths.

```sql
-- Hierarchical dimensions for drill-down
CREATE TABLE dim_geography (
  geography_id INT PRIMARY KEY,
  region VARCHAR,
  country VARCHAR,
  state_province VARCHAR,
  city VARCHAR,
  zip_code VARCHAR,
  level INT  -- 1=region, 2=country, 3=state, 4=city
);

-- Create indexes for drill-down navigation
CREATE INDEX idx_geography_region ON dim_geography(region);
CREATE INDEX idx_geography_country ON dim_geography(region, country);
CREATE INDEX idx_geography_state ON dim_geography(region, country, state_province);
CREATE INDEX idx_geography_city ON dim_geography(region, country, state_province, city);

-- Drill-down queries
-- Level 1: By Region
SELECT 
  region,
  COUNT(*) as order_count,
  SUM(order_amount) as total_revenue
FROM fact_orders fo
JOIN dim_geography dg ON fo.geography_id = dg.geography_id
GROUP BY region
ORDER BY total_revenue DESC;

-- Level 2: By Country (drill into region)
SELECT 
  country,
  COUNT(*) as order_count,
  SUM(order_amount) as total_revenue
FROM fact_orders fo
JOIN dim_geography dg ON fo.geography_id = dg.geography_id
WHERE dg.region = 'North America'
GROUP BY country
ORDER BY total_revenue DESC;

-- Level 3: By State (drill into country)
SELECT 
  state_province,
  COUNT(*) as order_count,
  SUM(order_amount) as total_revenue
FROM fact_orders fo
JOIN dim_geography dg ON fo.geography_id = dg.geography_id
WHERE dg.region = 'North America' AND dg.country = 'USA'
GROUP BY state_province
ORDER BY total_revenue DESC;

-- Create drill-down metadata
CREATE TABLE drill_down_paths (
  path_id INT PRIMARY KEY,
  dimension_name VARCHAR,
  level_name VARCHAR,
  parent_level VARCHAR,
  sql_template VARCHAR
);

INSERT INTO drill_down_paths VALUES 
  (1, 'geography', 'region', NULL, 'SELECT region FROM dim_geography GROUP BY region'),
  (2, 'geography', 'country', 'region', 'SELECT DISTINCT country FROM dim_geography WHERE region = ?'),
  (3, 'geography', 'state', 'country', 'SELECT DISTINCT state_province FROM dim_geography WHERE country = ?'),
  (4, 'geography', 'city', 'state', 'SELECT DISTINCT city FROM dim_geography WHERE state_province = ?');
```

### Q194: How do I implement cohort analysis for retention and churn metrics?

1. Create cohort tables tracking user groups by signup date.
2. Calculate retention rates over time.
3. Identify churn patterns.
4. Compare cohort performance.
5. Monitor cohort metrics regularly.
6. Visualize retention curves.

```sql
-- Cohort analysis table
CREATE TABLE user_cohorts (
  user_id INT,
  cohort_date DATE,  -- First signup month
  cohort_month INT,  -- 1=signup month, 2=one month later, etc.
  month_date DATE,
  was_active BOOLEAN,
  PRIMARY KEY (user_id, cohort_date, cohort_month)
);

-- Populate cohort data
INSERT INTO user_cohorts
SELECT 
  u.user_id,
  DATE_TRUNC('month', u.created_at)::DATE as cohort_date,
  ((EXTRACT(YEAR FROM o.order_date) - EXTRACT(YEAR FROM u.created_at)) * 12 +
   (EXTRACT(MONTH FROM o.order_date) - EXTRACT(MONTH FROM u.created_at))) + 1 as cohort_month,
  DATE_TRUNC('month', o.order_date)::DATE as month_date,
  true as was_active
FROM users u
LEFT JOIN orders o ON u.user_id = o.customer_id
GROUP BY u.user_id, DATE_TRUNC('month', u.created_at), DATE_TRUNC('month', o.order_date);

-- Calculate retention rate by cohort
CREATE MATERIALIZED VIEW cohort_retention_matrix AS
SELECT 
  cohort_date,
  cohort_month,
  COUNT(DISTINCT user_id) as users_in_cohort,
  ROUND(100.0 * COUNT(DISTINCT user_id) / 
    (SELECT COUNT(DISTINCT user_id) FROM user_cohorts WHERE cohort_month = 1 AND cohort_date = cohort_date),
    2) as retention_percent
FROM user_cohorts
WHERE was_active = true
GROUP BY cohort_date, cohort_month
ORDER BY cohort_date DESC, cohort_month;

-- Cohort retention table (wide format for visualization)
SELECT 
  cohort_date,
  MAX(CASE WHEN cohort_month = 1 THEN users_in_cohort END) as month_0,
  MAX(CASE WHEN cohort_month = 2 THEN retention_percent END) as month_1_retention,
  MAX(CASE WHEN cohort_month = 3 THEN retention_percent END) as month_2_retention,
  MAX(CASE WHEN cohort_month = 4 THEN retention_percent END) as month_3_retention,
  MAX(CASE WHEN cohort_month = 5 THEN retention_percent END) as month_4_retention,
  MAX(CASE WHEN cohort_month = 6 THEN retention_percent END) as month_5_retention
FROM cohort_retention_matrix
GROUP BY cohort_date
ORDER BY cohort_date DESC;

-- Churn analysis
CREATE VIEW churn_analysis AS
SELECT 
  u.user_id,
  u.created_at,
  MAX(o.created_at) as last_order_date,
  DATE_PART('day', now() - MAX(o.created_at)) as days_since_last_order,
  CASE 
    WHEN DATE_PART('day', now() - MAX(o.created_at)) > 90 THEN 'churned'
    WHEN DATE_PART('day', now() - MAX(o.created_at)) > 30 THEN 'at_risk'
    ELSE 'active'
  END as churn_status
FROM users u
LEFT JOIN orders o ON u.user_id = o.customer_id
GROUP BY u.user_id, u.created_at;

-- Monitor churn by cohort
SELECT 
  DATE_TRUNC('month', u.created_at)::DATE as signup_cohort,
  COUNT(*) FILTER (WHERE churn_status = 'churned') as churned_users,
  COUNT(*) as total_users,
  ROUND(100.0 * COUNT(*) FILTER (WHERE churn_status = 'churned') / COUNT(*), 2) as churn_rate
FROM churn_analysis ca
JOIN users u ON ca.user_id = u.user_id
GROUP BY DATE_TRUNC('month', u.created_at)::DATE
ORDER BY signup_cohort DESC;
```

### Q195: How do I implement funnel analysis for conversion tracking?

1. Define funnel steps (visit -> browse -> add to cart -> purchase).
2. Track user progression through funnel.
3. Calculate conversion rates between steps.
4. Identify drop-off points.
5. Segment by user characteristics.
6. Monitor funnel performance over time.

```sql
-- Funnel tracking events
CREATE TABLE funnel_events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id INT,
  event_type VARCHAR,  -- page_visit, product_view, add_to_cart, purchase
  event_timestamp TIMESTAMP DEFAULT now(),
  product_id INT,
  order_id INT,
  session_id VARCHAR
);

-- Calculate funnel conversions
CREATE VIEW funnel_analysis AS
WITH funnel_steps AS (
  -- Step 1: Page visits (starting point)
  SELECT user_id, session_id, event_timestamp as step_1_time
  FROM funnel_events
  WHERE event_type = 'page_visit'
  
  UNION ALL
  
  -- Step 2: Product views (qualified leads)
  SELECT user_id, session_id, event_timestamp as step_2_time
  FROM funnel_events
  WHERE event_type = 'product_view'
  
  UNION ALL
  
  -- Step 3: Add to cart (interested)
  SELECT user_id, session_id, event_timestamp as step_3_time
  FROM funnel_events
  WHERE event_type = 'add_to_cart'
  
  UNION ALL
  
  -- Step 4: Purchase (conversion)
  SELECT user_id, session_id, event_timestamp as step_4_time
  FROM funnel_events
  WHERE event_type = 'purchase'
)
SELECT 
  session_id,
  user_id,
  CASE 
    WHEN step_4_time IS NOT NULL THEN 'converted'
    WHEN step_3_time IS NOT NULL THEN 'added_to_cart'
    WHEN step_2_time IS NOT NULL THEN 'viewed_product'
    WHEN step_1_time IS NOT NULL THEN 'visited'
    ELSE 'unknown'
  END as funnel_stage
FROM funnel_steps;

-- Funnel conversion rates
SELECT 
  COUNT(*) as total_visits,
  COUNT(*) FILTER (WHERE funnel_stage IN ('viewed_product', 'added_to_cart', 'converted')) as product_viewers,
  COUNT(*) FILTER (WHERE funnel_stage IN ('added_to_cart', 'converted')) as cart_users,
  COUNT(*) FILTER (WHERE funnel_stage = 'converted') as converters,
  ROUND(100.0 * COUNT(*) FILTER (WHERE funnel_stage IN ('viewed_product', 'added_to_cart', 'converted')) / COUNT(*), 2) as view_rate,
  ROUND(100.0 * COUNT(*) FILTER (WHERE funnel_stage IN ('added_to_cart', 'converted')) / COUNT(*), 2) as cart_rate,
  ROUND(100.0 * COUNT(*) FILTER (WHERE funnel_stage = 'converted') / COUNT(*), 2) as conversion_rate
FROM funnel_analysis;

-- Funnel by segment
SELECT 
  DATE_TRUNC('day', event_timestamp)::DATE as date,
  COUNT(*) as visits,
  COUNT(*) FILTER (WHERE funnel_stage IN ('viewed_product', 'added_to_cart', 'converted')) as viewers,
  COUNT(*) FILTER (WHERE funnel_stage = 'converted') as purchases,
  ROUND(100.0 * COUNT(*) FILTER (WHERE funnel_stage = 'converted') / COUNT(*), 2) as conversion_rate
FROM funnel_analysis
GROUP BY DATE_TRUNC('day', event_timestamp)::DATE
ORDER BY date DESC;

-- Identify drop-off points
WITH step_counts AS (
  SELECT 
    'Step 1: Page Visits' as step,
    COUNT(*) as count
  FROM funnel_events
  WHERE event_type = 'page_visit'
  
  UNION ALL
  
  SELECT 'Step 2: Product Views', COUNT(*)
  FROM funnel_events
  WHERE event_type = 'product_view'
  
  UNION ALL
  
  SELECT 'Step 3: Add to Cart', COUNT(*)
  FROM funnel_events
  WHERE event_type = 'add_to_cart'
  
  UNION ALL
  
  SELECT 'Step 4: Purchase', COUNT(*)
  FROM funnel_events
  WHERE event_type = 'purchase'
)
SELECT 
  step,
  count,
  LAG(count) OVER (ORDER BY CASE WHEN step = 'Step 1: Page Visits' THEN 1
                                    WHEN step = 'Step 2: Product Views' THEN 2
                                    WHEN step = 'Step 3: Add to Cart' THEN 3
                                    ELSE 4 END) as previous_step_count,
  ROUND(100.0 * count / LAG(count) OVER (ORDER BY CASE WHEN step = 'Step 1: Page Visits' THEN 1
                                                         WHEN step = 'Step 2: Product Views' THEN 2
                                                         WHEN step = 'Step 3: Add to Cart' THEN 3
                                                         ELSE 4 END), 2) as conversion_percent
FROM step_counts;
```

---

## SECTION 69: ADVANCED STATISTICAL ANALYSIS

### Q196: How do I implement statistical hypothesis testing in SQL?

1. Calculate test statistics for null hypothesis.
2. Compute p-values for significance testing.
3. Determine sample size for confidence levels.
4. Implement t-tests, chi-square tests, etc.
5. Report confidence intervals.
6. Document assumptions and limitations.

```sql
-- T-test: Compare means between two groups
CREATE VIEW t_test_comparison AS
WITH group_a AS (
  SELECT amount FROM orders WHERE variant = 'A'
),
group_b AS (
  SELECT amount FROM orders WHERE variant = 'B'
),
group_stats AS (
  SELECT 
    'A' as group_name,
    COUNT(*) as n,
    AVG(amount) as mean,
    STDDEV(amount) as stddev
  FROM group_a
  
  UNION ALL
  
  SELECT 
    'B',
    COUNT(*),
    AVG(amount),
    STDDEV(amount)
  FROM group_b
)
SELECT 
  a.group_name as group_a,
  b.group_name as group_b,
  a.mean - b.mean as mean_difference,
  SQRT(POWER(a.stddev, 2) / a.n + POWER(b.stddev, 2) / b.n) as standard_error,
  (a.mean - b.mean) / SQRT(POWER(a.stddev, 2) / a.n + POWER(b.stddev, 2) / b.n) as t_statistic,
  CASE 
    WHEN ABS((a.mean - b.mean) / SQRT(POWER(a.stddev, 2) / a.n + POWER(b.stddev, 2) / b.n)) > 1.96 
      THEN 'Significant (p < 0.05)'
    ELSE 'Not Significant'
  END as result
FROM group_stats a, group_stats b
WHERE a.group_name = 'A' AND b.group_name = 'B';

-- Chi-square test: Independence of categorical variables
CREATE VIEW chi_square_test AS
WITH contingency_table AS (
  SELECT 
    variant,
    CASE WHEN converted = true THEN 'converted' ELSE 'not_converted' END as outcome,
    COUNT(*) as count
  FROM ab_test_results
  GROUP BY variant, outcome
),
totals AS (
  SELECT 
    SUM(CASE WHEN outcome = 'converted' THEN count ELSE 0 END) as total_converted,
    SUM(CASE WHEN outcome = 'not_converted' THEN count ELSE 0 END) as total_not_converted,
    SUM(count) as grand_total
  FROM contingency_table
)
SELECT 
  SUM(POWER(count - expected_count, 2) / expected_count) as chi_square_statistic
FROM contingency_table ct
CROSS JOIN LATERAL (
  SELECT 
    row_total * col_total / grand_total as expected_count
  FROM (
    SELECT 
      SUM(CASE WHEN ct2.variant = ct.variant THEN count ELSE 0 END) as row_total,
      SUM(CASE WHEN ct2.outcome = ct.outcome THEN count ELSE 0 END) as col_total,
      SUM(count) as grand_total
    FROM contingency_table ct2
  ) t
) expected;

-- Confidence interval for proportion
SELECT 
  COUNT(*) FILTER (WHERE converted = true) as conversions,
  COUNT(*) as total,
  ROUND(100.0 * COUNT(*) FILTER (WHERE converted = true) / COUNT(*), 2) as conversion_rate,
  ROUND(100.0 * (COUNT(*) FILTER (WHERE converted = true) / COUNT(*) - 
    1.96 * SQRT((COUNT(*) FILTER (WHERE converted = true) / COUNT(*) * 
              (1 - COUNT(*) FILTER (WHERE converted = true) / COUNT(*))) / COUNT(*))), 2) as ci_lower,
  ROUND(100.0 * (COUNT(*) FILTER (WHERE converted = true) / COUNT(*) + 
    1.96 * SQRT((COUNT(*) FILTER (WHERE converted = true) / COUNT(*) * 
              (1 - COUNT(*) FILTER (WHERE converted = true) / COUNT(*))) / COUNT(*))), 2) as ci_upper
FROM ab_test_results
WHERE variant = 'A';

-- Required sample size for A/B test
-- For 95% confidence, 80% power, baseline rate 10%, expected uplift to 12%
SELECT 
  POWER(1.96 * SQRT(0.1 * 0.9 + 0.12 * 0.88) / (0.12 - 0.1), 2) as required_sample_per_group;
```

### Q197: How do I implement time-series decomposition and trend analysis?

1. Decompose time-series into trend, seasonality, residual.
2. Identify seasonal patterns.
3. Forecast future values.
4. Detect anomalies in trends.
5. Monitor trend changes.
6. Document seasonal adjustments.

```sql
-- Time-series decomposition
CREATE MATERIALIZED VIEW timeseries_decomposition AS
WITH daily_metrics AS (
  SELECT 
    DATE_TRUNC('day', created_at)::DATE as date,
    COUNT(*) as orders,
    SUM(amount) as revenue
  FROM orders
  WHERE created_at > now() - interval '2 years'
  GROUP BY DATE_TRUNC('day', created_at)::DATE
),
-- Calculate 7-day moving average (trend)
trend AS (
  SELECT 
    date,
    orders,
    AVG(orders) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as trend_value
  FROM daily_metrics
),
-- Calculate detrended values
detrended AS (
  SELECT 
    date,
    orders,
    trend_value,
    orders - trend_value as detrended_value,
    EXTRACT(DOW FROM date) as day_of_week
  FROM trend
),
-- Calculate seasonal component (average by day of week)
seasonality AS (
  SELECT 
    date,
    orders,
    trend_value,
    detrended_value,
    AVG(detrended_value) OVER (PARTITION BY EXTRACT(DOW FROM date)) as seasonal_component
  FROM detrended
)
SELECT 
  date,
  orders as actual,
  ROUND(trend_value::numeric, 2) as trend,
  ROUND(seasonal_component::numeric, 2) as seasonality,
  ROUND((orders - trend_value - seasonal_component)::numeric, 2) as residual
FROM seasonality
WHERE trend_value IS NOT NULL
ORDER BY date;

-- Trend analysis (linear regression)
CREATE VIEW trend_analysis AS
WITH daily_orders AS (
  SELECT 
    ROW_NUMBER() OVER (ORDER BY DATE_TRUNC('day', created_at)::DATE) as day_num,
    DATE_TRUNC('day', created_at)::DATE as date,
    COUNT(*) as orders
  FROM orders
  WHERE created_at > now() - interval '1 year'
  GROUP BY DATE_TRUNC('day', created_at)::DATE
),
regression_data AS (
  SELECT 
    day_num,
    date,
    orders,
    AVG(day_num) OVER () as avg_x,
    AVG(orders) OVER () as avg_y,
    SUM((day_num - AVG(day_num) OVER ()) * (orders - AVG(orders) OVER ())) 
      OVER () as numerator,
    SUM(POWER(day_num - AVG(day_num) OVER (), 2)) OVER () as denominator
  FROM daily_orders
)
SELECT DISTINCT 
  numerator / denominator as slope,
  avg_y - (numerator / denominator) * avg_x as intercept,
  CASE 
    WHEN numerator / denominator > 0 THEN 'Increasing'
    WHEN numerator / denominator < 0 THEN 'Decreasing'
    ELSE 'Flat'
  END as trend_direction
FROM regression_data;

-- Anomaly detection (values > 2 standard deviations)
SELECT 
  date,
  orders,
  trend_value,
  seasonal_component,
  residual,
  STDDEV(residual) OVER () as residual_stddev,
  CASE 
    WHEN ABS(residual) > 2 * STDDEV(residual) OVER () THEN 'ANOMALY'
    ELSE 'NORMAL'
  END as status
FROM timeseries_decomposition
WHERE residual IS NOT NULL
ORDER BY date DESC;

-- Forecast next 30 days
WITH trend_model AS (
  SELECT 
    numerator / denominator as slope,
    avg_y - (numerator / denominator) * avg_x as intercept
  FROM (
    SELECT 
      ROW_NUMBER() OVER (ORDER BY DATE_TRUNC('day', created_at)::DATE) as day_num,
      AVG(COUNT(*)) OVER () as avg_y,
      AVG(ROW_NUMBER() OVER (ORDER BY DATE_TRUNC('day', created_at)::DATE)) OVER () as avg_x,
      SUM((ROW_NUMBER() OVER () - AVG(ROW_NUMBER() OVER ()) OVER ()) * 
          (COUNT(*) - AVG(COUNT(*)) OVER ())) OVER () as numerator,
      SUM(POWER(ROW_NUMBER() OVER () - AVG(ROW_NUMBER() OVER ()) OVER (), 2)) OVER () as denominator
    FROM orders
    WHERE created_at > now() - interval '1 year'
    GROUP BY DATE_TRUNC('day', created_at)::DATE
  ) regression
)
SELECT 
  CURRENT_DATE + (generate_series::int || ' days')::interval as forecast_date,
  ROUND((slope * (365 + generate_series) + intercept)::numeric, 0) as forecasted_orders
FROM trend_model,
     generate_series(1, 30)
ORDER BY forecast_date;
```

---

## SECTION 70: ADVANCED OPERATIONAL METRICS

### Q198: How do I implement SLA tracking and compliance reporting?

1. Define SLA metrics and targets.
2. Track actual performance against targets.
3. Calculate SLA compliance percentage.
4. Generate monthly compliance reports.
5. Alert on SLA violations.
6. Document SLA exceptions.

```sql
-- SLA definitions
CREATE TABLE sla_definitions (
  sla_id INT PRIMARY KEY,
  sla_name VARCHAR,
  metric_type VARCHAR,  -- latency, availability, throughput
  target_value DECIMAL,
  threshold_lower DECIMAL,
  threshold_upper DECIMAL,
  measurement_frequency VARCHAR
);

INSERT INTO sla_definitions VALUES 
  (1, 'Query P99 Latency', 'latency', 100, 80, 120, 'per_minute'),
  (2, 'Cluster Availability', 'availability', 99.9, 99.8, 100, 'per_hour'),
  (3, 'Backup Completion', 'backup', 100, 95, 100, 'daily');

-- Track SLA metrics
CREATE TABLE sla_metrics (
  metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sla_id INT REFERENCES sla_definitions(sla_id),
  measured_value DECIMAL,
  target_value DECIMAL,
  measured_at TIMESTAMP DEFAULT now(),
  is_compliant BOOLEAN
);

-- Collect metrics (via scheduled job or monitoring agent)
INSERT INTO sla_metrics (sla_id, measured_value, target_value, is_compliant)
SELECT 
  1 as sla_id,
  (SELECT PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency) FROM query_metrics 
   WHERE timestamp > now() - interval '1 minute') as measured_value,
  100 as target_value,
  ((SELECT PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency) FROM query_metrics 
    WHERE timestamp > now() - interval '1 minute') <= 100) as is_compliant;

-- Calculate SLA compliance percentage
CREATE VIEW monthly_sla_compliance AS
SELECT 
  sd.sla_name,
  DATE_TRUNC('month', sm.measured_at)::DATE as month,
  COUNT(*) as total_measurements,
  COUNT(*) FILTER (WHERE is_compliant = true) as compliant_measurements,
  ROUND(100.0 * COUNT(*) FILTER (WHERE is_compliant = true) / COUNT(*), 2) as compliance_percent,
  CASE 
    WHEN ROUND(100.0 * COUNT(*) FILTER (WHERE is_compliant = true) / COUNT(*), 2) >= sd.target_value 
      THEN 'PASS'
    ELSE 'FAIL'
  END as sla_status
FROM sla_metrics sm
JOIN sla_definitions sd ON sm.sla_id = sd.sla_id
GROUP BY sd.sla_name, DATE_TRUNC('month', sm.measured_at)::DATE, sd.target_value
ORDER BY month DESC, sd.sla_name;

-- SLA violation alert
CREATE TABLE sla_violations (
  violation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sla_id INT REFERENCES sla_definitions(sla_id),
  violation_time TIMESTAMP DEFAULT now(),
  measured_value DECIMAL,
  target_value DECIMAL,
  variance_percent DECIMAL,
  severity VARCHAR,  -- CRITICAL, WARNING, INFO
  resolved_at TIMESTAMP
);

-- Insert SLA violations
INSERT INTO sla_violations (sla_id, measured_value, target_value, variance_percent, severity)
SELECT 
  sm.sla_id,
  sm.measured_value,
  sm.target_value,
  ROUND(((sm.measured_value - sm.target_value) / sm.target_value) * 100, 2),
  CASE 
    WHEN sm.measured_value > sm.target_value * 1.5 THEN 'CRITICAL'
    WHEN sm.measured_value > sm.target_value * 1.2 THEN 'WARNING'
    ELSE 'INFO'
  END
FROM sla_metrics sm
WHERE is_compliant = false
  AND NOT EXISTS (SELECT 1 FROM sla_violations sv WHERE sv.sla_id = sm.sla_id);

-- Monthly SLA report
SELECT 
  DATE_TRUNC('month', now())::DATE as report_month,
  COUNT(*) as total_slas,
  COUNT(*) FILTER (WHERE compliance_percent >= target_value) as compliant_slas,
  AVG(compliance_percent) as avg_compliance,
  ROUND(100.0 * COUNT(*) FILTER (WHERE compliance_percent >= target_value) / COUNT(*), 2) as overall_compliance
FROM monthly_sla_compliance
WHERE month = DATE_TRUNC('month', now())::DATE;
```

### Q199: How do I implement comprehensive operational dashboards for different audiences?

1. Executive dashboard: Business KPIs only.
2. Operations dashboard: System health metrics.
3. Engineering dashboard: Technical details and logs.
4. Finance dashboard: Cost and resource utilization.
5. Update frequency appropriate for audience.
6. Drill-down capabilities for deep dives.

```sql
-- Executive Dashboard (Business-focused)
CREATE MATERIALIZED VIEW exec_dashboard AS
SELECT 
  DATE_TRUNC('day', now())::DATE as report_date,
  (SELECT COUNT(*) FROM orders WHERE created_at > now() - interval '24 hours') as orders_24h,
  (SELECT SUM(amount) FROM orders WHERE created_at > now() - interval '24 hours') as revenue_24h,
  (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE created_at > now() - interval '24 hours') as new_customers_24h,
  (SELECT AVG(amount) FROM orders WHERE created_at > now() - interval '24 hours') as avg_order_value_24h,
  (SELECT COUNT(*) FILTER (WHERE status = 'completed') FROM orders 
   WHERE created_at > now() - interval '24 hours') as completed_orders_24h,
  (SELECT COUNT(*) FILTER (WHERE status = 'failed') FROM orders 
   WHERE created_at > now() - interval '24 hours') as failed_orders_24h;

-- Operations Dashboard (System health)
CREATE MATERIALIZED VIEW ops_dashboard AS
SELECT 
  (SELECT COUNT(*) FROM crdb_internal.nodes WHERE is_live = true) as live_nodes,
  (SELECT COUNT(*) FROM crdb_internal.nodes) as total_nodes,
  (SELECT COUNT(*) FROM crdb_internal.ranges WHERE unavailable_replicas > 0) as unhealthy_ranges,
  (SELECT MAX(capacity_used::float / capacity_available) FROM crdb_internal.stores) as max_disk_utilization,
  (SELECT AVG(sql_exec_latency_p99) FROM crdb_internal.node_metrics) as avg_p99_latency_ms,
  (SELECT SUM(ranges_writes_per_second) FROM crdb_internal.node_metrics) as total_writes_per_sec,
  (SELECT COUNT(*) FROM system.jobs WHERE status = 'failed') as failed_jobs;

-- Engineering Dashboard (Technical details)
CREATE MATERIALIZED VIEW eng_dashboard AS
SELECT 
  (SELECT COUNT(*) FROM system.event_log WHERE timestamp > now() - interval '1 hour') as events_last_hour,
  (SELECT COUNT(*) FROM system.event_log WHERE event_type = 'error' AND timestamp > now() - interval '1 hour') as errors_last_hour,
  (SELECT COUNT(*) FROM crdb_internal.active_transactions WHERE elapsed_time > interval '5 minutes') as long_transactions,
  (SELECT COUNT(*) FROM system.statement_statistics WHERE latency_p99 > 1000) as slow_queries,
  (SELECT COUNT(DISTINCT job_id) FROM system.jobs WHERE status = 'running') as active_jobs;

-- Finance Dashboard (Cost tracking)
CREATE MATERIALIZED VIEW finance_dashboard AS
SELECT 
  DATE_TRUNC('month', now())::DATE as billing_month,
  (SELECT COUNT(*) FROM crdb_internal.nodes) * 730 * 1.00 as compute_cost_estimate,  -- $1/vCPU-hour
  (SELECT SUM(capacity_available) FROM crdb_internal.stores) / 1024 / 1024 / 1024 * 0.023 as storage_cost_estimate,
  ((SELECT COUNT(*) FROM crdb_internal.nodes) * 730 * 1.00) +
  ((SELECT SUM(capacity_available) FROM crdb_internal.stores) / 1024 / 1024 / 1024 * 0.023) as total_monthly_cost,
  (SELECT COUNT(*) FROM orders WHERE created_at > now() - interval '1 month') as monthly_orders,
  ROUND((((SELECT COUNT(*) FROM crdb_internal.nodes) * 730 * 1.00) +
    ((SELECT SUM(capacity_available) FROM crdb_internal.stores) / 1024 / 1024 / 1024 * 0.023)) /
    (SELECT COUNT(*) FROM orders WHERE created_at > now() - interval '1 month'), 4) as cost_per_order;

-- Refresh schedules
CREATE SCHEDULE exec_dash_refresh FOR REFRESH MATERIALIZED VIEW exec_dashboard RECURRING EVERY 1 hour;
CREATE SCHEDULE ops_dash_refresh FOR REFRESH MATERIALIZED VIEW ops_dashboard RECURRING EVERY 5 minutes;
CREATE SCHEDULE eng_dash_refresh FOR REFRESH MATERIALIZED VIEW eng_dashboard RECURRING EVERY 1 minute;
CREATE SCHEDULE finance_dash_refresh FOR REFRESH MATERIALIZED VIEW finance_dashboard RECURRING EVERY 1 day;
```

### Q200: How do I implement end-to-end performance monitoring across application stack?

1. Instrument application code for tracing.
2. Correlate database metrics with application metrics.
3. Implement end-to-end request tracking.
4. Visualize bottlenecks across layers.
5. Monitor infrastructure metrics.
6. Alert on performance degradation.

```python
import time
import logging
from opentelemetry import trace, metrics
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from prometheus_client import start_http_server

# Initialize metrics collection
prometheus_reader = PrometheusMetricReader()
meter_provider = MeterProvider(metric_readers=[prometheus_reader])
meter = meter_provider.get_meter(__name__)

# Create metric instruments
request_duration = meter.create_histogram(
    name="http_request_duration_seconds",
    description="HTTP request duration",
)

db_query_duration = meter.create_histogram(
    name="database_query_duration_seconds",
    description="Database query duration",
)

database_connections = meter.create_up_down_counter(
    name="database_connections",
    description="Active database connections",
)

class PerformanceMonitor:
    def __init__(self):
        self.tracer = trace.get_tracer(__name__)
        self.logger = logging.getLogger(__name__)
    
    def monitor_request(self, func):
        """Decorator to monitor HTTP request performance"""
        def wrapper(*args, **kwargs):
            start_time = time.time()
            
            with self.tracer.start_as_current_span("http_request") as span:
                try:
                    result = func(*args, **kwargs)
                    elapsed = time.time() - start_time
                    
                    span.set_attribute("http.status_code", 200)
                    span.set_attribute("http.duration_ms", elapsed * 1000)
                    
                    request_duration.record(elapsed)
                    
                    self.logger.info(f"Request {func.__name__} completed in {elapsed:.3f}s")
                    return result
                
                except Exception as e:
                    elapsed = time.time() - start_time
                    span.set_attribute("http.status_code", 500)
                    span.set_attribute("error", str(e))
                    request_duration.record(elapsed)
                    raise
        
        return wrapper
    
    def monitor_database_query(self, query_name):
        """Decorator to monitor database query performance"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start_time = time.time()
                database_connections.add(1)
                
                with self.tracer.start_as_current_span("database_query") as span:
                    try:
                        span.set_attribute("db.query_name", query_name)
                        result = func(*args, **kwargs)
                        elapsed = time.time() - start_time
                        
                        span.set_attribute("db.duration_ms", elapsed * 1000)
                        db_query_duration.record(elapsed)
                        
                        self.logger.debug(f"Query {query_name} executed in {elapsed:.3f}s")
                        return result
                    
                    finally:
                        database_connections.add(-1)
            
            return wrapper
        return decorator

# Usage
monitor = PerformanceMonitor()

@monitor.monitor_request
def handle_order_request(order_id):
    """Handle order request end-to-end"""
    
    # Fetch customer
    customer = get_customer_data(order_id)
    
    # Fetch order details
    order_details = get_order_details(order_id)
    
    # Process payment
    payment_result = process_payment(order_id, order_details['amount'])
    
    # Update inventory
    update_inventory(order_details['product_id'], order_details['quantity'])
    
    return {'status': 'success', 'order_id': order_id}

@monitor.monitor_database_query("get_customer_data")
def get_customer_data(customer_id):
    """Fetch customer data with monitoring"""
    return db.query_one("SELECT * FROM customers WHERE id = ?", customer_id)

# Start Prometheus metrics server
start_http_server(8000)

# Application runs with full observability
```

---

## SECTION 71: FINAL OPTIMIZATION TECHNIQUES

### Q201: How do I implement connection pooling optimization?

1. Size connection pool to match workload.
2. Monitor pool utilization and contention.
3. Implement idle timeout to release unused connections.
4. Use connection pooling at application layer.
5. Implement circuit breakers for pool exhaustion.
6. Test under peak load.

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool
import psycopg2
from psycopg2 import pool

# SQLAlchemy connection pooling
engine = create_engine(
    'postgresql://user:pass@localhost:26257/mydb',
    poolclass=QueuePool,
    pool_size=20,  # Keep 20 connections open
    max_overflow=40,  # Allow up to 40 additional connections
    pool_recycle=3600,  # Recycle connections after 1 hour
    pool_pre_ping=True,  # Test connection before using
    echo_pool=False,  # Log pool operations
)

# PgBouncer configuration for connection multiplexing
pgbouncer_config = """
[databases]
mydb = host=localhost port=26257 user=root password=secret

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3

statement_lifetime = 0
idle_in_transaction_session_timeout = 600
"""

# Application connection pool monitoring
class ConnectionPoolMonitor:
    def __init__(self, engine):
        self.engine = engine
        self.pool = engine.pool
    
    def get_pool_status(self):
        """Get current pool status"""
        return {
            'size': self.pool.size(),
            'checked_out': len(self.pool._queue.queue) if hasattr(self.pool, '_queue') else 0,
            'overflow': self.pool.overflow(),
            'total_connections': self.pool.size() + self.pool.overflow()
        }
    
    def monitor_pool_health(self):
        """Monitor pool utilization"""
        status = self.get_pool_status()
        
        utilization_percent = (status['total_connections'] / 
                              (self.pool.size() + self.pool.overflow())) * 100
        
        if utilization_percent > 80:
            self.alert_high_pool_utilization(status)
        
        return status
    
    def alert_high_pool_utilization(self, status):
        """Alert when pool utilization is high"""
        logging.warning(f"High connection pool utilization: {status}")

# Usage
monitor = ConnectionPoolMonitor(engine)

# Periodic monitoring
import threading
def monitor_pool():
    while True:
        status = monitor.monitor_pool_health()
        time.sleep(60)

threading.Thread(target=monitor_pool, daemon=True).start()
```

### Q202: How do I optimize for workloads with uneven query patterns?

1. Identify query patterns and their frequency.
2. Optimize for most common patterns.
3. Cache results for repeated queries.
4. Use query result reuse techniques.
5. Implement adaptive optimization.
6. Monitor pattern changes.

```sql
-- Identify most common queries
SELECT 
  query,
  COUNT(*) as execution_count,
  AVG(latency) as avg_latency_ms,
  MAX(latency) as max_latency_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency) as p99_latency_ms,
  ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM crdb_internal.node_statement_statistics), 2) as percent_of_total
FROM crdb_internal.node_statement_statistics
GROUP BY query
ORDER BY execution_count DESC
LIMIT 20;

-- Optimize top queries
-- 1. Add indexes for most frequent
-- 2. Implement caching for slow but frequent
-- 3. Denormalize if necessary

-- Result reuse (for identical queries within time window)
CREATE TABLE query_result_cache (
  query_hash VARCHAR PRIMARY KEY,
  query_text VARCHAR,
  result_data BYTEA,
  result_row_count INT,
  cached_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP,
  hit_count INT DEFAULT 0
);

-- Application-level result reuse
import hashlib
import json
import pickle

class QueryResultCache:
    def __init__(self, ttl_seconds=300):
        self.ttl = ttl_seconds
    
    def get_query_hash(self, query, params):
        """Generate hash of query and parameters"""
        content = f"{query}:{json.dumps(params)}"
        return hashlib.md5(content.encode()).hexdigest()
    
    def get_cached_result(self, query, params):
        """Get cached result if available"""
        query_hash = self.get_query_hash(query, params)
        
        cached = db.query_one(
            "SELECT result_data, expires_at FROM query_result_cache "
            "WHERE query_hash = ? AND expires_at > now()",
            query_hash
        )
        
        if cached:
            # Increment hit counter
            db.execute(
                "UPDATE query_result_cache SET hit_count = hit_count + 1 "
                "WHERE query_hash = ?",
                query_hash
            )
            return pickle.loads(cached['result_data'])
        
        return None
    
    def cache_result(self, query, params, result):
        """Cache query result"""
        query_hash = self.get_query_hash(query, params)
        result_data = pickle.dumps(result)
        
        db.execute(
            "INSERT INTO query_result_cache (query_hash, query_text, result_data, expires_at) "
            "VALUES (?, ?, ?, now() + interval '?') "
            "ON CONFLICT (query_hash) DO UPDATE SET result_data = ?, expires_at = now() + interval '?'",
            query_hash, query, result_data, f"{self.ttl} seconds",
            result_data, f"{self.ttl} seconds"
        )
    
    def execute_with_cache(self, query, params):
        """Execute query with result caching"""
        # Check cache first
        cached_result = self.get_cached_result(query, params)
        if cached_result:
            return cached_result
        
        # Execute query
        result = db.query(query, params)
        
        # Cache result
        self.cache_result(query, params, result)
        
        return result
```

### Q203: How do I implement graceful query degradation under load?

1. Monitor query queue depth.
2. Implement priority levels for queries.
3. Queue low-priority queries during load.
4. Use approximate results when exact too expensive.
5. Implement timeouts and cancellation.
6. Serve stale data if current unavailable.

```python
import queue
import threading
from enum import Enum

class QueryPriority(Enum):
    CRITICAL = 1  # P99 < 10ms
    HIGH = 2      # P99 < 100ms
    NORMAL = 3    # P99 < 1000ms
    LOW = 4       # Best effort

class QueryQueueManager:
    def __init__(self, max_queue_depth=1000, load_threshold=0.8):
        self.priority_queues = {
            QueryPriority.CRITICAL: queue.Queue(maxsize=100),
            QueryPriority.HIGH: queue.Queue(maxsize=200),
            QueryPriority.NORMAL: queue.Queue(maxsize=500),
            QueryPriority.LOW: queue.Queue(maxsize=1000),
        }
        self.load_threshold = load_threshold
        self.current_load = 0
    
    def get_current_load(self):
        """Get current system load (CPU utilization)"""
        # Implementation: query monitoring system
        return self.current_load
    
    def submit_query(self, query, params, priority=QueryPriority.NORMAL, timeout=None):
        """Submit query with priority and load-aware handling"""
        current_load = self.get_current_load()
        
        # Under high load, degrade lower-priority queries
        if current_load > self.load_threshold:
            if priority == QueryPriority.LOW:
                # Reject or defer low-priority queries
                raise Exception("System overloaded, please retry low-priority query")
            elif priority == QueryPriority.NORMAL:
                # Use approximate results for normal queries
                return self.get_approximate_result(query, params)
        
        # Queue query by priority
        try:
            self.priority_queues[priority].put(
                (query, params, timeout),
                block=True,
                timeout=1
            )
        except queue.Full:
            if priority == QueryPriority.LOW:
                raise Exception("Query queue full")
            else:
                # Critical queries bypass queue
                return self.execute_critical_query(query, params, timeout)
    
    def get_approximate_result(self, query, params):
        """Return approximate result for non-critical queries"""
        # Use sampling or pre-computed estimates
        approx_query = query.replace("COUNT(*)", "APPROX_COUNT(*)")
        return db.query(approx_query, params)
    
    def execute_critical_query(self, query, params, timeout):
        """Execute critical query with priority handling"""
        try:
            return db.query(query, params, timeout=timeout)
        except Exception as e:
            logging.error(f"Critical query failed: {e}")
            raise
    
    def process_queue(self):
        """Process queued queries by priority"""
        while True:
            # Process in priority order
            for priority in [QueryPriority.CRITICAL, QueryPriority.HIGH, 
                           QueryPriority.NORMAL, QueryPriority.LOW]:
                try:
                    query, params, timeout = self.priority_queues[priority].get_nowait()
                    db.query(query, params, timeout=timeout)
                except queue.Empty:
                    continue
            
            time.sleep(0.1)

# Serve stale data under extreme load
class StaleCacheManager:
    def __init__(self, freshness_threshold_seconds=60):
        self.cache = {}
        self.freshness_threshold = freshness_threshold_seconds
    
    def get_data(self, query, params, allow_stale=False):
        """Get fresh data, or stale if fresh unavailable"""
        cache_key = f"{query}:{params}"
        
        try:
            # Try to get fresh data (with timeout)
            result = db.query(query, params, timeout=1)
            
            # Cache result
            self.cache[cache_key] = {
                'data': result,
                'timestamp': time.time(),
                'is_fresh': True
            }
            
            return result
        
        except Exception as e:
            if allow_stale and cache_key in self.cache:
                cached = self.cache[cache_key]
                age = time.time() - cached['timestamp']
                
                if age < self.freshness_threshold * 10:  # Allow very stale data
                    logging.warning(f"Serving stale data ({age:.0f}s old) for query")
                    return cached['data']
            
            raise
```

---

## SECTION 72: ADVANCED INCIDENT MANAGEMENT

### Q204: How do I implement postmortem and incident learning processes?

1. Document all incidents systematically.
2. Conduct blameless postmortems.
3. Identify root causes and contributing factors.
4. Track action items and fixes.
5. Share learnings across team.
6. Monitor for recurring issues.

```markdown
# Incident Postmortem Template

## Incident Summary
- Title: [Brief description]
- Date/Time: [When incident occurred]
- Duration: [How long until resolved]
- Severity: [Critical/High/Medium/Low]

## Timeline
- 14:30 - Alert triggered (high query latency)
- 14:35 - On-call engineer acknowledged
- 14:40 - Root cause identified (failed node)
- 14:50 - Node restarted, traffic restored
- 15:00 - All metrics normal, incident resolved

## Root Cause Analysis
Primary cause: Node hardware failure
Contributing factors: 
- Single point of failure in network path
- Slow failover detection (5 minute threshold)

## Impact Assessment
- Duration: 20 minutes
- Affected services: Order processing
- Estimated business impact: $50K revenue loss
- Customers affected: ~1000 active users

## Lessons Learned
1. Network redundancy insufficient
2. Failover detection too slow
3. Missing alerting on node health

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add secondary network path | Infra | 2024-02-01 | P0 |
| Reduce failover threshold to 30s | SRE | 2024-01-25 | P1 |
| Implement node health metrics | Monitoring | 2024-01-20 | P1 |

## Prevention
Changes made to prevent recurrence:
1. Added network redundancy
2. Implemented quicker failover
3. Enhanced monitoring

## Follow-up
- 1-week check: Verify fixes effective
- 30-day check: Confirm no regression
```

```python
# Incident tracking system
class IncidentManager:
    def __init__(self):
        self.incidents = []
    
    def create_incident(self, title, severity, description):
        """Create incident record"""
        incident = {
            'id': len(self.incidents) + 1,
            'title': title,
            'severity': severity,
            'description': description,
            'created_at': datetime.now(),
            'status': 'open',
            'root_cause': None,
            'resolution': None,
            'postmortem': None,
            'action_items': []
        }
        self.incidents.append(incident)
        return incident
    
    def record_root_cause(self, incident_id, root_cause, contributing_factors):
        """Record root cause analysis"""
        incident = self.incidents[incident_id - 1]
        incident['root_cause'] = root_cause
        incident['contributing_factors'] = contributing_factors
    
    def add_action_item(self, incident_id, action, owner, due_date, priority):
        """Add action item to prevent recurrence"""
        incident = self.incidents[incident_id - 1]
        incident['action_items'].append({
            'action': action,
            'owner': owner,
            'due_date': due_date,
            'priority': priority,
            'status': 'open'
        })
    
    def generate_postmortem(self, incident_id):
        """Generate postmortem report"""
        incident = self.incidents[incident_id - 1]
        
        report = f"""
        INCIDENT POSTMORTEM
        ===================
        
        Title: {incident['title']}
        Severity: {incident['severity']}
        Duration: {(incident.get('resolved_at', datetime.now()) - incident['created_at']).total_seconds()} seconds
        
        Root Cause: {incident.get('root_cause', 'TBD')}
        Contributing Factors: {incident.get('contributing_factors', 'TBD')}
        
        Action Items:
        """
        
        for item in incident['action_items']:
            report += f"\n  - {item['action']} (Owner: {item['owner']}, Due: {item['due_date']})"
        
        return report
    
    def check_recurring_issues(self):
        """Check for recurring issues (same root cause)"""
        root_causes = {}
        
        for incident in self.incidents:
            rc = incident.get('root_cause')
            if rc:
                root_causes[rc] = root_causes.get(rc, 0) + 1
        
        recurring = {k: v for k, v in root_causes.items() if v > 1}
        
        if recurring:
            logging.warning(f"Recurring issues detected: {recurring}")
        
        return recurring

# Usage
manager = IncidentManager()

# Create incident
incident = manager.create_incident(
    "Database query latency spike",
    "High",
    "Query P99 latency exceeded 1000ms"
)

# Record analysis
manager.record_root_cause(
    incident['id'],
    "Inefficient query plan due to missing index",
    ["High data volume", "Query optimizer choosing sequential scan"]
)

# Add action items
manager.add_action_item(
    incident['id'],
    "Add index on frequently used column",
    "dba@company.com",
    "2024-02-01",
    "P0"
)

# Generate postmortem
print(manager.generate_postmortem(incident['id']))
```

### Q205: How do I implement on-call rotation and alerting schedules?

1. Define on-call rotation (weekly or bi-weekly).
2. Implement alerting rules by severity.
3. Escalate based on time/response.
4. Track on-call metrics and fatigue.
5. Implement runbooks for common issues.
6. Provide on-call support and training.

```python
from datetime import datetime, timedelta
from enum import Enum

class AlertSeverity(Enum):
    CRITICAL = 1
    HIGH = 2
    MEDIUM = 3
    LOW = 4

class OnCallSchedule:
    def __init__(self):
        self.rotation = []
        self.alerts = []
    
    def create_rotation(self, engineers, rotation_days=7):
        """Create on-call rotation"""
        start_date = datetime.now()
        
        for i, engineer in enumerate(engineers):
            for day in range(rotation_days):
                self.rotation.append({
                    'engineer': engineer,
                    'start_date': start_date + timedelta(days=i*rotation_days + day),
                    'end_date': start_date + timedelta(days=i*rotation_days + day + 1),
                    'primary': i == 0,
                })
    
    def get_on_call_engineer(self, timestamp=None):
        """Get on-call engineer for timestamp"""
        if timestamp is None:
            timestamp = datetime.now()
        
        for rotation in self.rotation:
            if rotation['start_date'] <= timestamp <= rotation['end_date']:
                return rotation['engineer']
        
        return None
    
    def get_escalation_path(self, severity):
        """Get escalation path based on severity"""
        if severity == AlertSeverity.CRITICAL:
            return {
                'immediate': self.get_on_call_engineer(),
                'escalate_after_5min': 'engineering_manager',
                'escalate_after_15min': 'director'
            }
        elif severity == AlertSeverity.HIGH:
            return {
                'immediate': self.get_on_call_engineer(),
                'escalate_after_15min': 'engineering_manager'
            }
        else:
            return {
                'immediate': self.get_on_call_engineer(),
                'escalate_after_30min': 'engineering_manager'
            }
    
    def send_alert(self, alert_title, description, severity):
        """Send alert following escalation path"""
        escalation = self.get_escalation_path(severity)
        
        # Send to primary
        primary_engineer = escalation['immediate']
        self.notify_engineer(primary_engineer, alert_title, severity)
        
        # Schedule escalation if not acknowledged
        threading.Timer(
            300.0,  # 5 minutes
            self.escalate_alert,
            [alert_title, escalation]
        ).start()
    
    def escalate_alert(self, alert_title, escalation):
        """Escalate alert if not acknowledged"""
        # Check if primary acknowledged
        if not self.is_acknowledged(alert_title):
            # Escalate to next level
            escalation_contact = escalation.get('escalate_after_5min')
            if escalation_contact:
                self.notify_engineer(escalation_contact, alert_title + " (escalated)", AlertSeverity.CRITICAL)
    
    def notify_engineer(self, engineer, alert_title, severity):
        """Send notification to engineer"""
        # Implementation: SMS, email, Slack, PagerDuty
        logging.info(f"Alert to {engineer}: {alert_title} (Severity: {severity.name})")
    
    def is_acknowledged(self, alert_title):
        """Check if alert acknowledged"""
        # Implementation: query PagerDuty or internal system
        return False
    
    def track_on_call_metrics(self):
        """Track on-call metrics for fatigue analysis"""
        metrics = {}
        
        for engineer in self.rotation:
            eng_name = engineer['engineer']
            metrics[eng_name] = {
                'alerts_received': len([a for a in self.alerts if a['on_call'] == eng_name]),
                'alerts_resolved': len([a for a in self.alerts if a['on_call'] == eng_name and a['resolved']]),
                'avg_time_to_resolution': self.calculate_avg_resolution_time(eng_name),
                'alerts_escalated': len([a for a in self.alerts if a['on_call'] == eng_name and a['escalated']])
            }
        
        # Alert if engineer has too many alerts (fatigue risk)
        for eng_name, stats in metrics.items():
            if stats['alerts_received'] > 20:
                logging.warning(f"High alert load for {eng_name}: {stats['alerts_received']} alerts")
        
        return metrics

# Usage
schedule = OnCallSchedule()

engineers = ['alice@company.com', 'bob@company.com', 'charlie@company.com']
schedule.create_rotation(engineers, rotation_days=7)

# Send alert
schedule.send_alert(
    "Database query latency critical",
    "P99 latency > 1000ms",
    AlertSeverity.CRITICAL
)

# Track metrics
metrics = schedule.track_on_call_metrics()
```

---

## SECTION 73: ADVANCED KNOWLEDGE BASE

### Q206: How do I implement searchable knowledge base for operational procedures?

1. Create structured documentation.
2. Implement full-text search.
3. Tag procedures by category and component.
4. Link related procedures.
5. Version control documentation.
6. Monitor documentation accuracy.

```sql
-- Knowledge base schema
CREATE TABLE kb_articles (
  article_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR NOT NULL,
  content TEXT NOT NULL,
  category VARCHAR,
  tags VARCHAR[],
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now(),
  author VARCHAR,
  version INT DEFAULT 1,
  is_published BOOLEAN DEFAULT false
);

-- Full-text index for search
CREATE INDEX idx_kb_fulltext ON kb_articles USING GIN (to_tsvector('english', content || ' ' || title));

-- Tag index for filtering
CREATE INDEX idx_kb_tags ON kb_articles USING GIN (tags);

-- Procedures for maintenance
INSERT INTO kb_articles (title, content, category, tags, author) VALUES
  ('Node Failure Recovery', 
   'Steps to recover from node failure: 1. Verify node down...', 
   'Operations',
   ARRAY['incident', 'recovery', 'node'],
   'dba@company.com'),
  ('Backup Restoration',
   'Steps to restore from backup: 1. Identify backup to restore...',
   'Backup',
   ARRAY['backup', 'recovery', 'restore'],
   'dba@company.com');

-- Full-text search
SELECT 
  article_id,
  title,
  category,
  ts_rank_cd(to_tsvector('english', content), query) as relevance
FROM kb_articles,
     to_tsquery('english', 'node & failure') as query
WHERE to_tsvector('english', content) @@ query
ORDER BY relevance DESC;

-- Tag-based search
SELECT * FROM kb_articles
WHERE 'recovery' = ANY(tags)
ORDER BY updated_at DESC;

-- Related articles
SELECT 
  a1.article_id as source_article,
  a2.article_id as related_article,
  COUNT(CASE WHEN a1.tags && a2.tags THEN 1 END) as common_tags
FROM kb_articles a1
CROSS JOIN kb_articles a2
WHERE a1.article_id != a2.article_id
GROUP BY a1.article_id, a2.article_id
HAVING COUNT(CASE WHEN a1.tags && a2.tags THEN 1 END) > 0
ORDER BY common_tags DESC;
```

---

## SECTION 74: ADVANCED AUTOMATION

### Q207: How do I implement infrastructure-as-code for CockroachDB deployments?

1. Define cluster configuration in code.
2. Version control all infrastructure.
3. Implement automated deployment pipelines.
4. Test infrastructure changes in staging.
5. Enable rollback procedures.
6. Document infrastructure decisions.

```python
# Infrastructure-as-Code using Python and Terraform

# cockroach_cluster.tf
terraform_code = """
variable "cluster_name" {
  type = string
}

variable "node_count" {
  type = number
  default = 3
}

variable "machine_type" {
  type = string
  default = "n1-standard-4"
}

resource "google_compute_instance" "cockroach_nodes" {
  count = var.node_count
  
  name = "${var.cluster_name}-node-${count.index + 1}"
  machine_type = var.machine_type
  zone = "us-central1-a"
  
  boot_disk {
    initialize_params {
      image = "ubuntu-2024-lts"
      size = 100
      type = "pd-ssd"
    }
  }
  
  network_interface {
    network = "default"
    access_config {}
  }
  
  metadata_startup_script = file("startup-script.sh")
  
  tags = ["cockroachdb", "prod"]
}

resource "google_sql_database_instance" "backup_storage" {
  name = "${var.cluster_name}-backups"
  database_version = "POSTGRES_15"
  
  settings {
    tier = "db-f1-micro"
    backup_configuration {
      enabled = true
      start_time = "02:00"
    }
  }
}
"""

# Configuration as Python
class CockroachDBCluster:
    def __init__(self, name, region, node_count=3):
        self.name = name
        self.region = region
        self.node_count = node_count
        self.config = {}
    
    def define_cluster(self):
        """Define cluster configuration"""
        self.config = {
            'cluster_name': self.name,
            'region': self.region,
            'nodes': [
                {
                    'name': f'{self.name}-node-{i}',
                    'instance_type': 'n1-standard-4',
                    'storage': '100GB',
                    'zone': f'{self.region}-a'
                }
                for i in range(1, self.node_count + 1)
            ],
            'networking': {
                'vpc': f'{self.name}-vpc',
                'firewall_rules': [
                    {'protocol': 'tcp', 'port': 26257, 'source': 'app-tier'},
                    {'protocol': 'tcp', 'port': 8080, 'source': 'admin-network'}
                ]
            },
            'backup': {
                'destination': f'gs://backups-{self.name}',
                'frequency': 'daily',
                'retention_days': 30
            },
            'monitoring': {
                'prometheus': True,
                'datadog_agent': True,
                'log_forwarding': 'stackdriver'
            }
        }
        return self.config
    
    def deploy_with_terraform(self):
        """Generate and apply Terraform"""
        # Generate terraform configuration
        tf_vars = f"""
        cluster_name = "{self.name}"
        node_count = {self.node_count}
        region = "{self.region}"
        """
        
        # Write to file
        with open('terraform.tfvars', 'w') as f:
            f.write(tf_vars)
        
        # Apply terraform
        subprocess.run(['terraform', 'apply', '-auto-approve'])
    
    def validate_deployment(self):
        """Validate deployed cluster"""
        # Verify all nodes running
        # Verify replication
        # Run connectivity tests
        pass

# Usage
cluster = CockroachDBCluster('production-cluster', 'us-central1', node_count=3)
cluster.define_cluster()
cluster.deploy_with_terraform()
cluster.validate_deployment()
```

---

## SECTION 75: FINAL BEST PRACTICES SUMMARY

### Q208: What is comprehensive production readiness assessment?

**INFRASTRUCTURE READINESS:**
- [ ] Minimum 3 nodes across availability zones
- [ ] Network redundancy and low latency
- [ ] Dedicated storage (SSD/NVMe) per node
- [ ] Automated failover mechanisms
- [ ] Disaster recovery procedures documented

**DATA INTEGRITY:**
- [ ] Backup automation configured
- [ ] Point-in-time recovery tested
- [ ] Cross-region backup copies
- [ ] Data consistency validation procedures
- [ ] PITR restoration window validated

**SECURITY & COMPLIANCE:**
- [ ] TLS enabled end-to-end
- [ ] User authentication and RBAC implemented
- [ ] Audit logging enabled
- [ ] Data encryption at rest configured
- [ ] Compliance requirements documented

**MONITORING & OBSERVABILITY:**
- [ ] Metrics collection operational
- [ ] Dashboards created for all audiences
- [ ] Alerting rules configured and tested
- [ ] Log aggregation active
- [ ] On-call procedures documented

**OPERATIONS:**
- [ ] Runbooks written for common procedures
- [ ] Incident response plan established
- [ ] Postmortem process defined
- [ ] Training completed for operations team
- [ ] Change management process implemented

**PERFORMANCE:**
- [ ] Load testing completed
- [ ] Queries optimized
- [ ] Indexes validated
- [ ] Performance baselines established
- [ ] Scaling procedures tested

**RELIABILITY:**
- [ ] Failover procedures tested
- [ ] Backup restoration tested
- [ ] Cluster recovery procedures tested
- [ ] Network failure scenarios tested
- [ ] Application reconnection tested

### Q209: How do I implement and maintain production SLAs?

```python
class SLAManager:
    def __init__(self):
        self.slas = {
            'availability': 99.9,  # % uptime
            'query_latency_p99': 100,  # milliseconds
            'backup_completion': 100,  # % success
            'failover_time': 5  # minutes
        }
        self.current_metrics = {}
    
    def calculate_error_budget(self, sla_target, time_window_days=30):
        """Calculate remaining error budget"""
        total_seconds = time_window_days * 24 * 3600
        allowed_downtime = (1 - sla_target/100) * total_seconds
        
        return {
            'total_seconds_in_window': total_seconds,
            'allowed_downtime_seconds': allowed_downtime,
            'allowed_downtime_minutes': allowed_downtime / 60,
            'allowed_downtime_hours': allowed_downtime / 3600
        }
    
    def monitor_sla_compliance(self):
        """Monitor and report SLA compliance"""
        compliance = {}
        
        # Check availability
        uptime_percent = self.current_metrics.get('uptime', 100)
        compliance['availability'] = {
            'target': self.slas['availability'],
            'actual': uptime_percent,
            'compliant': uptime_percent >= self.slas['availability']
        }
        
        # Check query latency
        p99_latency = self.current_metrics.get('query_p99', 0)
        compliance['query_latency'] = {
            'target': self.slas['query_latency_p99'],
            'actual': p99_latency,
            'compliant': p99_latency <= self.slas['query_latency_p99']
        }
        
        return compliance

# Usage
sla_manager = SLAManager()

# Calculate error budget for 99.9% SLA over 30 days
error_budget = sla_manager.calculate_error_budget(99.9, 30)
print(f"Error budget for month: {error_budget['allowed_downtime_minutes']:.1f} minutes")

# Monitor compliance
compliance = sla_manager.monitor_sla_compliance()
```

### Q210: Complete operational excellence checklist and next steps

**IMMEDIATE (First Week):**
- [ ] Deploy cluster to production
- [ ] Enable monitoring and alerting
- [ ] Run sanity checks and data validation
- [ ] Establish on-call rotation
- [ ] Document as-built architecture

**SHORT-TERM (First Month):**
- [ ] Optimize based on real workload patterns
- [ ] Refine alerting thresholds
- [ ] Conduct failure scenario tests
- [ ] Train operations team
- [ ] Document lessons learned

**MEDIUM-TERM (First Quarter):**
- [ ] Implement capacity planning
- [ ] Automate routine operations
- [ ] Refine disaster recovery procedures
- [ ] Conduct security assessment
- [ ] Plan for growth/scaling

**LONG-TERM (Ongoing):**
- [ ] Monitor industry changes
- [ ] Plan technology upgrades
- [ ] Optimize costs continuously
- [ ] Maintain knowledge base
- [ ] Foster operational excellence culture

---

# SECTION 76: ADVANCED DATA MIGRATION STRATEGIES

### Q211: How do I implement zero-downtime data migration from PostgreSQL to CockroachDB?

1. Set up bidirectional replication during migration.
2. Verify data consistency continuously.
3. Test application with CockroachDB in parallel.
4. Gradually shift traffic to CockroachDB.
5. Maintain rollback capability.
6. Monitor both systems during cutover.

```python
import psycopg2
from concurrent.futures import ThreadPoolExecutor
import hashlib
import time

class ZeroDowntimeMigration:
    def __init__(self, pg_conn_str, crdb_conn_str):
        self.pg_conn = psycopg2.connect(pg_conn_str)
        self.crdb_conn = psycopg2.connect(crdb_conn_str)
        self.migration_state = {}
    
    def phase_1_schema_replication(self):
        """Export schema from PostgreSQL and import to CockroachDB"""
        pg_cursor = self.pg_conn.cursor()
        crdb_cursor = self.crdb_conn.cursor()
        
        # Export schema
        pg_cursor.execute("""
            SELECT table_name, column_name, data_type
            FROM information_schema.columns
            WHERE table_schema = 'public'
            ORDER BY table_name, ordinal_position
        """)
        
        schema_info = pg_cursor.fetchall()
        
        # Create tables in CockroachDB
        current_table = None
        create_statement = ""
        
        for table_name, column_name, data_type in schema_info:
            if table_name != current_table:
                if current_table:
                    crdb_cursor.execute(create_statement + ");")
                    self.crdb_conn.commit()
                
                current_table = table_name
                create_statement = f"CREATE TABLE IF NOT EXISTS {table_name} ({column_name} {data_type}"
            else:
                create_statement += f", {column_name} {data_type}"
        
        if current_table:
            crdb_cursor.execute(create_statement + ");")
            self.crdb_conn.commit()
        
        print(f"Phase 1 Complete: Schema replicated ({len(schema_info)} columns)")
        self.migration_state['phase_1_complete'] = True
    
    def phase_2_initial_data_copy(self, batch_size=10000):
        """Copy all existing data from PostgreSQL to CockroachDB"""
        pg_cursor = self.pg_conn.cursor()
        crdb_cursor = self.crdb_conn.cursor()
        
        # Get list of tables
        pg_cursor.execute("""
            SELECT table_name FROM information_schema.tables
            WHERE table_schema = 'public' AND table_type = 'BASE TABLE'
        """)
        
        tables = [row[0] for row in pg_cursor.fetchall()]
        
        for table_name in tables:
            print(f"Copying table: {table_name}")
            
            pg_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            total_rows = pg_cursor.fetchone()[0]
            
            # Copy in batches
            for offset in range(0, total_rows, batch_size):
                pg_cursor.execute(f"SELECT * FROM {table_name} LIMIT {batch_size} OFFSET {offset}")
                rows = pg_cursor.fetchall()
                
                if rows:
                    # Get column names
                    column_names = [desc[0] for desc in pg_cursor.description]
                    
                    # Insert into CockroachDB
                    placeholders = ','.join(['%s'] * len(column_names))
                    insert_sql = f"INSERT INTO {table_name} ({','.join(column_names)}) VALUES ({placeholders})"
                    
                    for row in rows:
                        crdb_cursor.execute(insert_sql, row)
                    
                    self.crdb_conn.commit()
                    print(f"  Copied {offset + len(rows)}/{total_rows} rows")
        
        self.migration_state['phase_2_complete'] = True
        print("Phase 2 Complete: Initial data copy finished")
    
    def phase_3_bidirectional_replication(self):
        """Set up CDC for continuous replication"""
        # PostgreSQL -> CockroachDB using Debezium or similar
        # CockroachDB -> PostgreSQL for failback capability
        
        # This would use Change Data Capture (CDC) tools
        print("Phase 3: Bidirectional replication started")
        self.migration_state['phase_3_active'] = True
    
    def phase_4_data_consistency_verification(self):
        """Verify data consistency between systems"""
        pg_cursor = self.pg_conn.cursor()
        crdb_cursor = self.crdb_conn.cursor()
        
        pg_cursor.execute("""
            SELECT table_name FROM information_schema.tables
            WHERE table_schema = 'public' AND table_type = 'BASE TABLE'
        """)
        
        tables = [row[0] for row in pg_cursor.fetchall()]
        inconsistencies = []
        
        for table_name in tables:
            # Compare row counts
            pg_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            pg_count = pg_cursor.fetchone()[0]
            
            crdb_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            crdb_count = crdb_cursor.fetchone()[0]
            
            if pg_count != crdb_count:
                inconsistencies.append({
                    'table': table_name,
                    'pg_count': pg_count,
                    'crdb_count': crdb_count
                })
            
            # Sample checksum verification
            pg_cursor.execute(f"""
                SELECT MD5(STRING_AGG(MD5(ROW(*)::TEXT), '' ORDER BY *)) 
                FROM {table_name} LIMIT 1000
            """)
            pg_hash = pg_cursor.fetchone()[0]
            
            crdb_cursor.execute(f"""
                SELECT MD5(STRING_AGG(MD5(ROW(*)::TEXT), '' ORDER BY *)) 
                FROM {table_name} LIMIT 1000
            """)
            crdb_hash = crdb_cursor.fetchone()[0]
            
            if pg_hash != crdb_hash:
                inconsistencies.append({
                    'table': table_name,
                    'type': 'checksum_mismatch',
                    'pg_hash': pg_hash,
                    'crdb_hash': crdb_hash
                })
        
        if inconsistencies:
            print(f"Inconsistencies found: {inconsistencies}")
            return False
        
        print("Phase 4 Complete: Data consistency verified")
        self.migration_state['phase_4_complete'] = True
        return True
    
    def phase_5_application_testing(self):
        """Test application against both systems in parallel"""
        print("Phase 5: Running application tests")
        
        # Configure application to:
        # 1. Route reads to CockroachDB
        # 2. Write to both PostgreSQL and CockroachDB
        # 3. Compare results for consistency
        
        self.migration_state['phase_5_active'] = True
    
    def phase_6_traffic_cutover(self, percentage=10):
        """Gradually shift traffic to CockroachDB"""
        print(f"Phase 6: Cutting over {percentage}% of traffic to CockroachDB")
        
        # Application configuration:
        # - Route percentage% of traffic to CockroachDB
        # - Monitor error rates and latency
        # - Maintain fallback to PostgreSQL
        
        self.migration_state['traffic_cutover_percent'] = percentage
    
    def phase_7_rollback(self):
        """Rollback to PostgreSQL if issues detected"""
        print("Phase 7: Initiating rollback to PostgreSQL")
        
        # Stop traffic to CockroachDB
        # Resume all traffic to PostgreSQL
        # Investigate issues
        
        self.migration_state['rollback_initiated'] = True
    
    def execute_migration(self):
        """Execute complete migration"""
        try:
            self.phase_1_schema_replication()
            self.phase_2_initial_data_copy()
            self.phase_3_bidirectional_replication()
            
            if not self.phase_4_data_consistency_verification():
                raise Exception("Data consistency check failed")
            
            self.phase_5_application_testing()
            
            # Gradually increase traffic cutover
            for percentage in [10, 25, 50, 75, 100]:
                self.phase_6_traffic_cutover(percentage)
                time.sleep(300)  # Monitor for 5 minutes
            
            print("Migration complete! All traffic now on CockroachDB")
        
        except Exception as e:
            print(f"Migration failed: {e}")
            self.phase_7_rollback()

# Usage
migration = ZeroDowntimeMigration(
    'postgresql://user:pass@pg-host:5432/mydb',
    'postgresql://root:pass@crdb-host:26257/mydb'
)
migration.execute_migration()
```

### Q212: How do I handle large table partitioning during migration?

1. Identify partition strategy based on query patterns.
2. Create partitions in CockroachDB before data migration.
3. Migrate data partition by partition.
4. Verify partition alignment.
5. Update application queries if needed.
6. Monitor partition performance.

```sql
-- Source: PostgreSQL partitioned table
CREATE TABLE orders (
  order_id BIGSERIAL,
  customer_id INT,
  order_date DATE,
  amount DECIMAL,
  PRIMARY KEY (order_id)
) PARTITION BY RANGE (DATE_TRUNC('month', order_date));

-- Create partitions for each month
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Target: CockroachDB partitioned table
CREATE TABLE orders_crdb (
  order_id BIGINT,
  customer_id INT,
  order_date DATE,
  amount DECIMAL,
  PRIMARY KEY (order_id)
) PARTITION BY RANGE (order_date) (
  PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
  PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01')
);

-- Migration query (partition by partition)
INSERT INTO orders_crdb
SELECT * FROM orders_2024_01;

-- Verify migration
SELECT 
  'orders_2024_01' as partition_name,
  COUNT(*) as row_count,
  MIN(order_date) as min_date,
  MAX(order_date) as max_date
FROM orders_2024_01
UNION ALL
SELECT 
  'orders_crdb (p_2024_01)',
  COUNT(*),
  MIN(order_date),
  MAX(order_date)
FROM orders_crdb
WHERE order_date >= '2024-01-01' AND order_date < '2024-02-01';

-- Monitor partition growth
SELECT 
  partition_name,
  row_count,
  pg_size_pretty(partition_size) as size
FROM (
  SELECT 
    'p_2024_01' as partition_name,
    COUNT(*) as row_count,
    pg_total_relation_size('orders_crdb_p_2024_01') as partition_size
  FROM orders_crdb
  WHERE order_date >= '2024-01-01' AND order_date < '2024-02-01'
) t;
```

---

## SECTION 77: ADVANCED MULTI-TENANCY PATTERNS

### Q213: How do I implement tenant isolation with performance guarantees?

1. Separate databases per tenant for strict isolation.
2. Implement resource quotas per tenant.
3. Monitor per-tenant metrics.
4. Implement tenant-level SLAs.
5. Provide tenant-specific dashboards.
6. Implement automated scaling per tenant.

```sql
-- Multi-tenant architecture with isolation
CREATE TABLE tenants (
  tenant_id UUID PRIMARY KEY,
  tenant_name VARCHAR,
  tier VARCHAR,  -- basic, standard, premium
  max_qps INT,
  max_storage_gb BIGINT,
  created_at TIMESTAMP DEFAULT now()
);

-- Tenant-specific configuration
CREATE TABLE tenant_config (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(tenant_id),
  max_connections INT,
  query_timeout_ms INT,
  backup_frequency VARCHAR,
  data_residency_region VARCHAR
);

-- Tenant resource usage tracking
CREATE TABLE tenant_resource_usage (
  tenant_id UUID,
  timestamp TIMESTAMP,
  queries_executed INT,
  bytes_processed BIGINT,
  storage_used_gb BIGINT,
  active_connections INT,
  PRIMARY KEY (tenant_id, timestamp)
);

-- Monitor tenant resource limits
CREATE VIEW tenant_limit_status AS
SELECT 
  t.tenant_id,
  t.tenant_name,
  t.max_qps,
  COUNT(DISTINCT tru.timestamp) as queries_in_window,
  CASE 
    WHEN COUNT(DISTINCT tru.timestamp) > t.max_qps THEN 'EXCEEDED'
    WHEN COUNT(DISTINCT tru.timestamp) > t.max_qps * 0.8 THEN 'WARNING'
    ELSE 'OK'
  END as qps_status,
  t.max_storage_gb,
  (SELECT storage_used_gb FROM tenant_resource_usage WHERE tenant_id = t.tenant_id ORDER BY timestamp DESC LIMIT 1) as current_storage_gb,
  CASE 
    WHEN (SELECT storage_used_gb FROM tenant_resource_usage WHERE tenant_id = t.tenant_id ORDER BY timestamp DESC LIMIT 1) > t.max_storage_gb THEN 'EXCEEDED'
    ELSE 'OK'
  END as storage_status
FROM tenants t
LEFT JOIN tenant_resource_usage tru ON t.tenant_id = tru.tenant_id
  AND tru.timestamp > now() - interval '1 minute'
GROUP BY t.tenant_id, t.tenant_name, t.max_qps, t.max_storage_gb;

-- Tenant-level SLA enforcement
CREATE TABLE tenant_slas (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(tenant_id),
  availability_target DECIMAL,  -- 99.9
  query_p99_latency_ms INT,
  failover_rto_minutes INT,
  backup_rpo_hours INT
);

-- Application-level tenant isolation
-- Route requests to tenant-specific database or schema
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_widgets;

-- Tables scoped to tenant
CREATE TABLE tenant_acme.users (
  user_id INT PRIMARY KEY,
  email VARCHAR,
  created_at TIMESTAMP
);

CREATE TABLE tenant_widgets.users (
  user_id INT PRIMARY KEY,
  email VARCHAR,
  created_at TIMESTAMP
);

-- Implement row-level security for multi-tenant schema
CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- Enforce at application layer
SET app.tenant_id = 'acme-tenant-id';
SELECT * FROM users;  -- Only returns acme tenant's users
```

### Q214: How do I implement tenant-aware backup and recovery?

1. Create separate backup schedules per tenant.
2. Store backups in tenant-isolated locations.
3. Implement tenant-specific retention policies.
4. Test restoration per tenant independently.
5. Provide self-service restore capability.
6. Monitor backup compliance per tenant.

```sql
-- Tenant-specific backup configuration
CREATE TABLE tenant_backup_config (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(tenant_id),
  backup_schedule VARCHAR,  -- daily, weekly, etc.
  retention_days INT,
  backup_destination VARCHAR,  -- s3://bucket/tenant_id/
  incremental_enabled BOOLEAN,
  encryption_key_id VARCHAR
);

-- Create tenant-specific backup schedule
CREATE SCHEDULE tenant_backup_acme FOR 
  BACKUP INTO 's3://backups/tenant_acme/'
  RECURRING EVERY 1 day
  WITH RETENTION '30d';

-- Track backup completion per tenant
CREATE TABLE tenant_backup_log (
  backup_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES tenants(tenant_id),
  backup_start TIMESTAMP,
  backup_end TIMESTAMP,
  backup_size_gb BIGINT,
  status VARCHAR,  -- success, failed, partial
  destination VARCHAR,
  retention_until DATE
);

-- Tenant-specific restore procedure
CREATE PROCEDURE restore_tenant_backup(
  p_tenant_id UUID,
  p_backup_id UUID
) AS $$
BEGIN
  -- Verify tenant has permission
  SELECT 1 FROM tenants WHERE tenant_id = p_tenant_id
  OR RAISE EXCEPTION 'Tenant not found';
  
  -- Get backup details
  SELECT destination INTO v_backup_location
  FROM tenant_backup_log
  WHERE backup_id = p_backup_id AND tenant_id = p_tenant_id
  OR RAISE EXCEPTION 'Backup not found';
  
  -- Execute restore
  EXECUTE 'RESTORE FROM ''' || v_backup_location || '''';
  
  -- Log restore
  INSERT INTO tenant_restore_log (tenant_id, backup_id, restore_time)
  VALUES (p_tenant_id, p_backup_id, now());
END $$ LANGUAGE plpgsql;

-- Monitor backup compliance per tenant
SELECT 
  t.tenant_id,
  t.tenant_name,
  tbc.retention_days,
  COUNT(*) as successful_backups,
  MAX(backup_end) as latest_backup,
  DATEDIFF('day', MAX(backup_end), now()) as days_since_backup,
  CASE 
    WHEN DATEDIFF('day', MAX(backup_end), now()) > 2 THEN 'OVERDUE'
    WHEN DATEDIFF('day', MAX(backup_end), now()) > 1 THEN 'WARNING'
    ELSE 'OK'
  END as backup_status
FROM tenants t
LEFT JOIN tenant_backup_config tbc ON t.tenant_id = tbc.tenant_id
LEFT JOIN tenant_backup_log tbl ON t.tenant_id = tbl.tenant_id AND tbl.status = 'success'
GROUP BY t.tenant_id, t.tenant_name, tbc.retention_days;

-- Verify backup integrity per tenant
CREATE PROCEDURE verify_tenant_backup_integrity(
  p_tenant_id UUID
) AS $$
BEGIN
  -- Restore to isolated environment
  -- Run data validation queries
  -- Compare checksums with production
  -- Report results
END $$ LANGUAGE plpgsql;
```

---

## SECTION 78: ADVANCED QUERY TUNING

### Q215: How do I implement query plan regression detection?

1. Baseline query plans in production.
2. Monitor for plan changes.
3. Alert when plan becomes more expensive.
4. Implement automatic plan freezing.
5. Provide manual plan override capability.
6. Track plan changes over time.

```sql
-- Store baseline query plans
CREATE TABLE query_plan_baseline (
  query_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_hash VARCHAR UNIQUE,
  query_text TEXT,
  baseline_plan TEXT,  -- EXPLAIN output
  baseline_cost DECIMAL,
  baseline_rows INT,
  created_at TIMESTAMP DEFAULT now()
);

-- Track plan changes
CREATE TABLE query_plan_history (
  plan_history_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_id UUID REFERENCES query_plan_baseline(query_id),
  current_plan TEXT,
  current_cost DECIMAL,
  current_rows INT,
  plan_changed BOOLEAN,
  cost_increase_percent DECIMAL,
  captured_at TIMESTAMP DEFAULT now()
);

-- Detect plan regression
CREATE FUNCTION detect_plan_regression(p_query_hash VARCHAR) RETURNS TABLE (
  regression_detected BOOLEAN,
  cost_increase_percent DECIMAL,
  baseline_cost DECIMAL,
  current_cost DECIMAL
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    (qph.current_cost > qpb.baseline_cost * 1.2)::BOOLEAN,
    ((qph.current_cost - qpb.baseline_cost) / qpb.baseline_cost) * 100,
    qpb.baseline_cost,
    qph.current_cost
  FROM query_plan_baseline qpb
  LEFT JOIN query_plan_history qph ON qpb.query_id = qph.query_id
  WHERE qpb.query_hash = p_query_hash
  ORDER BY qph.captured_at DESC
  LIMIT 1;
END $$ LANGUAGE plpgsql;

-- Alert on regression
CREATE FUNCTION alert_plan_regression() RETURNS void AS $$
DECLARE
  v_regression RECORD;
BEGIN
  FOR v_regression IN
    SELECT 
      qpb.query_id,
      qpb.query_hash,
      qpb.query_text,
      qph.current_cost,
      qpb.baseline_cost
    FROM query_plan_baseline qpb
    JOIN query_plan_history qph ON qpb.query_id = qph.query_id
    WHERE qph.current_cost > qpb.baseline_cost * 1.2
      AND qph.captured_at > now() - interval '1 hour'
  LOOP
    INSERT INTO alerts (severity, message, query_id)
    VALUES ('HIGH', 'Query plan regression detected: ' || v_regression.query_text, v_regression.query_id);
  END LOOP;
END $$ LANGUAGE plpgsql;

-- Manual plan override (force specific plan)
CREATE TABLE query_plan_override (
  override_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_id UUID REFERENCES query_plan_baseline(query_id),
  forced_plan TEXT,  -- EXPLAIN output or hints
  reason VARCHAR,
  created_by VARCHAR,
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);

-- Apply plan override in application
-- Use hints or explicit joins to force plan
SELECT /*+ INDEX(orders idx_customer_date) */ 
  o.* 
FROM orders o
WHERE customer_id = ? AND created_at > ?;
```

### Q216: How do I implement query hint system for performance control?

1. Store query hints in database.
2. Apply hints automatically for known patterns.
3. Allow developers to override hints.
4. Monitor hint effectiveness.
5. Version control hints with queries.
6. Audit hint changes.

```sql
-- Query hint configuration
CREATE TABLE query_hints (
  hint_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  query_pattern VARCHAR,  -- Pattern matching query
  hint_text VARCHAR,      -- /*+ INDEX(...) */ format
  priority INT,           -- Higher = applied first
  enabled BOOLEAN DEFAULT true,
  reason VARCHAR,
  created_by VARCHAR,
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);

-- Examples of hints
INSERT INTO query_hints (query_pattern, hint_text, reason) VALUES
  ('SELECT .* FROM orders WHERE customer_id', '/*+ INDEX(orders idx_customer_date) */', 'Force customer index'),
  ('SELECT .* FROM orders o JOIN customers c', '/*+ HASH_JOIN(o, c) */', 'Use hash join for large join');

-- Application function to apply hints
CREATE FUNCTION apply_query_hints(p_query_text TEXT) RETURNS TEXT AS $$
DECLARE
  v_hint_text VARCHAR;
  v_modified_query TEXT := p_query_text;
BEGIN
  -- Find matching hints
  SELECT hint_text INTO v_hint_text
  FROM query_hints
  WHERE enabled = true
    AND (expires_at IS NULL OR expires_at > now())
    AND p_query_text SIMILAR TO query_pattern
  ORDER BY priority DESC
  LIMIT 1;
  
  -- Apply hint if found
  IF v_hint_text IS NOT NULL THEN
    v_modified_query := REPLACE(p_query_text, 'SELECT', 'SELECT ' || v_hint_text);
  END IF;
  
  RETURN v_modified_query;
END $$ LANGUAGE plpgsql;

-- Track hint effectiveness
CREATE TABLE hint_effectiveness (
  hint_id UUID REFERENCES query_hints(hint_id),
  execution_with_hint_ms DECIMAL,
  execution_without_hint_ms DECIMAL,
  improvement_percent DECIMAL,
  measured_at TIMESTAMP DEFAULT now()
);

-- Monitor which hints are most effective
SELECT 
  qh.query_pattern,
  qh.hint_text,
  AVG(he.improvement_percent) as avg_improvement,
  COUNT(*) as usage_count,
  SUM(he.execution_without_hint_ms - he.execution_with_hint_ms) as total_time_saved
FROM query_hints qh
LEFT JOIN hint_effectiveness he ON qh.hint_id = he.hint_id
WHERE qh.enabled = true
GROUP BY qh.query_pattern, qh.hint_text
ORDER BY total_time_saved DESC;
```

---

## SECTION 79: ADVANCED TESTING STRATEGIES

### Q217: How do I implement chaos engineering for database resilience testing?

1. Define failure scenarios to test.
2. Implement controlled failure injection.
3. Monitor system behavior during failures.
4. Validate recovery procedures.
5. Document test results.
6. Improve based on findings.

```python
import random
import time
import subprocess
from datetime import datetime

class ChaosEngineer:
    def __init__(self, cluster_nodes):
        self.nodes = cluster_nodes
        self.test_results = []
    
    def inject_node_failure(self, node_name, duration_seconds=60):
        """Simulate node failure by stopping process"""
        test_id = f"node_failure_{node_name}_{datetime.now().isoformat()}"
        
        print(f"[{test_id}] Injecting failure: stopping {node_name}")
        
        start_time = time.time()
        subprocess.run(f"ssh {node_name} 'systemctl stop cockroachdb'", shell=True)
        
        # Monitor recovery
        recovery_complete = False
        while time.time() - start_time < duration_seconds + 60:
            try:
                result = subprocess.run(
                    f"ssh {node_name} 'cockroach sql -c SELECT 1'",
                    capture_output=True,
                    timeout=5
                )
                if result.returncode == 0:
                    recovery_time = time.time() - start_time - duration_seconds
                    recovery_complete = True
                    break
            except:
                pass
            
            time.sleep(5)
        
        # Restore node after duration
        if time.time() - start_time >= duration_seconds:
            print(f"[{test_id}] Restoring {node_name}")
            subprocess.run(f"ssh {node_name} 'systemctl start cockroachdb'", shell=True)
        
        # Record results
        test_result = {
            'test_id': test_id,
            'test_type': 'node_failure',
            'node': node_name,
            'failure_duration': duration_seconds,
            'recovery_complete': recovery_complete,
            'recovery_time_seconds': recovery_time if recovery_complete else None,
            'timestamp': datetime.now()
        }
        
        self.test_results.append(test_result)
        return test_result
    
    def inject_network_partition(self, node1, node2, duration_seconds=30):
        """Simulate network partition between nodes"""
        test_id = f"network_partition_{node1}_{node2}_{datetime.now().isoformat()}"
        
        print(f"[{test_id}] Injecting network partition between {node1} and {node2}")
        
        # Block traffic with iptables
        node1_ip = self.resolve_ip(node2)
        node2_ip = self.resolve_ip(node1)
        
        # Block on node1
        subprocess.run(
            f"ssh {node1} 'sudo iptables -A OUTPUT -d {node2_ip} -j DROP'",
            shell=True
        )
        
        # Block on node2
        subprocess.run(
            f"ssh {node2} 'sudo iptables -A OUTPUT -d {node1_ip} -j DROP'",
            shell=True
        )
        
        # Monitor cluster during partition
        partition_detected = False
        start_time = time.time()
        
        while time.time() - start_time < duration_seconds:
            status = self.check_cluster_status()
            if status['split_brain_detected']:
                partition_detected = True
            time.sleep(2)
        
        # Restore network
        print(f"[{test_id}] Restoring network partition")
        subprocess.run(
            f"ssh {node1} 'sudo iptables -D OUTPUT -d {node2_ip} -j DROP'",
            shell=True
        )
        subprocess.run(
            f"ssh {node2} 'sudo iptables -D OUTPUT -d {node1_ip} -j DROP'",
            shell=True
        )
        
        test_result = {
            'test_id': test_id,
            'test_type': 'network_partition',
            'nodes': [node1, node2],
            'duration': duration_seconds,
            'partition_detected': partition_detected,
            'timestamp': datetime.now()
        }
        
        self.test_results.append(test_result)
        return test_result
    
    def inject_high_latency(self, node_name, latency_ms=500, duration_seconds=60):
        """Simulate high network latency"""
        test_id = f"high_latency_{node_name}_{datetime.now().isoformat()}"
        
        print(f"[{test_id}] Injecting {latency_ms}ms latency on {node_name}")
        
        # Add latency with tc (traffic control)
        subprocess.run(
            f"ssh {node_name} 'sudo tc qdisc add dev eth0 root netem delay {latency_ms}ms'",
            shell=True
        )
        
        # Monitor impact
        metrics_before = self.get_cluster_metrics()
        time.sleep(duration_seconds)
        metrics_after = self.get_cluster_metrics()
        
        # Remove latency
        subprocess.run(
            f"ssh {node_name} 'sudo tc qdisc del dev eth0 root'",
            shell=True
        )
        
        test_result = {
            'test_id': test_id,
            'test_type': 'high_latency',
            'node': node_name,
            'latency_ms': latency_ms,
            'duration': duration_seconds,
            'metrics_impact': {
                'query_latency_increase_percent': (
                    (metrics_after['p99_latency'] - metrics_before['p99_latency']) / 
                    metrics_before['p99_latency'] * 100
                ),
                'throughput_decrease_percent': (
                    (metrics_before['throughput'] - metrics_after['throughput']) / 
                    metrics_before['throughput'] * 100
                )
            },
            'timestamp': datetime.now()
        }
        
        self.test_results.append(test_result)
        return test_result
    
    def inject_disk_full_scenario(self, node_name, duration_seconds=30):
        """Simulate disk space exhaustion"""
        test_id = f"disk_full_{node_name}_{datetime.now().isoformat()}"
        
        print(f"[{test_id}] Simulating disk full on {node_name}")
        
        # Create large file to consume disk space
        subprocess.run(
            f"ssh {node_name} 'dd if=/dev/zero of=/data/cockroach/fillfile bs=1M count=50000'",
            shell=True
        )
        
        # Monitor behavior (should go read-only)
        time.sleep(duration_seconds)
        
        # Recover by removing fill file
        subprocess.run(
            f"ssh {node_name} 'rm /data/cockroach/fillfile'",
            shell=True
        )
        
        test_result = {
            'test_id': test_id,
            'test_type': 'disk_full',
            'node': node_name,
            'duration': duration_seconds,
            'timestamp': datetime.now()
        }
        
        self.test_results.append(test_result)
        return test_result
    
    def run_chaos_test_suite(self):
        """Run comprehensive chaos test suite"""
        print("Starting chaos engineering test suite")
        
        # Test 1: Single node failure
        self.inject_node_failure(self.nodes[0], duration_seconds=60)
        time.sleep(120)  # Wait for recovery
        
        # Test 2: Network partition
        self.inject_network_partition(self.nodes[0], self.nodes[1], duration_seconds=30)
        time.sleep(60)
        
        # Test 3: High latency
        self.inject_high_latency(self.nodes[0], latency_ms=500, duration_seconds=60)
        time.sleep(60)
        
        # Test 4: Disk full
        self.inject_disk_full_scenario(self.nodes[1], duration_seconds=30)
        time.sleep(60)
        
        print("Chaos test suite complete")
        self.generate_report()
    
    def generate_report(self):
        """Generate chaos test report"""
        print("\n=== CHAOS ENGINEERING TEST REPORT ===\n")
        
        for result in self.test_results:
            print(f"Test: {result['test_id']}")
            print(f"Type: {result['test_type']}")
            print(f"Timestamp: {result['timestamp']}")
            
            if result['test_type'] == 'node_failure':
                print(f"Recovery: {'SUCCESS' if result['recovery_complete'] else 'FAILED'}")
                if result['recovery_complete']:
                    print(f"Recovery Time: {result['recovery_time_seconds']:.1f}s")
            
            elif result['test_type'] == 'high_latency':
                print(f"Query Latency Impact: {result['metrics_impact']['query_latency_increase_percent']:.1f}%")
                print(f"Throughput Impact: {result['metrics_impact']['throughput_decrease_percent']:.1f}%")
            
            print()
    
    def check_cluster_status(self):
        """Check cluster health during chaos"""
        # Implementation: query cluster status
        pass
    
    def get_cluster_metrics(self):
        """Get current cluster metrics"""
        # Implementation: fetch metrics
        pass
    
    def resolve_ip(self, hostname):
        """Resolve hostname to IP"""
        # Implementation: DNS resolution
        pass

# Usage
chaos = ChaosEngineer(['node1', 'node2', 'node3'])
chaos.run_chaos_test_suite()
```

### Q218: How do I implement performance regression testing in CI/CD?

1. Define baseline performance metrics.
2. Run performance tests on each commit.
3. Compare with baseline.
4. Alert if regression detected.
5. Block deployment if critical regression.
6. Track performance trends.

```yaml
# .github/workflows/performance-regression-test.yml
name: Performance Regression Testing

on:
  pull_request:
  push:
    branches: [main]

jobs:
  performance-test:
    runs-on: ubuntu-latest
    
    services:
      cockroachdb:
        image: cockroachdb/cockroach:latest
        options: >-
          --health-cmd="cockroach sql -c 'SELECT 1'"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 26257:26257
          - 8080:8080

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install psycopg2-binary pytest-benchmark pytest-xdist
      
      - name: Run performance baseline
        run: |
          python scripts/performance_baseline.py > baseline.json
      
      - name: Run performance regression tests
        run: |
          pytest tests/performance/ \
            --benchmark-json=results.json \
            --benchmark-compare=baseline.json \
            --benchmark-compare-fail=mean:10%
      
      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: |
            baseline.json
            results.json
      
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('results.json', 'utf8'));
            
            let comment = '## Performance Regression Test Results\n\n';
            comment += '| Test | Baseline | Current | Change |\n';
            comment += '|------|----------|---------|--------|\n';
            
            for (const benchmark of results.benchmarks) {
              const baseline = benchmark.group.baseline;
              const current = benchmark.stats.mean;
              const change = ((current - baseline) / baseline * 100).toFixed(2);
              
              comment += `| ${benchmark.name} | ${baseline.toFixed(2)}ms | ${current.toFixed(2)}ms | ${change}% |\n`;
            }
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

---

## SECTION 80: ADVANCED OPERATIONAL INSIGHTS

### Q219: How do I implement historical trend analysis for capacity planning?

1. Collect metrics continuously over time.
2. Store in time-series database.
3. Fit trend lines to historical data.
4. Project future resource needs.
5. Set capacity thresholds with buffer.
6. Trigger capacity planning when thresholds approached.

```python
import numpy as np
from datetime import datetime, timedelta
import psycopg2

class CapacityPlanner:
    def __init__(self, db_connection):
        self.db = db_connection
        self.lookback_days = 90
    
    def collect_historical_metrics(self):
        """Collect historical resource metrics"""
        cursor = self.db.cursor()
        
        cursor.execute("""
            SELECT 
              timestamp,
              cpu_percent,
              memory_percent,
              disk_percent,
              network_mbps
            FROM resource_metrics
            WHERE timestamp > now() - interval '%d days'
            ORDER BY timestamp
        """ % self.lookback_days)
        
        metrics = cursor.fetchall()
        return metrics
    
    def fit_trend_line(self, data_points):
        """Fit linear regression to metric data"""
        if len(data_points) < 2:
            return None
        
        x = np.array([i for i in range(len(data_points))])
        y = np.array([point[1] for point in data_points])
        
        # Linear regression
        z = np.polyfit(x, y, 1)
        p = np.poly1d(z)
        
        return {
            'slope': z[0],  # Growth rate
            'intercept': z[1],
            'model': p
        }
    
    def project_capacity_need(self, metric_name, current_value, max_capacity):
        """Project when capacity will be exceeded"""
        metrics = self.collect_historical_metrics()
        
        # Filter for specific metric
        if metric_name == 'cpu':
            data = [(m[0], m[1]) for m in metrics]
        elif metric_name == 'memory':
            data = [(m[0], m[2]) for m in metrics]
        elif metric_name == 'disk':
            data = [(m[0], m[3]) for m in metrics]
        else:
            return None
        
        trend = self.fit_trend_line(data)
        
        if trend is None or trend['slope'] <= 0:
            # Not growing
            return {'status': 'stable', 'days_to_capacity': None}
        
        # Calculate days to capacity
        growth_rate = trend['slope']
        remaining_capacity = max_capacity - current_value
        days_to_full = remaining_capacity / growth_rate if growth_rate > 0 else float('inf')
        
        return {
            'status': 'growing',
            'growth_rate_percent_per_day': growth_rate,
            'days_to_capacity': days_to_full,
            'projected_date': datetime.now() + timedelta(days=days_to_full)
        }
    
    def generate_capacity_report(self):
        """Generate capacity planning report"""
        thresholds = {
            'cpu': {'max': 100, 'warning': 80},
            'memory': {'max': 100, 'warning': 85},
            'disk': {'max': 100, 'warning': 85}
        }
        
        cursor = self.db.cursor()
        cursor.execute("""
            SELECT 
              cpu_percent,
              memory_percent,
              disk_percent
            FROM resource_metrics
            ORDER BY timestamp DESC
            LIMIT 1
        """)
        
        current = cursor.fetchone()
        
        report = {
            'timestamp': datetime.now(),
            'metrics': {}
        }
        
        # Analyze each metric
        for metric_name in ['cpu', 'memory', 'disk']:
            index = 0 if metric_name == 'cpu' else (1 if metric_name == 'memory' else 2)
            current_value = current[index]
            
            projection = self.project_capacity_need(
                metric_name,
                current_value,
                thresholds[metric_name]['max']
            )
            
            report['metrics'][metric_name] = {
                'current_value': current_value,
                'threshold_warning': thresholds[metric_name]['warning'],
                'status': 'WARNING' if current_value > thresholds[metric_name]['warning'] else 'OK',
                'projection': projection,
                'action': self.get_recommended_action(metric_name, current_value, projection)
            }
        
        return report
    
    def get_recommended_action(self, metric_name, current_value, projection):
        """Get recommended action based on projection"""
        if projection['status'] == 'stable':
            return 'No action needed - metrics stable'
        
        days_to_capacity = projection['days_to_capacity']
        
        if days_to_capacity < 7:
            return f'URGENT: Capacity will be exceeded in {days_to_capacity:.0f} days. Scale immediately.'
        elif days_to_capacity < 30:
            return f'HIGH PRIORITY: Capacity planning needed. Expected capacity exceeded in {days_to_capacity:.0f} days.'
        else:
            return f'Schedule capacity expansion for {projection["projected_date"].strftime("%Y-%m-%d")}'

# Usage
planner = CapacityPlanner(db_connection)
report = planner.generate_capacity_report()

print("=== CAPACITY PLANNING REPORT ===\n")
for metric, data in report['metrics'].items():
    print(f"{metric.upper()}: {data['status']}")
    print(f"  Current: {data['current_value']:.1f}%")
    print(f"  Action: {data['action']}")
    print()
```

### Q220: How do I implement anomaly detection for proactive alerting?

1. Establish normal baseline metrics.
2. Use statistical methods (z-score, IQR) to detect anomalies.
3. Implement machine learning models for complex patterns.
4. Alert before issues impact users.
5. Correlate anomalies across metrics.
6. Reduce false positives with context awareness.

```python
import numpy as np
from scipy import stats
import psycopg2
from datetime import datetime, timedelta

class AnomalyDetector:
    def __init__(self, db_connection, baseline_days=30):
        self.db = db_connection
        self.baseline_days = baseline_days
        self.sensitivity = 2.5  # Z-score threshold (2.5 = ~99% confidence)
    
    def collect_metric_history(self, metric_name, days=None):
        """Collect metric history for baseline"""
        if days is None:
            days = self.baseline_days
        
        cursor = self.db.cursor()
        cursor.execute(f"""
            SELECT timestamp, value
            FROM metrics
            WHERE metric_name = %s
              AND timestamp > now() - interval '{days} days'
            ORDER BY timestamp
        """, (metric_name,))
        
        return cursor.fetchall()
    
    def calculate_z_score_anomalies(self, metric_name):
        """Detect anomalies using z-score method"""
        history = self.collect_metric_history(metric_name, self.baseline_days)
        
        if len(history) < 2:
            return []
        
        values = np.array([h[1] for h in history])
        timestamps = [h[0] for h in history]
        
        # Calculate z-scores
        mean = np.mean(values)
        std_dev = np.std(values)
        
        if std_dev == 0:
            return []
        
        z_scores = np.abs((values - mean) / std_dev)
        
        # Identify anomalies
        anomalies = []
        for i, (timestamp, z_score, value) in enumerate(zip(timestamps, z_scores, values)):
            if z_score > self.sensitivity:
                anomalies.append({
                    'timestamp': timestamp,
                    'value': value,
                    'z_score': z_score,
                    'expected_range': (mean - 2*std_dev, mean + 2*std_dev),
                    'severity': 'HIGH' if z_score > 3 else 'MEDIUM'
                })
        
        return anomalies
    
    def detect_outliers_iqr(self, metric_name):
        """Detect outliers using Interquartile Range method"""
        history = self.collect_metric_history(metric_name)
        
        if len(history) < 4:
            return []
        
        values = np.array([h[1] for h in history])
        timestamps = [h[0] for h in history]
        
        # Calculate IQR
        Q1 = np.percentile(values, 25)
        Q3 = np.percentile(values, 75)
        IQR = Q3 - Q1
        
        lower_bound = Q1 - 1.5 * IQR
        upper_bound = Q3 + 1.5 * IQR
        
        outliers = []
        for timestamp, value in zip(timestamps, values):
            if value < lower_bound or value > upper_bound:
                outliers.append({
                    'timestamp': timestamp,
                    'value': value,
                    'lower_bound': lower_bound,
                    'upper_bound': upper_bound,
                    'deviation': max(value - upper_bound, lower_bound - value)
                })
        
        return outliers
    
    def detect_trend_anomaly(self, metric_name):
        """Detect sudden changes in trend"""
        history = self.collect_metric_history(metric_name, self.baseline_days)
        
        if len(history) < 7:
            return []
        
        values = np.array([h[1] for h in history])
        timestamps = [h[0] for h in history]
        
        # Split into two periods
        mid_point = len(values) // 2
        first_half = values[:mid_point]
        second_half = values[mid_point:]
        
        # Compare means
        mean1 = np.mean(first_half)
        mean2 = np.mean(second_half)
        
        # T-test for significance
        t_stat, p_value = stats.ttest_ind(first_half, second_half)
        
        trend_change = None
        if p_value < 0.05:  # Statistically significant
            change_percent = ((mean2 - mean1) / mean1 * 100) if mean1 != 0 else 0
            
            trend_change = {
                'period1_mean': mean1,
                'period2_mean': mean2,
                'change_percent': change_percent,
                'change_type': 'INCREASE' if mean2 > mean1 else 'DECREASE',
                'significance': 'HIGH' if p_value < 0.01 else 'MEDIUM'
            }
        
        return trend_change if trend_change else []
    
    def correlate_anomalies(self, anomalies_by_metric):
        """Correlate anomalies across multiple metrics"""
        correlations = []
        
        metric_names = list(anomalies_by_metric.keys())
        
        for i, metric1 in enumerate(metric_names):
            for metric2 in metric_names[i+1:]:
                if (anomalies_by_metric.get(metric1) and 
                    anomalies_by_metric.get(metric2)):
                    
                    # Check if anomalies occurred close in time
                    time_diff = abs(
                        (anomalies_by_metric[metric1][0]['timestamp'] -
                         anomalies_by_metric[metric2][0]['timestamp']).total_seconds()
                    )
                    
                    if time_diff < 300:  # Within 5 minutes
                        correlations.append({
                            'metrics': [metric1, metric2],
                            'time_diff_seconds': time_diff,
                            'likely_cause': self.infer_cause(metric1, metric2)
                        })
        
        return correlations
    
    def infer_cause(self, metric1, metric2):
        """Infer likely cause of correlated anomalies"""
        causation_map = {
            ('cpu', 'memory'): 'High process load',
            ('cpu', 'disk_io'): 'Disk contention',
            ('memory', 'swap_usage'): 'Memory pressure',
            ('network', 'latency'): 'Network congestion'
        }
        
        key = tuple(sorted([metric1, metric2]))
        return causation_map.get(key, 'Investigate correlation')
    
    def run_anomaly_detection(self):
        """Run comprehensive anomaly detection"""
        metrics = ['cpu', 'memory', 'disk', 'network', 'latency']
        
        results = {
            'timestamp': datetime.now(),
            'anomalies_by_type': {},
            'anomalies_by_metric': {},
            'correlations': []
        }
        
        all_anomalies = {}
        
        # Run detection on each metric
        for metric in metrics:
            z_score_anomalies = self.calculate_z_score_anomalies(metric)
            iqr_anomalies = self.detect_outliers_iqr(metric)
            trend_anomaly = self.detect_trend_anomaly(metric)
            
            results['anomalies_by_metric'][metric] = {
                'z_score': z_score_anomalies,
                'iqr': iqr_anomalies,
                'trend': trend_anomaly
            }
            
            if z_score_anomalies or iqr_anomalies or trend_anomaly:
                all_anomalies[metric] = z_score_anomalies
        
        # Correlate anomalies
        if len(all_anomalies) > 1:
            results['correlations'] = self.correlate_anomalies(all_anomalies)
        
        return results

# Usage
detector = AnomalyDetector(db_connection)
results = detector.run_anomaly_detection()

print("=== ANOMALY DETECTION RESULTS ===\n")
for metric, anomalies in results['anomalies_by_metric'].items():
    if any(anomalies.values()):
        print(f"\n{metric.upper()}:")
        if anomalies['z_score']:
            print(f"  Z-Score Anomalies: {len(anomalies['z_score'])}")
        if anomalies['iqr']:
            print(f"  IQR Outliers: {len(anomalies['iqr'])}")
        if anomalies['trend']:
            print(f"  Trend Change: {anomalies['trend']}")

if results['correlations']:
    print(f"\nCorrelated Anomalies:")
    for corr in results['correlations']:
        print(f"  {corr['metrics']}: {corr['likely_cause']}")
```

---

## SECTION 81: FINAL COMPREHENSIVE TOPICS

### Q221: How do I implement cost optimization across entire infrastructure?

1. Monitor cost per transaction and operation.
2. Identify expensive queries.
3. Right-size infrastructure for workload.
4. Implement reserved capacity where predictable.
5. Use spot/preemptible instances for fault-tolerant workloads.
6. Archive old data to reduce active storage.

```python
class CostOptimizer:
    def __init__(self):
        self.cost_per_unit = {
            'compute_per_vcpu_hour': 1.00,
            'storage_per_gb_month': 0.023,
            'backup_per_gb_month': 0.01,
            'network_per_gb': 0.12
        }
    
    def analyze_cost_by_workload(self, workload_metrics):
        """Analyze cost breakdown by workload"""
        costs = {
            'compute': workload_metrics['vcpu_hours'] * self.cost_per_unit['compute_per_vcpu_hour'],
            'storage': workload_metrics['storage_gb'] * self.cost_per_unit['storage_per_gb_month'],
            'backup': workload_metrics['backup_gb'] * self.cost_per_unit['backup_per_gb_month'],
            'network': workload_metrics['network_gb'] * self.cost_per_unit['network_per_gb']
        }
        
        costs['total'] = sum(costs.values())
        costs['cost_per_transaction'] = costs['total'] / workload_metrics['transactions']
        
        return costs
    
    def identify_cost_saving_opportunities(self, current_config):
        """Identify potential cost savings"""
        opportunities = []
        
        # Check for oversized instances
        if current_config['avg_cpu_utilization'] < 30:
            opportunities.append({
                'type': 'Downsize Compute',
                'potential_savings': current_config['monthly_compute_cost'] * 0.3,
                'action': 'Reduce node vCPU or instance count'
            })
        
        # Check for unused storage
        if current_config['storage_utilization'] < 50:
            opportunities.append({
                'type': 'Archive Old Data',
                'potential_savings': current_config['monthly_storage_cost'] * 0.2,
                'action': 'Move data older than 6 months to cold storage'
            })
        
        # Check for backup optimization
        if current_config['backup_retention_days'] > 60:
            opportunities.append({
                'type': 'Optimize Backup Retention',
                'potential_savings': current_config['monthly_backup_cost'] * 0.4,
                'action': 'Reduce retention from {} to 30 days'.format(
                    current_config['backup_retention_days']
                )
            })
        
        return opportunities
    
    def implement_cost_saving(self, saving_type, config):
        """Implement cost saving measure"""
        if saving_type == 'Downsize Compute':
            # Gradually reduce node count
            new_node_count = max(3, config['current_nodes'] - 1)
            return {'new_nodes': new_node_count}
        
        elif saving_type == 'Archive Old Data':
            # Archive data older than 6 months
            cutoff_date = datetime.now() - timedelta(days=180)
            return {'archive_date': cutoff_date}
        
        elif saving_type == 'Optimize Backup Retention':
            return {'new_retention_days': 30}

# Usage
optimizer = CostOptimizer()

workload = {
    'vcpu_hours': 8 * 730,  # 8 vCPU for full month
    'storage_gb': 500,
    'backup_gb': 1500,
    'network_gb': 100,
    'transactions': 1000000
}

costs = optimizer.analyze_cost_by_workload(workload)
print(f"Total Monthly Cost: ${costs['total']:.2f}")
print(f"Cost per Transaction: ${costs['cost_per_transaction']:.4f}")

current_config = {
    'avg_cpu_utilization': 25,
    'storage_utilization': 40,
    'monthly_compute_cost': 5840,
    'monthly_storage_cost': 11.50,
    'monthly_backup_cost': 15.00,
    'backup_retention_days': 90,
    'current_nodes': 5
}

opportunities = optimizer.identify_cost_saving_opportunities(current_config)
total_potential_savings = sum(o['potential_savings'] for o in opportunities)

print(f"\nCost Saving Opportunities: ${total_potential_savings:.2f}/month")
for opp in opportunities:
    print(f"  - {opp['type']}: ${opp['potential_savings']:.2f}")
    print(f"    Action: {opp['action']}")
```

### Q222: How do I implement automated performance tuning?

1. Collect query execution metrics continuously.
2. Identify slow queries automatically.
3. Generate index recommendations.
4. Test recommendations in staging.
5. Apply optimizations automatically (with approval).
6. Monitor effectiveness and adjust.

```python
class AutomaticTuning:
    def __init__(self, db_connection):
        self.db = db_connection
        self.tuning_history = []
    
    def identify_slow_queries(self, percentile=95, threshold_ms=100):
        """Identify queries slower than threshold"""
        cursor = self.db.cursor()
        
        cursor.execute(f"""
            SELECT 
              query,
              COUNT(*) as execution_count,
              AVG(latency_ms) as avg_latency,
              PERCENTILE_CONT({percentile/100}) WITHIN GROUP (ORDER BY latency_ms) as p{percentile}_latency,
              MIN(latency_ms) as min_latency,
              MAX(latency_ms) as max_latency
            FROM query_metrics
            WHERE timestamp > now() - interval '1 day'
            GROUP BY query
            HAVING PERCENTILE_CONT({percentile/100}) WITHIN GROUP (ORDER BY latency_ms) > {threshold_ms}
            ORDER BY p{percentile}_latency DESC
            LIMIT 20
        """)
        
        slow_queries = cursor.fetchall()
        return slow_queries
    
    def analyze_query_plan(self, query_text):
        """Analyze query execution plan"""
        cursor = self.db.cursor()
        
        cursor.execute(f"EXPLAIN (ANALYZE, FORMAT JSON) {query_text}")
        plan = cursor.fetchone()[0]
        
        analysis = {
            'query': query_text,
            'execution_time_ms': plan['Execution Time'],
            'total_cost': plan['Plan']['Total Cost'],
            'rows_returned': plan['Plan']['Actual Rows'],
            'plan_details': plan['Plan']
        }
        
        return analysis
    
    def generate_index_recommendations(self, query_text):
        """Generate index recommendations for slow query"""
        # Parse query to identify WHERE clauses and JOIN conditions
        # Check existing indexes
        # Recommend missing indexes
        
        recommendations = []
        
        # Simple heuristic: suggest index on first WHERE clause column
        import re
        
        where_match = re.search(r'WHERE\s+(\w+)\s*=', query_text, re.IGNORECASE)
        if where_match:
            column = where_match.group(1)
            recommendations.append({
                'type': 'Single Column Index',
                'sql': f"CREATE INDEX idx_{column} ON table_name({column});",
                'expected_improvement': '30-50%',
                'cost': 'Low'
            })
        
        return recommendations
    
    def test_recommendation_in_staging(self, recommendation):
        """Test index recommendation in staging environment"""
        staging_db = self._connect_to_staging()
        
        # Create index in staging
        staging_db.execute(recommendation['sql'])
        
        # Re-run slow query in staging
        execution_time_before = self._measure_query_time(staging_db, recommendation['query'])
        
        # With index
        execution_time_after = self._measure_query_time_with_index(staging_db, recommendation['query'])
        
        improvement_percent = ((execution_time_before - execution_time_after) / execution_time_before) * 100
        
        return {
            'recommendation': recommendation,
            'improvement_percent': improvement_percent,
            'is_beneficial': improvement_percent > 10
        }
    
    def apply_tuning_change(self, tuning_recommendation, approval_required=True):
        """Apply tuning change to production"""
        if approval_required:
            # Request approval (email, Slack, etc.)
            self.notify_approval(tuning_recommendation)
            return {'status': 'awaiting_approval'}
        
        cursor = self.db.cursor()
        
        try:
            cursor.execute(tuning_recommendation['sql'])
            self.db.commit()
            
            tuning_event = {
                'timestamp': datetime.now(),
                'type': tuning_recommendation['type'],
                'sql': tuning_recommendation['sql'],
                'status': 'success'
            }
        
        except Exception as e:
            tuning_event = {
                'timestamp': datetime.now(),
                'type': tuning_recommendation['type'],
                'sql': tuning_recommendation['sql'],
                'status': 'failed',
                'error': str(e)
            }
        
        self.tuning_history.append(tuning_event)
        return tuning_event
    
    def monitor_tuning_effectiveness(self, tuning_change):
        """Monitor if tuning change had desired effect"""
        cursor = self.db.cursor()
        
        cursor.execute(f"""
            SELECT 
              AVG(latency_ms) as current_avg_latency,
              PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as current_p99
            FROM query_metrics
            WHERE query = %s
              AND timestamp > %s
        """, (tuning_change['query'], tuning_change['applied_at']))
        
        current_metrics = cursor.fetchone()
        
        effectiveness = {
            'tuning_change': tuning_change['id'],
            'before_avg_latency': tuning_change['baseline_avg_latency'],
            'after_avg_latency': current_metrics[0],
            'improvement_percent': (
                (tuning_change['baseline_avg_latency'] - current_metrics[0]) / 
                tuning_change['baseline_avg_latency'] * 100
            ),
            'monitoring_complete': True
        }
        
        return effectiveness
    
    def run_automatic_tuning_cycle(self):
        """Run complete automatic tuning cycle"""
        print("Starting automatic tuning cycle...")
        
        # Step 1: Identify slow queries
        slow_queries = self.identify_slow_queries()
        print(f"Found {len(slow_queries)} slow queries")
        
        # Step 2: Analyze and recommend
        for query_text, exec_count, avg_latency, p95_latency, min_lat, max_lat in slow_queries:
            analysis = self.analyze_query_plan(query_text)
            recommendations = self.generate_index_recommendations(query_text)
            
            # Step 3: Test recommendations
            for rec in recommendations:
                test_result = self.test_recommendation_in_staging(rec)
                
                if test_result['is_beneficial']:
                    # Step 4: Apply (with approval)
                    result = self.apply_tuning_change(rec, approval_required=True)
                    print(f"Applied tuning: {result}")

# Usage
tuner = AutomaticTuning(db_connection)
tuner.run_automatic_tuning_cycle()
```

---

## SECTION 82: FINAL STRATEGIC GUIDANCE

### Q223: How do I establish database governance and standards?

1. Define SQL style guide and naming conventions.
2. Implement query review process.
3. Enforce schema change procedures.
4. Document architecture decisions.
5. Regular audits for compliance.
6. Training for developers on best practices.

```markdown
# CockroachDB Governance Framework

## SQL Style Guide

### Naming Conventions
- Tables: plural, snake_case (customers, orders)
- Columns: singular, snake_case (customer_id, order_date)
- Indexes: idx_table_column (idx_orders_customer_id)
- Constraints: fk_table_reference (fk_orders_customers)

### Query Standards
- Always use column list in INSERT (no INSERT INTO table VALUES)
- Use explicit JOINs (no comma joins)
- Order results predictably: ORDER BY id
- Use aliases for clarity in joins

### Schema Change Procedures
1. Design change, document in ticket
2. Create schema migration script
3. Test on staging environment
4. Request peer review
5. Schedule deployment window
6. Execute migration with rollback plan
7. Verify post-migration
8. Document actual changes

## Architecture Decision Record (ADR)
- Why: Business driver
- When: Expected timeline
- Who: Stakeholder approval
- Impact: Scale/cost/performance implications
- Alternatives: What else was considered

## Code Review Checklist
- [ ] No N+1 queries
- [ ] Appropriate indexes present
- [ ] Query complexity reasonable (EXPLAIN output reviewed)
- [ ] Follows naming conventions
- [ ] Documentation updated
- [ ] No hardcoded values or credentials
- [ ] Schema changes approved by DBA

## Compliance Audits
- Monthly: Query performance review
- Quarterly: Schema audit for unused objects
- Quarterly: Security review (access, encryption)
- Annually: Full infrastructure audit
```

### Q224: How do I build resilient database applications?

1. Implement circuit breakers for database calls.
2. Use connection pooling with proper timeout.
3. Implement retry logic with exponential backoff.
4. Cache frequently accessed data.
5. Implement fallback strategies.
6. Monitor application-database interactions.

```python
from functools import wraps
from datetime import datetime, timedelta
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout_seconds=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout_seconds
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker protection"""
        if self.state == 'OPEN':
            if self._should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                raise Exception('Circuit breaker is OPEN')
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        
        except Exception as e:
            self.on_failure()
            raise
    
    def on_success(self):
        """Handle successful call"""
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        """Handle failed call"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
    
    def _should_attempt_reset(self):
        """Check if enough time has passed to attempt reset"""
        if self.last_failure_time is None:
            return True
        
        return datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout)

class ResilientDB:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.circuit_breaker = CircuitBreaker()
        self.cache = {}
        self.cache_ttl = 300  # 5 minutes
    
    def execute_with_resilience(self, query, params=None, cache_key=None, ttl=None):
        """Execute query with full resilience"""
        # Check cache first
        if cache_key and cache_key in self.cache:
            cache_entry = self.cache[cache_key]
            if datetime.now() - cache_entry['timestamp'] < timedelta(seconds=ttl or self.cache_ttl):
                return cache_entry['data']
        
        # Retry with exponential backoff
        max_retries = 3
        for attempt in range(max_retries):
            try:
                # Use circuit breaker
                result = self.circuit_breaker.call(
                    self._execute_query,
                    query,
                    params
                )
                
                # Cache result
                if cache_key:
                    self.cache[cache_key] = {
                        'data': result,
                        'timestamp': datetime.now()
                    }
                
                return result
            
            except Exception as e:
                if attempt < max_retries - 1:
                    # Exponential backoff
                    wait_time = (2 ** attempt) + (time.time() % 1)
                    time.sleep(wait_time)
                else:
                    # Last attempt failed, try fallback
                    return self.get_fallback_result(query, cache_key)
    
    def _execute_query(self, query, params):
        """Execute actual query"""
        conn = psycopg2.connect(self.connection_string)
        cursor = conn.cursor()
        cursor.execute(query, params)
        return cursor.fetchall()
    
    def get_fallback_result(self, query, cache_key):
        """Fallback to stale data if available"""
        if cache_key and cache_key in self.cache:
            return self.cache[cache_key]['data']
        
        # No fallback available
        raise Exception('Database unavailable and no fallback data')

# Usage
db = ResilientDB('postgresql://user:pass@localhost:26257/mydb')

# Query with resilience
result = db.execute_with_resilience(
    "SELECT * FROM users WHERE id = %s",
    params=[123],
    cache_key='user:123',
    ttl=600
)
```

### Q225: Recommendations for production success

**SUCCESS FACTORS:**
1. Start with clear architecture design
2. Invest in monitoring from day one
3. Test failure scenarios regularly
4. Maintain comprehensive documentation
5. Foster strong operations culture
6. Continuous improvement mindset

**COMMON PITFALLS TO AVOID:**
- Skipping performance testing before launch
- Insufficient monitoring setup
- Poor network design (single path)
- Inadequate backup testing
- No runbooks for common issues
- Inadequate team training

**ONGOING RESPONSIBILITIES:**
- Weekly: Monitor dashboards and alerts
- Monthly: Review slow queries and optimize
- Quarterly: Load test and capacity planning
- Annually: Full infrastructure audit
- Continuously: Learn and share knowledge

---

# SECTION 83: ADVANCED SECURITY AND COMPLIANCE

### Q226: How do I implement data classification and handling policies?

1. Classify data by sensitivity level.
2. Define handling rules per classification.
3. Implement access controls based on classification.
4. Audit data access patterns.
5. Enforce encryption for sensitive data.
6. Monitor compliance continuously.

```sql
-- Data classification schema
CREATE TABLE data_classification (
  classification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  table_name VARCHAR,
  column_name VARCHAR,
  sensitivity_level VARCHAR,  -- public, internal, confidential, restricted
  encryption_required BOOLEAN,
  access_policy VARCHAR,
  created_at TIMESTAMP DEFAULT now()
);

-- Insert classification rules
INSERT INTO data_classification (table_name, column_name, sensitivity_level, encryption_required, access_policy)
VALUES 
  ('customers', 'email', 'internal', false, 'admin, sales'),
  ('customers', 'ssn', 'restricted', true, 'admin, compliance'),
  ('orders', 'amount', 'internal', false, 'admin, finance, sales'),
  ('orders', 'credit_card', 'restricted', true, 'admin, compliance');

-- Access control enforcement
CREATE VIEW data_access_policy AS
SELECT 
  dc.table_name,
  dc.column_name,
  dc.sensitivity_level,
  current_user as accessing_user,
  CASE 
    WHEN dc.sensitivity_level = 'public' THEN true
    WHEN dc.sensitivity_level = 'internal' AND current_user IN (SELECT role FROM user_roles) THEN true
    WHEN dc.sensitivity_level = 'confidential' AND current_user IN (SELECT role FROM confidential_access) THEN true
    WHEN dc.sensitivity_level = 'restricted' AND current_user IN (SELECT role FROM restricted_access) THEN true
    ELSE false
  END as access_allowed
FROM data_classification dc;

-- Audit data access
CREATE TABLE data_access_audit (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_name VARCHAR,
  table_name VARCHAR,
  column_name VARCHAR,
  operation VARCHAR,  -- SELECT, INSERT, UPDATE, DELETE
  row_count INT,
  accessed_at TIMESTAMP DEFAULT now(),
  classification_level VARCHAR
);

-- Create audit trigger
CREATE TRIGGER audit_classified_data
AFTER SELECT ON customers
FOR EACH ROW
EXECUTE FUNCTION log_classified_data_access();

-- Monitor sensitive data access
SELECT 
  user_name,
  COUNT(*) as access_count,
  COUNT(DISTINCT table_name) as tables_accessed,
  MAX(accessed_at) as last_access
FROM data_access_audit
WHERE classification_level IN ('confidential', 'restricted')
  AND accessed_at > now() - interval '24 hours'
GROUP BY user_name
HAVING COUNT(*) > 10;  -- Unusual access pattern
```

### Q227: How do I implement regulatory compliance automation?

1. Define compliance requirements (GDPR, HIPAA, SOC2).
2. Map requirements to database controls.
3. Automate compliance checks.
4. Generate compliance reports.
5. Alert on compliance violations.
6. Archive compliance evidence.

```sql
-- Compliance framework
CREATE TABLE compliance_requirements (
  requirement_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  regulation VARCHAR,  -- GDPR, HIPAA, SOC2, PCI-DSS
  requirement_name VARCHAR,
  requirement_description TEXT,
  control_type VARCHAR,
  control_description TEXT,
  automation_script TEXT,
  required BOOLEAN DEFAULT true
);

-- GDPR: Right to be forgotten
INSERT INTO compliance_requirements VALUES
  (gen_random_uuid(), 'GDPR', 'Right to be Forgotten', 
   'Delete all personal data when customer requests',
   'automated_delete',
   'DELETE FROM customers WHERE customer_id = ?',
   'SELECT COUNT(*) FROM customers WHERE customer_id = ?', true);

-- HIPAA: Audit logging
INSERT INTO compliance_requirements VALUES
  (gen_random_uuid(), 'HIPAA', 'Access Audit Logging',
   'Log all access to protected health information',
   'audit_logging',
   'Log SELECT queries on sensitive tables',
   'SELECT * FROM audit_log WHERE table_name IN (SELECT table_name FROM classified_data)', true);

-- PCI-DSS: Encryption at rest
INSERT INTO compliance_requirements VALUES
  (gen_random_uuid(), 'PCI-DSS', 'Encryption at Rest',
   'Encrypt payment card data at rest',
   'encryption',
   'Encrypt credit card columns',
   'SELECT setting FROM system.settings WHERE name = server.encryption_at_rest.enabled', true);

-- Automated compliance validation
CREATE TABLE compliance_validation_results (
  validation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  requirement_id UUID REFERENCES compliance_requirements(requirement_id),
  validation_status VARCHAR,  -- PASS, FAIL, WARNING
  validation_details JSONB,
  validated_at TIMESTAMP DEFAULT now()
);

-- Run compliance checks
CREATE FUNCTION validate_gdpr_compliance() RETURNS TABLE (
  requirement VARCHAR,
  status VARCHAR,
  details TEXT
) AS $$
BEGIN
  -- Check data residency
  RETURN QUERY
  SELECT 
    'Data Residency (EU)' as requirement,
    CASE WHEN (SELECT COUNT(*) FROM ranges WHERE region != 'eu') = 0 THEN 'PASS' ELSE 'FAIL' END as status,
    'Verify all EU customer data in EU regions' as details;
  
  -- Check encryption
  RETURN QUERY
  SELECT 
    'Encryption at Rest' as requirement,
    CASE WHEN (SELECT setting FROM system.settings WHERE name = 'server.encryption_at_rest.enabled') = 'true' THEN 'PASS' ELSE 'FAIL' END as status,
    'Verify encryption enabled' as details;
  
  -- Check audit logging
  RETURN QUERY
  SELECT 
    'Audit Logging' as requirement,
    CASE WHEN (SELECT COUNT(*) FROM event_log WHERE timestamp > now() - interval '24 hours') > 0 THEN 'PASS' ELSE 'FAIL' END as status,
    'Verify audit logs being generated' as details;
END $$ LANGUAGE plpgsql;

-- Generate compliance report
CREATE MATERIALIZED VIEW compliance_status_report AS
SELECT 
  cr.regulation,
  cr.requirement_name,
  cvr.validation_status,
  cvr.validated_at,
  CASE 
    WHEN cvr.validation_status = 'PASS' THEN 'Compliant'
    WHEN cvr.validation_status = 'FAIL' THEN 'Non-Compliant'
    WHEN cvr.validation_status = 'WARNING' THEN 'Needs Review'
  END as compliance_status
FROM compliance_requirements cr
LEFT JOIN compliance_validation_results cvr ON cr.requirement_id = cvr.requirement_id
ORDER BY cr.regulation, cr.requirement_name;

-- Alert on compliance violations
CREATE PROCEDURE alert_compliance_violations() AS $$
DECLARE
  v_violation RECORD;
BEGIN
  FOR v_violation IN
    SELECT cr.regulation, cr.requirement_name, cvr.validation_details
    FROM compliance_requirements cr
    JOIN compliance_validation_results cvr ON cr.requirement_id = cvr.requirement_id
    WHERE cvr.validation_status = 'FAIL'
      AND cvr.validated_at > now() - interval '1 hour'
  LOOP
    -- Send alert
    INSERT INTO alerts (severity, message, regulation)
    VALUES ('CRITICAL', 
            format('Compliance Violation: %s - %s', v_violation.regulation, v_violation.requirement_name),
            v_violation.regulation);
  END LOOP;
END $$ LANGUAGE plpgsql;
```

---

## SECTION 84: ADVANCED INFRASTRUCTURE PATTERNS

### Q228: How do I implement hybrid cloud deployment with CockroachDB?

1. Set up clusters across cloud providers (AWS, GCP, Azure).
2. Implement network connectivity between providers.
3. Manage multi-region failover.
4. Optimize data residency and latency.
5. Monitor cross-cloud performance.
6. Implement cost optimization across clouds.

```bash
#!/bin/bash
# Hybrid cloud deployment script

# Deploy CockroachDB across AWS, GCP, and on-premises

# AWS deployment
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --count 3 \
  --instance-type m5.2xlarge \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=crdb-aws-node}]' \
  --region us-east-1

# GCP deployment
gcloud compute instances create crdb-gcp-node-{1,2,3} \
  --machine-type=n1-standard-8 \
  --zone=us-central1-a \
  --image-family=ubuntu-2004-lts \
  --image-project=ubuntu-os-cloud

# On-premises deployment
# Deploy on existing infrastructure with proper networking

# Configure VPC peering and network connectivity
# AWS to GCP VPC peering
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-1a2b3c4d \
  --peer-vpc-id vpc-9f8e7d6c

# Setup firewall rules for cross-cloud communication
gcloud compute firewall-rules create allow-aws-cockroach \
  --allow tcp:26257 \
  --source-ranges 10.0.0.0/8  # AWS CIDR

# Join nodes across clouds
FIRST_NODE="aws-node-1.region.compute.amazonaws.com"

for node in gcp-node-1 gcp-node-2 on-prem-node-1; do
  ssh $node "cockroach start \
    --join=${FIRST_NODE}:26257 \
    --locality=cloud=$(echo $node | cut -d- -f1),region=multi-cloud"
done

# Verify cluster health
cockroach sql -e "SELECT node_id, address, locality FROM crdb_internal.nodes;"

# Configure zone constraints for data residency
cockroach sql -e "
ALTER TABLE sensitive_data CONFIGURE ZONE USING
  num_replicas = 5,
  constraints = '[+cloud=aws, +cloud=gcp]',
  lease_preferences = '[[+cloud=aws]]';
"

# Monitor cross-cloud performance
cockroach sql -e "
SELECT 
  source_node_id,
  dest_node_id,
  crdb_internal.get_node_locality(source_node_id)::text as source_cloud,
  crdb_internal.get_node_locality(dest_node_id)::text as dest_cloud,
  latency_ms
FROM crdb_internal.inter_node_latency
ORDER BY latency_ms DESC;
"
```

### Q229: How do I implement disaster recovery across regions?

1. Set up primary and standby clusters in different regions.
2. Implement physical cluster replication.
3. Monitor replication lag continuously.
4. Test failover procedures regularly.
5. Implement automatic DNS failover.
6. Document recovery procedures clearly.

```python
class DisasterRecoveryManager:
    def __init__(self, primary_region, standby_region):
        self.primary = primary_region
        self.standby = standby_region
        self.replication_status = {}
    
    def setup_cluster_replication(self):
        """Set up physical replication between regions"""
        print(f"Setting up replication from {self.primary} to {self.standby}")
        
        # Configure primary cluster
        primary_conn = self.connect_to_region(self.primary)
        primary_cursor = primary_conn.cursor()
        
        # Enable cluster replication
        primary_cursor.execute("ALTER CLUSTER SET REPLICATION FACTOR = 2")
        
        # Configure standby cluster
        standby_conn = self.connect_to_region(self.standby)
        standby_cursor = standby_conn.cursor()
        
        # Join to primary
        standby_cursor.execute(f"ALTER CLUSTER JOIN {self.primary}:26257")
        
        print("Cluster replication configured")
    
    def monitor_replication_lag(self):
        """Monitor replication lag between regions"""
        primary_conn = self.connect_to_region(self.primary)
        primary_cursor = primary_conn.cursor()
        
        primary_cursor.execute("""
            SELECT status, replicated_time 
            FROM crdb_internal.cluster_replication_status
        """)
        
        status = primary_cursor.fetchone()
        
        replication_lag_ms = (datetime.now() - status['replicated_time']).total_seconds() * 1000
        
        self.replication_status = {
            'status': status['status'],
            'replicated_time': status['replicated_time'],
            'lag_ms': replication_lag_ms,
            'healthy': replication_lag_ms < 1000  # < 1 second lag is good
        }
        
        if replication_lag_ms > 5000:
            self.alert_high_replication_lag(replication_lag_ms)
        
        return self.replication_status
    
    def test_failover(self):
        """Test failover procedure without actual failover"""
        print("Testing failover procedure...")
        
        # Create test data in primary
        primary_conn = self.connect_to_region(self.primary)
        primary_cursor = primary_conn.cursor()
        
        test_id = str(uuid.uuid4())
        primary_cursor.execute(
            "INSERT INTO failover_test (id, test_value) VALUES (%s, %s)",
            (test_id, 'test')
        )
        primary_conn.commit()
        
        # Wait for replication
        time.sleep(2)
        
        # Verify data in standby
        standby_conn = self.connect_to_region(self.standby)
        standby_cursor = standby_conn.cursor()
        
        standby_cursor.execute(
            "SELECT * FROM failover_test WHERE id = %s",
            (test_id,)
        )
        
        result = standby_cursor.fetchone()
        
        if result:
            print("✓ Failover test successful - data replicated correctly")
            return True
        else:
            print("✗ Failover test failed - data not replicated")
            return False
    
    def execute_failover(self):
        """Execute actual failover to standby region"""
        print(f"EXECUTING FAILOVER to {self.standby}")
        
        # Step 1: Stop writes to primary
        print("Step 1: Stopping writes to primary...")
        primary_conn = self.connect_to_region(self.primary)
        primary_cursor = primary_conn.cursor()
        
        primary_cursor.execute("SET CLUSTER SETTING kv.closed_timestamp.enabled = false")
        
        # Step 2: Verify replication caught up
        print("Step 2: Verifying replication...")
        for i in range(30):  # Wait up to 30 seconds
            status = self.monitor_replication_lag()
            if status['lag_ms'] < 100:
                break
            time.sleep(1)
        
        if status['lag_ms'] > 100:
            print("⚠ Warning: Replication lag > 100ms, potential data loss")
        
        # Step 3: Promote standby
        print("Step 3: Promoting standby to primary...")
        standby_conn = self.connect_to_region(self.standby)
        standby_cursor = standby_conn.cursor()
        
        standby_cursor.execute("ALTER CLUSTER SET REPLICATION FACTOR = 1")
        
        # Step 4: Update DNS
        print("Step 4: Updating DNS...")
        self.update_dns_to_standby()
        
        # Step 5: Notify applications
        print("Step 5: Notifying applications...")
        self.notify_applications_of_failover()
        
        print("✓ Failover complete")
    
    def execute_failback(self):
        """Failback from standby to recovered primary"""
        print(f"Executing failback from {self.standby} to {self.primary}")
        
        # Verify primary is recovered
        if not self.is_primary_healthy():
            print("Primary not yet healthy, postponing failback")
            return False
        
        # Re-establish replication from new primary (standby)
        self.setup_cluster_replication()
        
        # Wait for replication to catch up
        for i in range(60):
            status = self.monitor_replication_lag()
            if status['healthy']:
                break
            time.sleep(1)
        
        # Switch DNS back to primary
        self.update_dns_to_primary()
        
        print("✓ Failback complete")
        return True
    
    def update_dns_to_standby(self):
        """Update DNS to point to standby cluster"""
        # Implementation: update Route53, Cloud DNS, etc.
        pass
    
    def update_dns_to_primary(self):
        """Update DNS to point to primary cluster"""
        # Implementation: revert DNS changes
        pass
    
    def is_primary_healthy(self):
        """Check if primary cluster is healthy"""
        try:
            primary_conn = self.connect_to_region(self.primary, timeout=5)
            primary_cursor = primary_conn.cursor()
            primary_cursor.execute("SELECT 1")
            return True
        except:
            return False
    
    def alert_high_replication_lag(self, lag_ms):
        """Alert if replication lag too high"""
        print(f"ALERT: High replication lag detected: {lag_ms}ms")
        # Send to monitoring system
    
    def notify_applications_of_failover(self):
        """Notify applications of failover"""
        # Send notifications to app teams
        pass
    
    def connect_to_region(self, region, timeout=10):
        """Connect to CockroachDB cluster in region"""
        # Implementation: connect based on region
        pass
```

---

## SECTION 85: ADVANCED OPTIMIZATION TECHNIQUES

### Q230: How do I implement query federation across multiple clusters?

1. Route queries to appropriate cluster based on data location.
2. Implement transparent query routing.
3. Handle distributed transactions carefully.
4. Monitor query performance across clusters.
5. Implement fallback for failed clusters.
6. Optimize data placement for query patterns.

```python
class QueryFederation:
    def __init__(self, clusters_config):
        self.clusters = clusters_config
        self.routing_rules = {}
        self.cache = {}
    
    def define_routing_rule(self, pattern, target_cluster, condition=None):
        """Define which cluster should handle query pattern"""
        self.routing_rules[pattern] = {
            'cluster': target_cluster,
            'condition': condition
        }
    
    def route_query(self, query, context=None):
        """Route query to appropriate cluster"""
        # Match against routing rules
        for pattern, rule in self.routing_rules.items():
            if self._matches_pattern(query, pattern):
                if rule['condition'] is None or rule['condition'](context):
                    return rule['cluster']
        
        # Default to primary cluster
        return 'primary'
    
    def execute_federated_query(self, query, params=None, context=None):
        """Execute query on appropriate cluster"""
        target_cluster = self.route_query(query, context)
        
        try:
            conn = self.get_connection(target_cluster)
            cursor = conn.cursor()
            cursor.execute(query, params)
            return cursor.fetchall()
        
        except Exception as e:
            # Fallback to primary cluster
            if target_cluster != 'primary':
                print(f"Fallback to primary from {target_cluster}")
                conn = self.get_connection('primary')
                cursor = conn.cursor()
                cursor.execute(query, params)
                return cursor.fetchall()
            else:
                raise
    
    def execute_distributed_transaction(self, operations):
        """Execute transaction across multiple clusters"""
        # Two-phase commit (2PC) implementation
        
        # Phase 1: Prepare
        prepare_results = {}
        for cluster_name, ops in operations.items():
            try:
                conn = self.get_connection(cluster_name)
                cursor = conn.cursor()
                
                # Start transaction
                cursor.execute("BEGIN")
                
                # Execute operations
                for op in ops:
                    cursor.execute(op['sql'], op['params'])
                
                # Prepare for commit
                prepare_results[cluster_name] = 'READY'
            
            except Exception as e:
                prepare_results[cluster_name] = 'FAILED'
        
        # Check if all clusters ready
        if all(v == 'READY' for v in prepare_results.values()):
            # Phase 2: Commit
            for cluster_name in operations.keys():
                conn = self.get_connection(cluster_name)
                cursor = conn.cursor()
                cursor.execute("COMMIT")
            
            return True
        else:
            # Rollback
            for cluster_name in operations.keys():
                conn = self.get_connection(cluster_name)
                cursor = conn.cursor()
                cursor.execute("ROLLBACK")
            
            raise Exception("Distributed transaction failed during prepare phase")
    
    def _matches_pattern(self, query, pattern):
        """Check if query matches routing pattern"""
        import re
        return re.search(pattern, query, re.IGNORECASE) is not None
    
    def get_connection(self, cluster_name):
        """Get connection to cluster"""
        # Implementation: connection pooling per cluster
        pass

# Usage
clusters = {
    'primary': {'host': 'primary.example.com', 'region': 'us-east'},
    'secondary': {'host': 'secondary.example.com', 'region': 'us-west'},
    'analytics': {'host': 'analytics.example.com', 'region': 'eu-west'}
}

federation = QueryFederation(clusters)

# Define routing rules
federation.define_routing_rule(
    r'SELECT.*FROM customers.*WHERE.*region = \'US\'',
    'secondary'
)

federation.define_routing_rule(
    r'SELECT.*FROM analytics.*',
    'analytics'
)

# Execute federated query
result = federation.execute_federated_query(
    "SELECT * FROM customers WHERE id = %s",
    (123,)
)
```

---

## SECTION 86: ADVANCED OPERATIONAL EXCELLENCE

### Q231: How do I implement continuous optimization culture?

1. Establish metrics and baselines for continuous improvement.
2. Create feedback loops from production to development.
3. Regular performance review meetings.
4. Implement blameless postmortems.
5. Share learnings across team.
6. Recognize and reward optimization efforts.

```markdown
# Continuous Optimization Framework

## Weekly Performance Review
- Review top 10 slowest queries
- Identify opportunities for optimization
- Track changes from previous week
- Document findings

## Monthly Deep Dives
- Analyze query patterns and trends
- Review cluster health metrics
- Assess capacity utilization
- Plan optimizations for next quarter

## Quarterly Strategic Reviews
- Review SLA compliance
- Assess infrastructure alignment with growth
- Plan major architectural changes
- Review cost optimization initiatives

## Optimization Workflow
1. Identify inefficiency (alerts, monitoring, reviews)
2. Quantify impact (measure current state)
3. Design solution (multiple approaches)
4. Test in staging (validate improvement)
5. Deploy with monitoring (track production impact)
6. Measure improvement (confirm expected gains)
7. Document learning (share with team)

## Metrics to Track
- Query performance improvement %
- Index creation impact
- Schema optimization results
- Cost reduction achieved
- Availability improvements
- Developer productivity gains

## Recognition Program
- Monthly optimization awards
- Share wins in all-hands meetings
- Document case studies
- Create learning resources
- Career advancement tied to contributions
```

### Q232: How do I implement proactive capacity management?

1. Monitor utilization continuously.
2. Project future needs based on growth.
3. Plan capacity before crunch.
4. Implement auto-scaling where appropriate.
5. Optimize utilization continuously.
6. Balance cost and performance.

```python
class ProactiveCapacityManager:
    def __init__(self, warning_threshold=80, critical_threshold=90):
        self.warning_threshold = warning_threshold
        self.critical_threshold = critical_threshold
        self.capacity_history = []
    
    def project_capacity_needs(self, metric_name, lookback_days=90):
        """Project when capacity will be exceeded"""
        import numpy as np
        from datetime import datetime, timedelta
        
        # Collect historical data
        history = self.collect_metric_history(metric_name, lookback_days)
        
        if len(history) < 30:
            return None
        
        # Extract time series
        dates = [h['timestamp'] for h in history]
        values = [h['value'] for h in history]
        
        # Fit trend line
        x = np.array(range(len(values)))
        y = np.array(values)
        z = np.polyfit(x, y, 1)  # Linear regression
        
        slope = z[0]  # Daily change rate
        
        if slope <= 0:
            return {'status': 'stable', 'action': 'monitor'}
        
        # Calculate days to threshold
        current_value = values[-1]
        days_to_warning = (self.warning_threshold - current_value) / slope if slope > 0 else float('inf')
        days_to_critical = (self.critical_threshold - current_value) / slope if slope > 0 else float('inf')
        
        return {
            'metric': metric_name,
            'current_value': current_value,
            'growth_rate': slope,
            'days_to_warning': max(0, days_to_warning),
            'days_to_critical': max(0, days_to_critical),
            'warning_date': datetime.now() + timedelta(days=days_to_warning),
            'critical_date': datetime.now() + timedelta(days=days_to_critical),
            'recommended_action': self.get_recommended_action(metric_name, days_to_warning)
        }
    
    def get_recommended_action(self, metric_name, days_to_warning):
        """Get recommended action based on days to warning"""
        if days_to_warning < 7:
            return 'URGENT: Scale immediately'
        elif days_to_warning < 30:
            return 'Scale within next 2 weeks'
        elif days_to_warning < 60:
            return 'Plan scaling for next month'
        else:
            return 'Monitor and plan for future growth'
    
    def auto_scale_if_needed(self, metric_name, current_value):
        """Automatically scale if necessary"""
        if current_value > self.warning_threshold:
            print(f"Auto-scaling triggered for {metric_name}")
            self.trigger_scale_up()
            return True
        return False
    
    def trigger_scale_up(self):
        """Trigger cluster scale up"""
        # Implementation: add nodes, increase instance size, etc.
        pass
    
    def collect_metric_history(self, metric_name, days):
        """Collect historical metric data"""
        # Implementation: query metrics database
        pass
    
    def generate_capacity_plan(self):
        """Generate comprehensive capacity plan"""
        metrics = ['cpu', 'memory', 'disk', 'network']
        plan = {
            'timestamp': datetime.now(),
            'projections': {}
        }
        
        for metric in metrics:
            projection = self.project_capacity_needs(metric)
            if projection:
                plan['projections'][metric] = projection
        
        # Generate recommendations
        plan['recommendations'] = self.generate_capacity_recommendations(plan['projections'])
        
        return plan
    
    def generate_capacity_recommendations(self, projections):
        """Generate specific capacity recommendations"""
        recommendations = []
        
        for metric, projection in projections.items():
            if projection.get('days_to_critical', float('inf')) < 60:
                recommendations.append({
                    'priority': 'HIGH',
                    'metric': metric,
                    'action': projection['recommended_action'],
                    'timeline': projection['critical_date']
                })
        
        return sorted(recommendations, key=lambda x: x['timeline'])

# Usage
capacity_mgr = ProactiveCapacityManager()

# Project needs
projection = capacity_mgr.project_capacity_needs('disk', lookback_days=90)
print(f"Disk capacity warning in {projection['days_to_warning']:.0f} days")
print(f"Recommended action: {projection['recommended_action']}")

# Generate plan
plan = capacity_mgr.generate_capacity_plan()
for rec in plan['recommendations']:
    print(f"\n{rec['priority']}: {rec['action']}")
    print(f"Timeline: {rec['timeline'].strftime('%Y-%m-%d')}")
```

---

## SECTION 87: FINAL STRATEGIC OPERATIONS

### Q233: How do I build a self-managing distributed database system?

1. Implement autonomous operations with guardrails.
2. Reduce manual intervention through automation.
3. Enable self-healing for common issues.
4. Implement autonomous scaling.
5. Use AI/ML for predictive operations.
6. Maintain human oversight and control.

### Q234: How do I prepare for exponential growth scenarios?

1. Design for 10x growth from start.
2. Plan multi-region expansion.
3. Implement tiered data strategy (hot/warm/cold).
4. Design for horizontal scaling.
5. Plan organizational growth alongside systems.
6. Document architectural decisions.

### Q235: How do I transition from manual to automated operations?

1. Start with automation of frequent manual tasks.
2. Implement runbook automation gradually.
3. Build confidence through testing.
4. Maintain manual procedures as fallback initially.
5. Monitor automated procedures closely.
6. Gradually reduce manual intervention.

**Phase 1 (Weeks 1-4): Quick Wins**
- Automate daily health checks
- Automate common alerts acknowledgment
- Automate routine queries

**Phase 2 (Weeks 5-12): Core Operations**
- Automate cluster scaling
- Automate backup procedures
- Automate failover detection

**Phase 3 (Months 4-6): Advanced Automation**
- Autonomous optimization
- Predictive capacity planning
- Self-healing procedures

---

## SECTION 88: FINAL PRODUCTION WISDOM

### Q236: What is required for true production readiness?

**TECHNICAL READINESS:**
✓ Infrastructure tested under load
✓ All failure scenarios practiced
✓ Monitoring in place and validated
✓ Backups tested and verified
✓ Disaster recovery procedure documented and rehearsed

**OPERATIONAL READINESS:**
✓ On-call rotation established
✓ Runbooks written for all procedures
✓ Team trained on operations
✓ Escalation paths documented
✓ Communication procedures in place

**BUSINESS READINESS:**
✓ SLAs defined with stakeholders
✓ Cost model understood
✓ Business continuity plan approved
✓ Insurance/compliance reviewed
✓ Customer communication plan ready

### Q237: How do I measure operational excellence?

**KEY METRICS:**
- Mean Time to Detect (MTTD): < 5 minutes
- Mean Time to Recover (MTTR): < 15 minutes
- Availability: > 99.9%
- Error rate: < 0.1%
- P99 latency: < 100ms
- Alert accuracy: > 95%

**CONTINUOUS IMPROVEMENT:**
- Weekly: Review slow queries
- Monthly: Review incidents and learnings
- Quarterly: Review SLA compliance
- Annually: Full architecture review

### Q238: What are common mistakes to avoid?

1. **Under-provisioning infrastructure** - Always have 20-30% headroom
2. **Insufficient monitoring** - Monitor before problems occur
3. **Poor network design** - Single point of failure is death knell
4. **Inadequate testing** - Test failures before they happen
5. **Skipping documentation** - Future you will thank you
6. **Neglecting team training** - Automation fails without understanding

### Q239: How do I stay current with database trends?

1. Follow CockroachDB changelog and blog
2. Attend database conferences
3. Join user communities
4. Experiment with new features in staging
5. Read technical papers and research
6. Share knowledge with team

### Q240: Final checklist before production launch

```
INFRASTRUCTURE
☐ 3+ nodes across availability zones
☐ Network redundancy configured
☐ DNS failover working
☐ Load balancing configured
☐ Storage capacity planned for growth

SECURITY
☐ TLS enabled end-to-end
☐ User authentication configured
☐ Audit logging enabled
☐ Encryption at rest enabled
☐ Firewall rules configured

MONITORING & ALERTS
☐ Prometheus metrics collecting
☐ Grafana dashboards created
☐ Alert rules configured
☐ On-call notification working
☐ Log aggregation active

BACKUP & RECOVERY
☐ Backup schedule configured
☐ Restore procedure tested
☐ Cross-region backups setup
☐ RTO/RPO validated
☐ Point-in-time recovery tested

OPERATIONS
☐ Runbooks completed
☐ Team trained and certified
☐ Escalation procedures documented
☐ Incident response plan ready
☐ Change management process in place

PERFORMANCE
☐ Load testing completed
☐ Baselines established
☐ Scaling procedures tested
☐ Query optimization complete
☐ Index strategy validated

COMPLIANCE
☐ Security audit passed
☐ Compliance requirements met
☐ Data residency verified
☐ Audit logging validated
☐ Backup compliance verified

HANDOFF
☐ Documentation complete
☐ Architecture diagrams updated
☐ Cost model documented
☐ Business case approved
☐ Go/no-go decision made
```

---

## SECTION 89: FINAL COMPREHENSIVE SUMMARY

### Q241-Q250: Advanced Topics Overview

**Q241: Multi-tenancy at massive scale**
- Database per tenant for total isolation
- Automated tenant onboarding
- Tenant-specific SLAs and billing
- Cross-tenant analytics capabilities
- Automated tenant scaling

**Q242: Machine learning integration**
- Use ML for capacity prediction
- Implement ML for anomaly detection
- Use ML for query optimization
- Implement predictive alerting
- Deploy ML models at database layer

**Q243: Real-time analytics at scale**
- Columnar storage optimization
- Vectorized query execution
- Real-time aggregations with CDC
- Sub-second dashboard updates
- Streaming data ingestion

**Q244: Advanced sharding strategies**
- Consistent hashing implementation
- Range-based sharding
- Directory-based sharding
- Hash-based sharding
- Shard rebalancing during growth

**Q245: Blockchain and immutability**
- Append-only table design
- Merkle tree verification
- Cryptographic proof of data integrity
- Immutable audit logs
- Event sourcing patterns

**Q246: Edge computing integration**
- Edge cluster deployment
- Eventual consistency handling
- Conflict resolution strategies
- Bandwidth optimization
- Edge-to-cloud synchronization

**Q247: API-first database design**
- GraphQL support
- REST API generation
- Query language abstraction
- Client library generation
- Rate limiting and throttling

**Q248: Time-series data at scale**
- Specialized time-series tables
- Downsampling strategies
- Retention policies
- Real-time alerting on time-series
- Analytics on time-series data

**Q249: Financial data integrity**
- ACID guarantees for transactions
- Double-entry bookkeeping patterns
- Audit trail requirements
- Regulatory compliance
- Reconciliation procedures

**Q250: Future-ready database operations**
- Prepare for quantum-safe encryption
- Plan for AI/ML automation
- Design for serverless deployment
- Implement for edge computing
- Build for autonomous operations

---

## COMPREHENSIVE FAQ COMPLETION

**Coverage by Topic:**

1. **Cluster Setup & Management (Q1-Q25):** 25 FAQs
2. **Backup & Disaster Recovery (Q26-Q80):** 55 FAQs
3. **Security & Compliance (Q81-Q140):** 60 FAQs
4. **Performance Tuning (Q141-Q160):** 20 FAQs
5. **Advanced Operations (Q161-Q210):** 50 FAQs
6. **Migration & Integration (Q211-Q230):** 20 FAQs
7. **Advanced Topics (Q231-Q250):** 20 FAQs

**Total: 250 FAQs covering all aspects of CockroachDB administration**

---

## FINAL NOTES

This comprehensive guide contains:
- 250+ questions covering complete CockroachDB administration
- Real-world scenarios and case studies
- Production-ready procedures and checklists
- Security and compliance frameworks
- Performance optimization techniques
- Disaster recovery strategies
- Operational excellence principles
- Advanced architectural patterns

**To use this guide effectively:**
1. Start with sections relevant to your current phase
2. Reference specific topics as needed
3. Use checklists before production launch
4. Share learnings with your team
5. Update based on your specific environment

**Next steps after production launch:**
1. Establish monitoring and alerting immediately
2. Practice failover procedures regularly
3. Optimize based on actual workload
4. Share knowledge with team
5. Continuously improve operations

**Remember:** Database administration is a journey, not a destination. Continuous learning and improvement are key to long-term success.

---

**END OF COMPREHENSIVE COCKROACHDB DATABASE ADMINISTRATION GUIDE**


