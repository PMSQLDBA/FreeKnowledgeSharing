Runbook: PostgreSQL Database Restore from Dump

Owner: Database Administration Team |
Last Updated: July 2026

Purpose

This runbook restores a PostgreSQL database from a dump file. Use this when recovering from a crash, migrating to a new server, or provisioning a test environment with production data. This process works for both plain SQL (.sql) and custom format (.dump) dumps.

Prerequisites

- [ ] PostgreSQL server installed and running
- [ ] Access to the dump file location
- [ ] Superuser or sufficient createdb privileges
- [ ] Command-line access to the target server
- [ ] No active connections to the target database
- [ ] Disk space equal to approximately 1.5x the dump file size

Procedure

Step 1: Verify the Dump Format

Run this command to identify the dump type:

```
file /path/to/dump_file.dump
```

Expected result: Output shows either "PostgreSQL custom database dump" or plain text.

If it fails: Confirm the file path is correct and you have read permissions.

Step 2: Create a Clean Target Database

Drop any existing target database to avoid conflicts:

```
dropdb -U postgres --if-exists target_database_name
```

Then create a fresh database:

```
createdb -U postgres target_database_name
```

Expected result: No error messages. The command completes silently on success.

If it fails: Check that you have createdb privileges. If you get "role does not exist," ensure the postgres user exists on the target server.

Step 3: Restore the Dump - Custom Format

If the dump is in custom format (.dump file), use this command:

```
pg_restore -U postgres -d target_database_name --no-owner --no-privileges /path/to/mydatabase.dump
```

Expected result: The command runs and completes without errors.

If it fails: Check the error message. Most failures relate to missing roles or permissions (see Troubleshooting).

Step 4: Restore the Dump - Plain SQL Format

If the dump is plain SQL (.sql file), use this command instead:

```
psql -U postgres -d target_database_name -f /path/to/mydatabase.sql
```

Expected result: psql outputs verbose results as it processes each SQL command. Restore time depends on database size.

If it fails: Check that the SQL file is readable and not corrupted.

Step 5: Restore Large Databases with Parallel Processing

For custom-format dumps over 500 MB, accelerate the restore using parallel jobs:

```
pg_restore -U postgres -d target_database_name --no-owner --no-privileges -j 4 /path/to/mydatabase.dump
```

Change -j 4 to match your CPU core count or available I/O resources.

Expected result: Restore runs in parallel and completes faster than sequential restore.

If it fails: Lower the job count to -j 2 or -j 1. Some environments have per-user connection limits that prevent parallel restores.

Step 6: Monitor Quiet Restores

If the restore hangs or shows no output, add verbose logging:

```
pg_restore -U postgres -d target_database_name --no-owner --no-privileges -v /path/to/mydatabase.dump
```

Expected result: Console displays progress as each table, index, and constraint is restored.

If it fails: Terminate the process with Ctrl+C, check server logs, and proceed to Troubleshooting.

Verification

- [ ] Connect to the restored database and list tables:
  ```
  psql -U postgres -d target_database_name -c "\dt"
  ```

- [ ] Verify table row counts match expectations:
  ```
  SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
  ```

- [ ] Spot-check critical tables for data integrity. Run a simple SELECT on a table you know contains data and confirm results.

- [ ] Verify database size roughly matches the original:
  ```
  SELECT pg_size_pretty(pg_database_size('target_database_name'));
  ```

Troubleshooting

Symptom: database does not exist
Likely Cause: Target database was not created.
Fix: Run Step 2 again to create the database.

Symptom: role "user_name" does not exist
Likely Cause: The dump contains role assignments from a different server. The --no-owner --no-privileges flags were not used.
Fix: Confirm both flags are in your restore command. If error persists, drop the database and restore again.

Symptom: out of shared memory
Likely Cause: max_locks_per_transaction setting is too low for the number of objects being restored.
Fix: Edit postgresql.conf and increase max_locks_per_transaction to 256 or higher. Restart PostgreSQL and retry.

Symptom: restore hangs with no progress output
Likely Cause: The restore is running but output is buffered. I/O bottleneck on disk.
Fix: Add -v flag for verbose output. Check disk I/O with iostat or similar. Consider reducing -j worker count if using parallel restore.

Symptom: permission denied for schema public
Likely Cause: Schema permissions are locked down. Occurs after restore completes.
Fix: Grant schema access: GRANT ALL ON SCHEMA public TO postgres; Then reassign object ownership as needed.

Symptom: checksum validation failed
Likely Cause: Dump file is corrupted or truncated.
Fix: Verify the dump file integrity by checking its size against a known good copy. Re-download or re-create the dump.

Rollback

If the restore fails or produces an incorrect result, revert immediately:

```
dropdb -U postgres target_database_name
```

This removes the bad restore. Then either retry the restore procedure or restore from a different backup.

If you accidentally restored over a production database, you cannot undo this. Contact your backup team to restore from a previous backup tape or snapshot.

Escalation

Situation: Restore fails repeatedly with role or permission errors
Contact: Database Administrator
Method: Email with error message and dump file details.

Situation: Restore completes but data is incorrect or incomplete
Contact: Database Administrator + Data Integrity Team
Method: Create a ticket with row count comparisons and sample queries showing the problem.

Situation: Production database was overwritten by restore
Contact: Incident Commander, Chief Technology Officer
Method: Page on-call immediately. This is a critical incident.

Situation: Restore time exceeds acceptable window
Contact: Performance Engineer
Method: Discuss parallel restore settings, disk configuration, or index rebuild strategy.

History

Date: [Date]
Run By: [Name]
Notes: [Restore completed successfully for staging environment]

Date: [Date]
Run By: [Name]
Notes: [Initial restore failed due to missing max_locks_per_transaction setting; resolved with postgresql.conf change]

Tips for Success

1. Always create a fresh target database. Never restore into an existing production database.

2. Use --no-owner --no-privileges unless you have identical role structure on both servers.

3. For large databases, test parallel restore first on a non-critical system to verify your -j setting works.

4. Always verify row counts after restore. Assume nothing.

5. Keep restore commands in a script file. This reduces typos and allows easy automation.

Sample Automation Script

Create a file named restore_db.sh:

```
#!/bin/bash
set -e

DB_NAME="production_copy"
DUMP_FILE="/backups/latest.dump"
DB_USER="postgres"

echo "Starting restore of $DB_NAME from $DUMP_FILE"

dropdb -U $DB_USER --if-exists $DB_NAME
createdb -U $DB_USER $DB_NAME

pg_restore -U $DB_USER -d $DB_NAME --no-owner --no-privileges -j 4 $DUMP_FILE

echo "Restore complete. Verifying..."
psql -U $DB_USER -d $DB_NAME -c "SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 10;"

echo "Restore of $DB_NAME finished successfully."
```

Run with: bash restore_db.sh
