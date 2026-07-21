# SQL Server Contained Availability Groups (Contained AG)

## Complete Technical Guide — Fundamentals to Advanced 

Reference: 
https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/contained-availability-groups-overview?view=sql-server-ver17


# 1. Introduction to Contained Availability Groups

## What is a Contained Availability Group?

A **Contained Availability Group (Contained AG)** is an enhancement to **Always On Availability Groups** introduced in:

> **SQL Server 2022**

A traditional Availability Group protects **databases**, but many required objects still exist outside the database at the **SQL Server instance level**.

Contained AG extends the Availability Group boundary by including instance-level objects and settings inside the AG.

The goal:

> Make an Availability Group more portable, self-contained, and easier to migrate or manage.

---

# 2. Why Microsoft Introduced Contained AG?

## Traditional Availability Group Problem

In a normal AG:

```
                 Availability Group

        +----------------------------+
        |                            |
        |       User Database        |
        |                            |
        +----------------------------+


Outside AG:

SQL Instance
 |
 +-- Logins
 |
 +-- SQL Agent Jobs
 |
 +-- Credentials
 |
 +-- Linked Servers
 |
 +-- Permissions
 |
 +-- Configuration
```

The database moves/fails over, but many dependencies do not.

---

## Real Production Problem

Example:

Production:

```
SQLPROD01

Database:
    BankingDB

Login:
    App_Login

SQL Agent Job:
    BackupJob

Credential:
    AzureStorageCredential
```

Failover:

```
SQLPROD02 becomes Primary
```

Database comes online:

```
BankingDB
       ✅
```

But:

```
App_Login
       ❌ Missing

SQL Agent Job
       ❌ Missing

Credential
       ❌ Missing
```

DBA must manually synchronize objects.

---

# 3. Traditional AG vs Contained AG

## Traditional Availability Group

Boundary:

```
Availability Group
|
|
+-- Databases

Outside:

+-- Logins
+-- Jobs
+-- Credentials
+-- Server Configuration
```

---

## Contained Availability Group

Boundary:

```
Contained Availability Group

|
+-- Databases
|
+-- Users
|
+-- Logins
|
+-- SQL Agent Jobs
|
+-- Permissions
|
+-- Metadata
|
+-- Configuration
```

The AG becomes more portable.

---

# 4. Architecture of Contained AG

Example:

```
                 Application

                     |
                     |
              AG Listener

                     |
        +------------+------------------------------------+
        |                                                 |
        |                                                 |
 SQL Server Node 1                                  SQL Server Node 2
                     -------Synchronization--->
 Primary Replica                                     Secondary Replica

 
```

Inside the AG:

```
Contained Availability Group

|
|-- Database
|
|-- Login Metadata
|
|-- SQL Agent Jobs
|
|-- Credentials
|
|-- Permissions
|
|-- Replica Settings
```

---

# 5. How Contained AG Works Internally

Normally SQL Server stores metadata here:

```
master database
```

Examples:

Server login:

```
master.sys.server_principals
```

SQL Agent jobs:

```
msdb.dbo.sysjobs
```

Credentials:

```
master.sys.credentials
```

---

Contained AG creates a boundary where these objects are replicated with the AG.

Concept:

```
Traditional:

master
 |
 |
Instance Objects


Contained:

AG Metadata Container

 |
 |
Replicated with AG
```

---

# 6. Enabling Contained Availability Group

Syntax:

```sql
ALTER AVAILABILITY GROUP [AG_Name]
SET (CONTAINED = ON);
```

Example:

```sql
ALTER AVAILABILITY GROUP [FinanceAG]
SET
(
    CONTAINED = ON
);
```

Verify:

```sql
SELECT 
    name,
    contained
FROM sys.availability_groups;
```

Expected:

```
FinanceAG    1
```

---

# 7. Features Included in Contained AG

## 7.1 Contained Logins

### Traditional AG

Login exists on each SQL Server:

```
SQL01

Login:
AppUser


SQL02

Login:
???
```

DBA manually creates login.

---

Contained AG:

```
Contained AG

Login:
AppUser

Replicated automatically
```

Benefits:

* Avoid orphaned users
* Easier failover
* Easier migration

---

# 7.2 SQL Agent Job Containment

Problem:

Jobs are stored in:

```
msdb
```

Example:

```
Backup Job

Index Maintenance Job

ETL Job
```

Failover:

```
Primary SQL01
      |
      X

Secondary SQL02
```

Jobs may not exist.

---

Contained AG:

```
AG

|
+-- SQL Agent Jobs
```

Jobs move with AG.

---

# 7.3 Credential Containment

Example:

Backup to Azure Blob:

```
Credential:

AzureBackupCredential
```

Traditional:

Create manually:

```
SQL01
SQL02
SQL03
```

Contained AG:

Credential follows AG.

---

# 7.4 Database Scoped Configuration

Contained AG supports database-level behavior.

Examples:

```
Query Store Settings

Parameter Sensitive Plan Settings

Compatibility Settings
```

---

# 7.5 Metadata Synchronization

Contained AG synchronizes required metadata.

Example:

```
Primary Replica

Create Login

       |
       |
Transaction Log

       |
       |

Secondary Replica
```

---

# 8. Advantages of Contained AG

## Advantage 1: Simplified Failover

Traditional:

After failover:

```
Database
   +
Login synchronization
   +
Jobs synchronization
   +
Credentials
```

Contained AG:

```
Failover

Database
+
Required metadata
```

---

## Advantage 2: Easier Migration

Scenario:

```
SQL Server 2019

      |
      |
Migration

      |
      |

SQL Server 2022
```

Contained AG reduces manual migration steps.

---

## Advantage 3: Better Hybrid Cloud Support

Example:

On-prem:

```
SQL Server AG
```

Migration:

```
        |
        |
        V

Azure SQL VM
```

Contained objects travel with AG.

---

## Advantage 4: Reduces DBA Manual Tasks

Without Contained AG:

```
Create Login
Create Job
Create Credential
Sync Permissions
Validate
```

With Contained AG:

```
Configure once
Replicate automatically
```

---

## Advantage 5: Application Portability

Applications see:

```
AG Listener
```

Not:

```
Specific SQL Instance
```

---

# 9. Disadvantages and Limitations

## 1. SQL Server Version Requirement

Requires:

```
SQL Server 2022+
```

Cannot enable on:

```
SQL Server 2019
SQL Server 2017
```

---

# 2. Migration Complexity

Existing AG:

```
Traditional AG
```

Conversion requires:

* Planning
* Testing
* Validation

---

# 3. Not Every Server Object Is Contained

Important:

Contained AG does NOT magically replicate everything.

Examples requiring attention:

* Operating system configuration
* Windows services
* File system folders
* External applications
* Linked server dependencies

---

# 4. Additional Management Complexity

DBAs must understand:

```
Instance scope

vs

Contained AG scope
```

---

# 5. Security Considerations

More objects move automatically.

Need governance around:

* Permissions
* Compliance
* Auditing

---

# 10. Real-Time Enterprise Scenarios

---

# Scenario 1: Banking Application

## Before

Architecture:

```
Primary DC

SQL01

BankDB


Secondary DC

SQL02

BankDB
```

Objects:

```
App Login
SQL Jobs
Encryption Keys
Credentials
```

Manual synchronization.

---

## After Contained AG

```
Contained AG

|
+ BankDB
|
+ App Login
|
+ Backup Jobs
|
+ Security Metadata
```

Failover:

```
SQL01 Failure

       |

SQL02

Everything available
```

---

# Scenario 2: Data Center Migration

Current:

```
SQL Server 2019

AG

SQL01
SQL02
```

Target:

```
SQL Server 2022

SQL03
SQL04
```

Contained AG:

```
Move:

Database

+
Metadata

+
Jobs

+
Security
```

Less migration effort.

---

# Scenario 3: Managed Service Provider

A hosting company manages:

```
100 SQL Instances
```

Each customer:

```
CustomerAG

Database
Users
Jobs
Security
```

Contained AG provides isolation.

---

# Scenario 4: Hybrid Cloud Disaster Recovery

Production:

```
On-Prem SQL Server

Contained AG

        |

Distributed AG

        |

Azure SQL VM
```

Benefits:

* Easier DR onboarding
* Reduced manual configuration

---

# 11. Contained AG vs Contained Database

Very important interview topic.

## Contained Database

Scope:

```
Single Database
```

Contains:

* Users
* Database settings

---

## Contained Availability Group

Scope:

```
Entire Availability Group
```

Contains:

* Databases
* Logins
* Jobs
* Metadata
* Configuration

Comparison:

| Feature         | Contained DB | Contained AG |
| --------------- | ------------ | ------------ |
| Database users  | Yes          | Yes          |
| Server logins   | No           | Yes          |
| SQL Agent Jobs  | No           | Yes          |
| AG support      | No           | Yes          |
| SQL Server 2022 | Yes          | Yes          |

---

# 12. Monitoring Contained AG

Check AG:

```sql
SELECT
name,
contained
FROM sys.availability_groups;
```

Check replicas:

```sql
SELECT *
FROM sys.availability_replicas;
```

Check databases:

```sql
SELECT *
FROM sys.dm_hadr_database_replica_states;
```

---

# 13. When Should You Use Contained AG?

Recommended:

✅ SQL Server 2022+
✅ Large enterprise environments
✅ Frequent migrations
✅ Hybrid cloud deployments
✅ Multi-site DR
✅ Applications with many dependencies

---

# 14. When Should You Avoid It?

Avoid when:

❌ SQL Server versions older than 2022
❌ Simple two-node HA
❌ Small databases with no metadata complexity
❌ Team lacks AG operational maturity

---

# 15. Senior DBA Interview Questions

### Q1:

What problem does Contained AG solve?

Answer:

> It extends the Availability Group boundary beyond databases by including instance-level metadata and objects, reducing dependency on individual SQL Server instances.

---

### Q2:

Is Contained AG the same as Contained Database?

Answer:

> No. Contained Database contains database-level objects. Contained AG contains Availability Group-related metadata and instance-level objects.

---

### Q3:

Which SQL Server version introduced Contained AG?

Answer:

> SQL Server 2022.

---

### Q4:

Does Contained AG eliminate all migration tasks?

Answer:

> No. It reduces synchronization requirements but does not migrate operating system, external applications, or every SQL Server instance dependency.

---

# Summary

```
Traditional AG:

Database Protection


Contained AG:

Database Protection
+
Metadata Protection
+
Improved Portability
+
Simplified Failover
```

For a Senior SQL Server DBA, Contained AG is mainly a **migration, hybrid cloud, and operational simplification feature introduced in SQL Server 2022**, not a replacement for traditional Always On AG.
