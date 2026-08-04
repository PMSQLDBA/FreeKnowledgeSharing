# Technical SOP – SQL Server Distributed Availability Groups (DAG)

Reference: https://www.powershelldba.de/blog/articles/distributed-availability-groups.html

## Enterprise DBA Standard Operating Procedure (SOP) with Real-Time Migration Scenario

> **Audience:** SQL Server DBAs, Senior DBAs, Database Architects, Infrastructure Engineers

> **Objective:** Perform Zero-Downtime SQL Server Database Migration between Datacenters or Cloud using Distributed Availability Groups (DAG). The SOP below is based on the concepts and workflow described in the uploaded document. 

---

# 1. Business Requirement (Real-Time Scenario)

### Customer Environment

A financial organization hosts its production SQL Server environment in **New York Datacenter (Site A)**.

The organization plans to migrate to **Azure East US (Site B)** because:

* Existing hardware warranty is expiring.
* Storage capacity reached 90%.
* Azure provides better disaster recovery.
* Business cannot afford downtime.

Database Size

* 8 TB Production Database
* 24×7 Trading Application
* 4,000 Concurrent Users
* 500 Transactions/sec

Business Requirement

* Zero downtime
* No backup/restore during cutover
* No application outage
* Minimal risk
* Ability to rollback if required

Distributed Availability Group (DAG) is selected because it enables geographic migration with asynchronous replication between two separate Always On Availability Groups. 

---

# 2. Architecture

```
                DATACENTER A (Production)

             AG_PRODUCTION
          ---------------------
          SQL01  (Primary)
          SQL02  (Sync Secondary)
          SQL03  (Async Forwarder)
                 |
                 |
      Distributed Availability Group
      Asynchronous Replication
                 |
                 |
        AG_DR / AG_AZURE
      ---------------------
      SQL11 (Primary in AG)
      SQL12 (Sync Replica)
      SQL13 (Async Replica)

           Azure East US
```

Each site maintains its own Availability Group, and the DAG asynchronously replicates changes between the two AGs. 

---

# 3. Prerequisites Checklist

## SQL Server

✔ Enterprise Edition

✔ Windows Server Failover Cluster

✔ AlwaysOn Enabled

✔ Database in Full Recovery Mode

✔ AG Listener Created

✔ Endpoints Configured

✔ Database Health Check Completed

---

## Network

Port 5022 Open

Port 1433 Open

DNS Resolution Working

Latency Verified

Bandwidth Validated

---

## Security

Service Accounts

Certificates

Firewall Rules

Permissions

---

# 4. Migration Workflow

---

## Step 1

### Validate Existing Production AG

Check

```
AG Status

Replica Status

Synchronization

Listener

Failover Mode

Database Health
```

Example

```
Production AG

AG_PRODUCTION

Primary

SQL01

Secondary

SQL02

Healthy

YES
```

The production AG should be healthy before introducing the secondary site. 

---

## Step 2

### Build Secondary AG

Deploy

```
Windows Cluster

SQL Servers

Storage

AlwaysOn AG

Listener
```

Example

```
Azure

SQL11

SQL12

SQL13

AG_AZURE
```

Initially, the new AG is independent and seeded from production backups. 

---

## Step 3

### Restore Database

Take Full Backup

↓

Copy Backup

↓

Restore WITH NORECOVERY

↓

Join Database

---

## Step 4

### Create Distributed AG

Architecture

```
AG_PRODUCTION

↓

Distributed AG

↓

AG_AZURE
```

High-level syntax from the source:

```sql
CREATE DISTRIBUTED AVAILABILITY GROUP 'DAG_Prod'
PRIMARY AVAILABILITY GROUP 'AG_Site_A'
SECONDARY AVAILABILITY GROUP 'AG_Site_B'
LISTENER 'dag-listener';
```



---

## Step 5

### Verify Synchronization

Monitor

```
Redo Queue

Send Queue

Synchronization

LSN

Latency
```

Healthy Example

```
Redo Queue

0 MB

Send Queue

2 MB

Latency

1 Second
```

Wait until synchronization is acceptable before proceeding. 

---

## Step 6

### Application Testing

Application Team validates

Login

Dashboard

Transactions

Reports

ETL Jobs

API Calls

Read-only Queries

---

## Step 7

### Planned Migration

Maintenance Window

Although business users continue working,

DBA performs:

```
Freeze Deployment

Stop Batch Jobs

Confirm Sync

Promote Secondary AG

Redirect Listener

Resume Jobs
```

This aligns with the planned failover workflow where the secondary AG is promoted and client connections are redirected. 

---

## Step 8

### Redirect Applications

Old

```
Server=AGPRODListener
```

New

```
Server=AGAZUREListener
```

or

Update DNS Alias

Applications reconnect automatically based on connection strategy. 

---

# 5. DBA Validation Checklist

Before Cutover

```
✓ Backup Completed

✓ AG Healthy

✓ DAG Healthy

✓ No Blocking

✓ SQL Agent Jobs Disabled

✓ Replication Healthy

✓ Monitoring Enabled

✓ Users Informed
```

After Cutover

```
✓ Login Success

✓ Application Working

✓ Jobs Running

✓ Reports Running

✓ Performance Normal

✓ Blocking Normal

✓ Alerts Working
```

---

# 6. Monitoring During Migration

Continuously monitor:

```
CPU

Memory

Disk Latency

Network

Redo Queue

Send Queue

Transaction Log Growth

AlwaysOn Dashboard

SQL Error Log

Windows Event Log
```

Pay special attention to the forwarder replica log queue depth and inter-site latency. 

---

# 7. Rollback Plan

If Migration Fails

```
Stop Application

Redirect Connection

Point Back to

Production Listener

Resume Traffic
```

Since the original production AG remains available, rollback is typically quick.

---

# 8. Real-Time DBA Responsibilities

### Before Migration

* Review AG Health
* Verify Backups
* Confirm Synchronization
* Coordinate with Network Team
* Validate Application Readiness
* Confirm DNS Readiness

### During Migration

* Monitor Synchronization
* Execute Failover
* Monitor Blocking
* Watch SQL Error Log
* Validate User Connections
* Coordinate with Application Team

### After Migration

* Validate Jobs
* Check Replication
* Review Wait Statistics
* Review Blocking
* Confirm Performance
* Collect Sign-off

---

# 9. Common Issues and Resolutions

| Issue                      | Root Cause                    | Resolution                                     |
| -------------------------- | ----------------------------- | ---------------------------------------------- |
| High Send Queue            | Network latency               | Verify WAN bandwidth and latency               |
| Redo Queue Increasing      | Secondary overloaded          | Tune storage/CPU and investigate workload      |
| Listener Not Reachable     | DNS or firewall issue         | Validate listener configuration and networking |
| Application Cannot Connect | Connection string not updated | Redirect to the correct AG listener            |
| Synchronization Lag        | WAN congestion                | Delay cutover until synchronization stabilizes |

---

# 10. Advantages

* Zero-downtime migration
* Multi-datacenter architecture
* Disaster Recovery readiness
* Independent Availability Groups per site
* Local HA plus geographic redundancy
* Planned and unplanned failover capability
* Reduced migration risk

These benefits are the primary reasons organizations adopt Distributed AGs for cross-site migrations. 

---

# 11. Limitations

* Asynchronous replication between sites can result in some data loss during a catastrophic primary-site failure.
* Higher operational complexity because two Availability Groups must be managed.
* SQL Server Enterprise Edition licensing is required for all participating instances.
* Application connection logic must support listener changes, DNS updates, or retry behavior. 

---

# 12. Real-Time Production Example

**Scenario:** A financial services company migrated its 8 TB production SQL Server environment from an on-premises New York datacenter to Azure East US.

**Implementation:**

1. Built a new Always On Availability Group in Azure.
2. Seeded it with a production backup.
3. Created a Distributed Availability Group linking the on-premises AG to the Azure AG.
4. Monitored synchronization for two weeks.
5. Performed a planned promotion of the Azure AG.
6. Updated application connectivity automatically through infrastructure automation.

**Outcome:**

* Zero customer downtime
* No manual backup/restore during cutover
* Automatic application reconnection
* Successful migration with no business interruption

This mirrors the migration workflow described in the uploaded reference. 

---

## DBA Interview Tips

* Explain the difference between a standard Always On Availability Group and a Distributed Availability Group.
* Emphasize that a DAG connects **two independent Availability Groups**, not individual replicas.
* Be prepared to discuss **RPO/RTO trade-offs**, particularly the asynchronous nature of inter-site replication.
* Describe how you would monitor **forwarder replica health**, **send/redo queues**, **network latency**, and **application connectivity** during a migration.
* Walk through a complete migration runbook, including planning, validation, cutover, monitoring, and rollback.
