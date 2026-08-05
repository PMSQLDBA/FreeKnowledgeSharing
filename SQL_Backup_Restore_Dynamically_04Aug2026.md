## Dynamic Cross-Server SQL Server Backup & Restore Automation Stored Procedure

### Purpose

The stored procedure **`dbo.usp_Generate_CrossServer_BackupRestore`** is designed to automate and standardize **SQL Server database backup and restore operations across different SQL Server instances**.

[SP Script] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_Generate_CrossServer_BackupRestore)

It eliminates manual steps involved in:

* Taking database backups from Production
* Moving backup files between servers
* Restoring databases with different names
* Handling existing databases during refresh activities
* Resolving orphaned database users after migration
* Updating database compatibility levels after restore

This solution is useful for common DBA activities such as:

* Production → UAT refresh
* Production → QA refresh
* Disaster Recovery database synchronization
* Database migration between SQL Server instances
* Environment cloning
* Cloud/on-premises migration activities

---

# High-Level Workflow

```
                 SOURCE SQL SERVER
                       |
                       |
             1. Generate Backup Script
                       |
                       |
              Backup Database (.BAK)
                       |
                       |
        Shared Backup Location / UNC Path
                       |
                       |
             2. Restore on Destination
                       |
                       |
          +-----------------------------+
          | Post Restore Automation     |
          +-----------------------------+
          | Fix Orphan Users            |
          | Update Compatibility Level  |
          | Validate Database           |
          +-----------------------------+

                 DESTINATION SQL SERVER
```

---

# How the Solution Works

## Step 1: Execute the Stored Procedure

The DBA executes the procedure from the **DBAScripts database**.

Example:

```sql
EXEC dbo.usp_Generate_CrossServer_BackupRestore
```

The procedure does not immediately perform backup/restore.

Instead, it dynamically generates:

1. Backup T-SQL block for the source server
2. Restore T-SQL block for the destination server
3. Post-restore administration scripts

The generated scripts can then be:

* Executed manually
* Executed through SQL Server Management Studio
* Automated through PowerShell Invoke-Sqlcmd
* Executed through CMS (Central Management Server)
* Integrated into DBA automation pipelines

---

# Input Parameters Explanation

## 1. Source Server

### Parameter

```sql
@SourceServer
```

### Purpose

Defines the SQL Server instance where the database currently exists.

Example:

```sql
@SourceServer='PROD-SQL01'
```

Customer Scenario:

> Production SQL Server contains the live Sales database. We need to refresh the Stage environment.

---

# 2. Source Database List

### Parameter

```sql
@SourceDatabaseNames
```

Supports multiple databases.

Example:

```sql
@SourceDatabaseNames='SalesDB,HRDB,FinanceDB'
```

The procedure validates:

* Database exists
* Database name is correct
* Database is available for backup

---

# 3. Backup Location

### Parameter

```sql
@BackupPath
```

Example:

```sql
\\SQLBACKUP01\SQLBackups\
```

Recommended:

Use a UNC shared folder accessible by both SQL Servers.

Example architecture:

```
PROD-SQL01
     |
     |
     | BACKUP
     |
     V

\\SQLBACKUP01\SQLBackups

     |
     |
     | RESTORE
     |
     V

STAGE-SQL01
```

---

# 4. Destination Server

### Parameter

```sql
@DestinationServer
```

Example:

```sql
@DestinationServer='STAGE-SQL01'
```

Defines where the database will be restored.

---

# 5. Database Rename During Restore

### Parameter

```sql
@DestinationDBName
```

Purpose:

Allows database cloning with a different name.

Example:

Production:

```
SalesDB
```

Stage:

```
SalesDB_UAT
```

Execution:

```sql
@DestinationDBName='SalesDB_UAT'
```

Result:

```
Production Database
        |
        |
        Backup
        |
        |
        Restore
        |
        V

SalesDB_UAT
```

---

# 6. Overwrite Existing Database Option

### Parameter

```sql
@OverwriteExisting
```

Values:

| Value | Behavior                  |
| ----- | ------------------------- |
| 0     | Protect existing database |
| 1     | Replace existing database |

---

## Example

### Safe Refresh

```sql
@OverwriteExisting=0
```

If database exists:

```
Database already exists.
Restore stopped.
```

Useful for:

* Production migrations
* First-time deployments

---

### Forced Refresh

```sql
@OverwriteExisting=1
```

Generates:

```sql
WITH REPLACE
```

Useful for:

* Daily QA refresh
* Development refresh
* Testing environments

---

# 7. Automatic Orphan User Fix

## Parameter

```sql
@AutoFixOrphanUsers=1
```

Problem:

When restoring a database to another SQL Server:

```
Source Server

Login:
John


Database User:
John
```

After restore:

```
Destination Server

Login:
Missing

Database User:
John
```

Result:

```
Orphan User
```

Application connection fails.

---

## Solution

The procedure automatically generates:

```sql
ALTER USER [username]
WITH LOGIN=[username]
```

Result:

```
Database User
       |
       |
       Mapping Fixed
       |
       |
Server Login
```

---

# 8. Automatic Compatibility Upgrade

## Parameter

```sql
@SetTargetCompatibility=1
```

Purpose:

Automatically adjusts the database compatibility level based on the destination SQL Server version.

Example:

Source:

```
SQL Server 2017

Compatibility:
140
```

Destination:

```
SQL Server 2022

Compatibility:
160
```

The procedure updates:

```sql
ALTER DATABASE DatabaseName
SET COMPATIBILITY_LEVEL = 160
```

Benefits:

* Enables latest optimizer features
* Improves query performance
* Supports newer SQL Server capabilities

---

# Customer Scenario 1

# Production to Stage Database Refresh

## Requirement

Customer wants:

```
Production
---------
Server:
PROD-SQL01

Database:
SalesDB


Refresh To:

Stage
-----
Server:
STAGE-SQL01

Database:
SalesDB_STAGE
```

---

## Execution

```sql
EXEC dbo.usp_Generate_CrossServer_BackupRestore

@SourceServer='PROD-SQL01',

@SourceDatabaseNames='SalesDB',

@BackupPath='\\SQLBACKUP01\SQLBackups\',

@DestinationServer='STAGE-SQL01',

@DestinationDBName='SalesDB_STAGE',

@OverwriteExisting=1,

@AutoFixOrphanUsers=1,

@SetTargetCompatibility=1;
```

---

## Generated Activities

The procedure creates:

### Source Side

```sql
BACKUP DATABASE [SalesDB]
TO DISK='\\SQLBACKUP01\SQLBackups\SalesDB_20260804.bak'
WITH COMPRESSION
```

---

### Destination Side

```sql
RESTORE DATABASE [SalesDB_STAGE]

FROM DISK=
'\\SQLBACKUP01\SQLBackups\SalesDB_20260804.bak'

WITH MOVE,
REPLACE,
RECOVERY
```

---

### Post Restore

Automatically:

1. Fix users
2. Upgrade compatibility
3. Complete database readiness

---

# Customer Scenario 2

# Multiple Database DR Migration

Requirement:

Move:

```
HR_PROD
Finance_PROD
CRM_PROD
```

From:

```
Production SQL Server
```

To:

```
DR SQL Server
```

Without overwriting.

---

Execution:

```sql
EXEC dbo.usp_Generate_CrossServer_BackupRestore

@SourceServer='PROD-SQL01',

@SourceDatabaseNames=
'HR_PROD,Finance_PROD,CRM_PROD',

@BackupPath=
'\\SQLBACKUP01\SQLBackups\',

@DestinationServer='DR-SQL01',

@OverwriteExisting=0;
```

---

# Operational Benefits to Customer

| Manual DBA Activity     | Automated By SP         |
| ----------------------- | ----------------------- |
| Backup scripting        | Dynamic generation      |
| File naming             | Timestamp based         |
| Restore scripting       | Generated automatically |
| Database rename         | Parameter driven        |
| Replace handling        | Configurable            |
| Logical file mapping    | Dynamic                 |
| Orphan user fixing      | Automated               |
| Compatibility upgrade   | Automated               |
| Multi database handling | Supported               |

---

# Recommended Production Usage Model

## Option 1: DBA Controlled Execution

```
Execute SP
     |
Review Generated Scripts
     |
Approve
     |
Execute Backup
     |
Execute Restore
     |
Validate Database
```

Recommended for:

* Production migrations
* Major releases

---

## Option 2: Fully Automated Pipeline

Integration:

```
GitHub Actions
       |
Azure DevOps
       |
PowerShell
       |
Invoke-Sqlcmd
       |
Stored Procedure
       |
SQL Server Restore
```

Recommended for:

* Daily refresh
* Dev/Test environments
* CI/CD database deployments

---

# DBA Validation Checklist After Restore

## Database Status

```sql
SELECT name,state_desc
FROM sys.databases;
```

Expected:

```
ONLINE
```

---

## User Mapping

```sql
EXEC sp_change_users_login 'Report';
```

Expected:

```
No orphan users
```

---

## Compatibility

```sql
SELECT name, compatibility_level
FROM sys.databases;
```

---

## Application Connectivity Test

Validate:

* Application login
* Jobs
* Linked Servers
* SSIS packages
* Reports
* Stored procedures

---

# Final Customer Summary

> "This stored procedure provides a reusable automation framework for cross-server SQL Server database refresh and migration activities. 
It dynamically generates backup and restore commands based on source and destination environments, supports database renaming, protects existing databases through configurable overwrite controls, automatically repairs orphaned users, and aligns database compatibility settings with the target SQL Server version. 
This reduces manual DBA effort, minimizes migration errors, and provides a repeatable process for Production, QA, DR, and cloud migration scenarios."

[SP Script] (https://github.com/PMSQLDBA/FreeKnowledgeSharing/blob/main/usp_Generate_CrossServer_BackupRestore)
