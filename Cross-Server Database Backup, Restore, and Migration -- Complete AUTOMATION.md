### Cross-Server Database Backup, Restore, and Migration -- Complete AUTOMATION

#### 1. Purpose & Overview

This Standard Operating Procedure (SOP) defines the process for executing automated, single or bulk database migrations using the stored procedure `[dbo].[usp_Generate_CrossServer_BackupRestore_V1]`. 
This procedure automates source backups, dynamic target file path discovery, pre-flight safety validations, database restores with engine-level error trapping (such as disk volume limits), compatibility updates, orphaned user remediation, and HTML email auditing.

Sample output:

<img width="1585" height="498" alt="sample output" src="https://github.com/user-attachments/assets/78c32aad-d418-453b-aacc-df89c2541697" />

---

#### 2. Prerequisites

* **SQL Server Version:** SQL Server 2016 or higher.
* **Permissions:** `sysadmin` privileges (or equivalent `DbCreator` and `ProcessAdmin` roles) on both source and destination instances.
* **Linked Servers:** Required for cross-server migrations, with RPC and RPC Out enabled.
* **Database Mail:** Configured and active on the primary execution instance for automated reporting.
* **Shared Storage:** A network share or shared directory accessible by both source and destination servers for storing intermediate `.bak` files.

---

#### 3. Parameters Reference

| Parameter | Data Type | Default | Description |
| --- | --- | --- | --- |
| `@SourceServer` | `SYSNAME` | `NULL` | Source SQL Server instance name (defaults to local `@@SERVERNAME`). |
| `@SourceDatabaseNames` | `VARCHAR(MAX)` | **Required** | Comma-separated list of databases to migrate (e.g., `'alpha, advworks'`). |
| `@BackupPath` | `VARCHAR(260)` | **Required** | Shared network directory path for backup files (e.g., `'\\Server\BackupShare\'`). |
| `@DestinationServer` | `SYSNAME` | `NULL` | Target SQL Server instance name (defaults to local `@@SERVERNAME`). |
| `@DestinationDBName` | `SYSNAME` | `NULL` | Target database name override (only applies if a single source database is specified). |
| `@OverwriteExisting` | `BIT` | `0` | Controls behavior if target database exists (`0` = Skip & log warning, `1` = Drop & replace). |
| `@AutoFixOrphanUsers` | `BIT` | `1` | Automatically maps database users to existing server logins post-restore. |
| `@SetTargetCompatibility` | `BIT` | `1` | Aligns restored database compatibility level with the destination instance master database. |
| `@EmailProfile` | `SYSNAME` | `NULL` | Database Mail profile name for status notifications. |
| `@NotificationEmail` | `VARCHAR(500)` | `NULL` | Recipient email address(es) for the HTML migration audit summary report. |

---

#### 4. Execution Workflow Steps

1. **Preparation:** Verify network paths, permissions, and disk space availability on the destination volume.
2. **Procedure Execution:** Run the script using the template format below.
3. **Auditing & Verification:** Review the console output or the dispatched HTML email report confirming successful migration status, execution times, and error handling logs.

---

#### 5. Example Execution Template

```sql
--For Single DB:
EXEC [dbo].[usp_Generate_CrossServer_BackupRestore_V1]
     @SourceServer           = 'pmsqldba',
     @SourceDatabaseNames    = 'alpha',
     @BackupPath             = '\\Pmsqldba\testbkps\SHAREDBKPS\',
     @DestinationServer      = 'pmsqldba\chennai',
     @DestinationDBName      = 'alpha',
     @OverwriteExisting      = 1,
     @AutoFixOrphanUsers     = 1,
     @SetTargetCompatibility = 1,
     @EmailProfile           = 'DBAAlerts', -- Replace with your SQL Database Mail Profile name
     @NotificationEmail      = 'sqldbateam2026@gmail.com'; -- Replace with the recipient email address(es)


--For Multiple DBs:
EXEC [dbo].[usp_Generate_CrossServer_BackupRestore_V1]
     @SourceServer           = 'pmsqldba',
     @SourceDatabaseNames    = 'alpha,BETA, DBAScripts, Finance, Inventory',
     @BackupPath             = '\\Pmsqldba\testbkps\SHAREDBKPS\',
     @DestinationServer      = 'pmsqldba\chennai',
     @DestinationDBName      = 'alpha,BETA, DBAScripts, Finance, Inventory',
     @OverwriteExisting      = 1,
     @AutoFixOrphanUsers     = 1,
     @SetTargetCompatibility = 1,
     @EmailProfile           = 'DBAAlerts', -- Replace with your SQL Database Mail Profile name
     @NotificationEmail      = 'sqldbateam2026'; -- Replace with the recipient email address(es)

--Below is the main SP: 

```sql
USE DBAScripts
GO

CREATE OR ALTER PROCEDURE [dbo].[usp_Generate_CrossServer_BackupRestore_V1]
(
      @SourceServer         SYSNAME = NULL,
      @SourceDatabaseNames   VARCHAR(MAX),
      @BackupPath            VARCHAR(260),
      @DestinationServer     SYSNAME = NULL,
      @DestinationDBName     SYSNAME = NULL,
      @OverwriteExisting     BIT = 0,
      @AutoFixOrphanUsers    BIT = 1,
      @SetTargetCompatibility BIT = 1,
      @EmailProfile          SYSNAME = NULL,
      @NotificationEmail     VARCHAR(500) = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @ErrorMessage NVARCHAR(MAX) = NULL;

    BEGIN TRY
        IF @SourceServer IS NULL OR @SourceServer = ''
            SET @SourceServer = @@SERVERNAME;
        
        IF @DestinationServer IS NULL OR @DestinationServer = ''
            SET @DestinationServer = @@SERVERNAME;

        IF @DestinationDBName = '' 
            SET @DestinationDBName = NULL;

        DECLARE @timestamp CHAR(15) =
            CONVERT(CHAR(8), GETDATE(), 112) + '_' + REPLACE(CONVERT(CHAR(8), GETDATE(), 108), ':', '');

        CREATE TABLE #SelectedDBs (
            RowID INT IDENTITY(1,1),
            SourceDB SYSNAME,
            TargetDB SYSNAME
        );

        CREATE TABLE #MigrationAudit (
            SourceServer SYSNAME,
            SourceDB SYSNAME,
            BackupStartDate DATETIME,
            BackupFinishDate DATETIME,
            DestinationServer SYSNAME,
            DestinationDB SYSNAME,
            RestoreStartTime DATETIME,
            RestoreFinishDate DATETIME,
            TotalRestoreTimeSec INT,
            ExecutionStatus VARCHAR(20),
            ErrorDetails NVARCHAR(MAX)
        );

        ;WITH DBList AS
        (
            SELECT LTRIM(RTRIM(value)) AS DatabaseName
            FROM STRING_SPLIT(@SourceDatabaseNames, ',')
        ),
        ValidDBs AS
        (
            SELECT d.DatabaseName
            FROM DBList d
            INNER JOIN sys.databases sd ON sd.name = d.DatabaseName
        )
        INSERT INTO #SelectedDBs (SourceDB, TargetDB)
        SELECT 
            DatabaseName,
            CASE 
                WHEN @DestinationDBName IS NOT NULL AND (SELECT COUNT(*) FROM ValidDBs) = 1 THEN @DestinationDBName
                ELSE DatabaseName 
            END
        FROM ValidDBs;

        IF NOT EXISTS (SELECT 1 FROM #SelectedDBs)
        BEGIN
            RAISERROR('No valid source databases found matching the criteria.', 16, 1);
            RETURN;
        END

        DECLARE @CurrentRow INT = 1;
        DECLARE @TotalRows INT = (SELECT COUNT(*) FROM #SelectedDBs);
        DECLARE @SrcDB SYSNAME, @TgtDB SYSNAME;
        DECLARE @BackupFile VARCHAR(500);
        DECLARE @BackupCmd NVARCHAR(MAX);
        DECLARE @RestoreCmd NVARCHAR(MAX);
        DECLARE @MoveCmd NVARCHAR(MAX);
        DECLARE @IsCrossServer BIT = CASE WHEN @DestinationServer <> @@SERVERNAME THEN 1 ELSE 0 END;

        DECLARE @BackupStart DATETIME, @BackupFinish DATETIME;
        DECLARE @RestoreStart DATETIME, @RestoreFinish DATETIME;

        WHILE @CurrentRow <= @TotalRows
        BEGIN
            SELECT @SrcDB = SourceDB, @TgtDB = TargetDB 
            FROM #SelectedDBs 
            WHERE RowID = @CurrentRow;

            SET @BackupFile = @BackupPath + @SrcDB + '_' + @timestamp + '.bak';

            PRINT '======================================================================'
            PRINT 'Processing Database: ' + @SrcDB + ' to ' + @TgtDB
            PRINT '======================================================================'

            ------------------------------------------------------------------
            -- A. Execute Backup (On Source Server)
            ------------------------------------------------------------------
            PRINT 'Starting backup of database: ' + @SrcDB
            SET @BackupStart = GETDATE();

            SET @BackupCmd = 
                'BACKUP DATABASE [' + @SrcDB + '] ' +
                'TO DISK = ''' + @BackupFile + ''' ' +
                'WITH COMPRESSION, STATS = 10, INIT, NAME = ''Maint-CrossServerTransfer-' + @SrcDB + '''';
            
            EXEC sp_executesql @BackupCmd;
            SET @BackupFinish = GETDATE();

            PRINT 'Backup completed successfully for database: ' + @SrcDB

            ------------------------------------------------------------------
            -- B. Execute Restore (On Destination Server)
            ------------------------------------------------------------------
            PRINT 'Starting restore of database: ' + @TgtDB
            SET @RestoreStart = GETDATE();

            IF @OverwriteExisting = 0
            BEGIN
                DECLARE @DBExists INT = 0;
                
                IF @IsCrossServer = 1
                BEGIN
                    DECLARE @CheckDBSQL NVARCHAR(MAX) = N'SELECT @DBExists_Out = CASE WHEN DB_ID(@TargetDBName) IS NOT NULL THEN 1 ELSE 0 END;';
                    DECLARE @ParmDefinition NVARCHAR(MAX) = N'@TargetDBName SYSNAME, @DBExists_Out INT OUTPUT';
                    DECLARE @RemoteExecSQL NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@CheckDBSQL, '''', '''''') + ''', N''' + REPLACE(@ParmDefinition, '''', '''''') + ''', @TargetDBName = @p1, @DBExists_Out = @p2 OUTPUT;';
                    
                    EXEC sp_executesql @RemoteExecSQL, N'@p1 SYSNAME, @p2 INT OUTPUT', @p1 = @TgtDB, @p2 = @DBExists OUTPUT;
                END
                ELSE
                BEGIN
                    SET @DBExists = CASE WHEN DB_ID(@TgtDB) IS NOT NULL THEN 1 ELSE 0 END;
                END

                IF @DBExists = 1
                BEGIN
                    PRINT 'Warning: Database ' + @TgtDB + ' already exists on target. Skipping restore.'
                    
                    INSERT INTO #MigrationAudit (SourceServer, SourceDB, BackupStartDate, BackupFinishDate, DestinationServer, DestinationDB, RestoreStartTime, RestoreFinishDate, TotalRestoreTimeSec, ExecutionStatus, ErrorDetails)
                    VALUES (@SourceServer, @SrcDB, @BackupStart, @BackupFinish, @DestinationServer, @TgtDB, @RestoreStart, GETDATE(), 0, 'SKIPPED', 'Database already exists on target.');

                    SET @CurrentRow = @CurrentRow + 1;
                    CONTINUE;
                END
            END
            ELSE
            BEGIN
                DECLARE @DropSQL NVARCHAR(MAX) = 'IF DB_ID(''[' + @TgtDB + ']'') IS NOT NULL DROP DATABASE [' + @TgtDB + '];';
                IF @IsCrossServer = 1
                BEGIN
                    DECLARE @RemoteDropSQL NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@DropSQL, '''', '''''') + '''';
                    EXEC sp_executesql @RemoteDropSQL;
                END
                ELSE
                BEGIN
                    EXEC sp_executesql @DropSQL;
                END
            END

            DECLARE @TargetDefaultData NVARCHAR(512), @TargetDefaultLog NVARCHAR(512);

            IF @IsCrossServer = 1
            BEGIN
                DECLARE @PathQuery NVARCHAR(MAX) = N'
                    SELECT 
                        @DefaultData_Out = COALESCE(
                            CAST(SERVERPROPERTY(''InstanceDefaultDataPath'') AS NVARCHAR(512)),
                            (SELECT LEFT(physical_name, LEN(physical_name) - CHARINDEX(''\'', REVERSE(physical_name)) + 1) FROM sys.master_files WHERE database_id = 1 AND type = 0)
                        ),
                        @DefaultLog_Out = COALESCE(
                            CAST(SERVERPROPERTY(''InstanceDefaultLogPath'') AS NVARCHAR(512)),
                            (SELECT LEFT(physical_name, LEN(physical_name) - CHARINDEX(''\'', REVERSE(physical_name)) + 1) FROM sys.master_files WHERE database_id = 1 AND type = 1)
                        );
                ';
                DECLARE @PathParams NVARCHAR(MAX) = N'@DefaultData_Out NVARCHAR(512) OUTPUT, @DefaultLog_Out NVARCHAR(512) OUTPUT';
                DECLARE @RemotePathExecSQL NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@PathQuery, '''', '''''') + ''', N''' + REPLACE(@PathParams, '''', '''''') + ''', @DefaultData_Out = @p1 OUTPUT, @DefaultLog_Out = @p2 OUTPUT;';

                EXEC sp_executesql @RemotePathExecSQL, N'@p1 NVARCHAR(512) OUTPUT, @p2 NVARCHAR(512) OUTPUT', @p1 = @TargetDefaultData OUTPUT, @p2 = @TargetDefaultLog OUTPUT;
            END
            ELSE
            BEGIN
                SELECT 
                    @TargetDefaultData = COALESCE(
                        CAST(SERVERPROPERTY('InstanceDefaultDataPath') AS NVARCHAR(512)),
                        (SELECT LEFT(physical_name, LEN(physical_name) - CHARINDEX('\', REVERSE(physical_name)) + 1) FROM sys.master_files WHERE database_id = 1 AND type = 0)
                    ),
                    @TargetDefaultLog = COALESCE(
                        CAST(SERVERPROPERTY('InstanceDefaultLogPath') AS NVARCHAR(512)),
                        (SELECT LEFT(physical_name, LEN(physical_name) - CHARINDEX('\', REVERSE(physical_name)) + 1) FROM sys.master_files WHERE database_id = 1 AND type = 1)
                    );
            END

            IF @TargetDefaultLog IS NULL OR @TargetDefaultLog = ''
                SET @TargetDefaultLog = @TargetDefaultData;

            SET @MoveCmd = '';
            SELECT @MoveCmd = @MoveCmd + 
                CHAR(13) + ', MOVE ''' + mf.name + ''' TO ''' + 
                CASE mf.type 
                    WHEN 0 THEN @TargetDefaultData + @TgtDB + '_' + mf.name + '.mdf'
                    ELSE @TargetDefaultLog + @TgtDB + '_' + mf.name + '.ldf'
                END + '''' 
            FROM sys.master_files mf
            WHERE mf.database_id = DB_ID(@SrcDB)
            ORDER BY mf.type;

            SET @RestoreCmd = 
                'RESTORE DATABASE [' + @TgtDB + '] ' +
                'FROM DISK = ''' + @BackupFile + ''' ' +
                'WITH STATS = 10, REPLACE, RECOVERY' + 
                @MoveCmd;

            DECLARE @InnerErrorMsg NVARCHAR(MAX) = NULL;

            IF @IsCrossServer = 1
            BEGIN
                DECLARE @RemoteRestoreWrapper NVARCHAR(MAX) = N'
                    BEGIN TRY
                        exec(' + QUOTENAME(@RestoreCmd, '''') + ');
                    END TRY
                    BEGIN CATCH
                        SET @ErrorMsg_Out = ERROR_MESSAGE();
                    END CATCH
                ';
                
                DECLARE @ParamDef NVARCHAR(MAX) = N'@ErrorMsg_Out NVARCHAR(MAX) OUTPUT';
                DECLARE @RemoteExecWrapperSQL NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@RemoteRestoreWrapper, '''', '''''') + ''', N''' + REPLACE(@ParamDef, '''', '''''') + ''', @ErrorMsg_Out = @p1 OUTPUT;';
                
                EXEC sp_executesql @RemoteExecWrapperSQL, N'@p1 NVARCHAR(MAX) OUTPUT', @p1 = @InnerErrorMsg OUTPUT;
            END
            ELSE
            BEGIN
                BEGIN TRY
                    EXEC sp_executesql @RestoreCmd;
                END TRY
                BEGIN CATCH
                    SET @InnerErrorMsg = ERROR_MESSAGE();
                END CATCH
            END

            IF @InnerErrorMsg IS NOT NULL
            BEGIN
                SET @RestoreFinish = GETDATE();
                PRINT 'Restore failed: ' + @InnerErrorMsg;

                INSERT INTO #MigrationAudit (SourceServer, SourceDB, BackupStartDate, BackupFinishDate, DestinationServer, DestinationDB, RestoreStartTime, RestoreFinishDate, TotalRestoreTimeSec, ExecutionStatus, ErrorDetails)
                VALUES (@SourceServer, @SrcDB, @BackupStart, @BackupFinish, @DestinationServer, @TgtDB, @RestoreStart, @RestoreFinish, DATEDIFF(SECOND, @RestoreStart, @RestoreFinish), 'FAILED', @InnerErrorMsg);

                SET @CurrentRow = @CurrentRow + 1;
                CONTINUE;
            END

            SET @RestoreFinish = GETDATE();
            PRINT 'Restore completed successfully for database: ' + @TgtDB

            INSERT INTO #MigrationAudit (SourceServer, SourceDB, BackupStartDate, BackupFinishDate, DestinationServer, DestinationDB, RestoreStartTime, RestoreFinishDate, TotalRestoreTimeSec, ExecutionStatus, ErrorDetails)
            VALUES (@SourceServer, @SrcDB, @BackupStart, @BackupFinish, @DestinationServer, @TgtDB, @RestoreStart, @RestoreFinish, DATEDIFF(SECOND, @RestoreStart, @RestoreFinish), 'SUCCESS', 'None');

            ------------------------------------------------------------------
            -- C. Post-Restore Automation (Compatibility & Orphans)
            ------------------------------------------------------------------
            
            -- Auto Upgrade Compatibility Level
            IF @SetTargetCompatibility = 1
            BEGIN
                PRINT 'Updating compatibility level for database: ' + @TgtDB
                
                DECLARE @CompatSQL NVARCHAR(MAX) = N'
                    IF DB_ID(N''' + @TgtDB + ''') IS NOT NULL
                    BEGIN
                        DECLARE @CompatLvl INT;
                        SELECT @CompatLvl = compatibility_level FROM sys.databases WHERE name = ''master'';
                        DECLARE @AltSQL NVARCHAR(MAX) = ''ALTER DATABASE [' + @TgtDB + '] SET COMPATIBILITY_LEVEL = '' + CAST(@CompatLvl AS VARCHAR(5));
                        EXEC sp_executesql @AltSQL;
                    END
                ';
                
                IF @IsCrossServer = 1
                BEGIN
                    DECLARE @RemoteCompatExec NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@CompatSQL, '''', '''''') + ''';';
                    EXEC sp_executesql @RemoteCompatExec;
                END
                ELSE
                BEGIN
                    EXEC sp_executesql @CompatSQL;
                END
                
                PRINT 'Compatibility level check completed for: ' + @TgtDB
            END

            -- Automate fixing orphaned database users
            IF @AutoFixOrphanUsers = 1
            BEGIN
                PRINT 'Fixing orphaned users for database: ' + @TgtDB
                
                DECLARE @CheckOrphanSQL NVARCHAR(MAX) = '
                    IF DB_ID(N''' + @TgtDB + ''') IS NOT NULL
                    BEGIN
                        DECLARE @UserRepairSQL NVARCHAR(MAX) = '''';
                        SELECT @UserRepairSQL += ''ALTER USER ['' + dp.name + ''] WITH LOGIN = ['' + dp.name + '']; '' + CHAR(13)
                        FROM [' + @TgtDB + '].sys.database_principals dp
                        WHERE dp.type = ''S'' 
                        AND dp.authentication_type = 1
                        AND EXISTS (SELECT 1 FROM sys.server_principals sp WHERE sp.name = dp.name);
                        
                        IF @UserRepairSQL <> ''''
                        BEGIN
                            EXEC [' + @TgtDB + '].sys.sp_executesql @UserRepairSQL;
                        END
                    END
                ';
                
                IF @IsCrossServer = 1
                BEGIN
                    DECLARE @RemoteOrphanSQL NVARCHAR(MAX) = 'EXEC [' + @DestinationServer + '].master.dbo.sp_executesql N''' + REPLACE(@CheckOrphanSQL, '''', '''''') + '''';
                    EXEC sp_executesql @RemoteOrphanSQL;
                END
                ELSE
                BEGIN
                    EXEC sp_executesql @CheckOrphanSQL;
                END
                
                PRINT 'Orphaned users checked/remapped for: ' + @TgtDB
            END

            PRINT '======================================================================'
            PRINT 'Database ' + @SrcDB + ' backup and restore process completed'
            PRINT '======================================================================'
            PRINT ''

            SET @CurrentRow = @CurrentRow + 1;
        END

        -- Generate and Send HTML Email Notification
        IF @NotificationEmail IS NOT NULL AND @NotificationEmail <> '' AND EXISTS (SELECT 1 FROM #MigrationAudit)
        BEGIN
            DECLARE @EmailSubject VARCHAR(255) = 'Database Migration Report - ' + CAST(GETDATE() AS VARCHAR(20));
            DECLARE @HTMLBody NVARCHAR(MAX);

            SET @HTMLBody = N'
            <html>
            <head>
                <style>
                    body { font-family: Arial, sans-serif; font-size: 14px; color: #333; }
                    h3 { color: #0056b3; }
                    table { border-collapse: collapse; width: 100%; margin-top: 10px; }
                    th, td { border: 1px solid #dddddd; text-align: left; padding: 8px; }
                    th { background-color: #0056b3; color: white; }
                    tr:nth-child(even) { background-color: #f9f9f9; }
                    .status-success { color: green; font-weight: bold; }
                    .status-failed { color: red; font-weight: bold; }
                    .error-text { color: #d9534f; font-size: 12px; font-family: Consolas, monospace; }
                </style>
            </head>
            <body>
                <h3>Database Backup & Restore Execution Summary</h3>
                <p>The cross-server backup and restore task execution details are listed below:</p>
                <table>
                    <tr>
                        <th>Source Server</th>
                        <th>Source DB</th>
                        <th>Backup Start Date</th>
                        <th>Backup Finish Date</th>
                        <th>Destination Server</th>
                        <th>Destination DB</th>
                        <th>Restore Start Time</th>
                        <th>Restore Finish Time</th>
                        <th>Total Time</th>
                        <th>Status / Error Details</th>
                    </tr>';

            SELECT @HTMLBody += N'
                    <tr>
                        <td>' + ISNULL(SourceServer, '') + '</td>
                        <td>' + ISNULL(SourceDB, '') + '</td>
                        <td>' + ISNULL(CONVERT(VARCHAR(20), BackupStartDate, 120), '') + '</td>
                        <td>' + ISNULL(CONVERT(VARCHAR(20), BackupFinishDate, 120), '') + '</td>
                        <td>' + ISNULL(DestinationServer, '') + '</td>
                        <td>' + ISNULL(DestinationDB, '') + '</td>
                        <td>' + ISNULL(CONVERT(VARCHAR(20), RestoreStartTime, 120), '') + '</td>
                        <td>' + ISNULL(CONVERT(VARCHAR(20), RestoreFinishDate, 120), '') + '</td>
                        <td>' + CAST(TotalRestoreTimeSec / 60 AS VARCHAR(10)) + ' min ' + CAST(TotalRestoreTimeSec % 60 AS VARCHAR(10)) + ' sec</td>
                        <td><span class="' + CASE WHEN ExecutionStatus = 'SUCCESS' THEN 'status-success' ELSE 'status-failed' END + '">' + ExecutionStatus + '</span>' + 
                            CASE WHEN ErrorDetails <> 'None' THEN '<br/><span class="error-text"><b>Error:</b> ' + ErrorDetails + '</span>' ELSE '' END + '</td>
                    </tr>'
            FROM #MigrationAudit;

            SET @HTMLBody += N'
                </table>
                <br>
                <p>Regards,<br><b>DBA Automation Team</b></p>
            </body>
            </html>';

            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = @EmailProfile,
                @recipients = @NotificationEmail,
                @subject = @EmailSubject,
                @body = @HTMLBody,
                @body_format = 'HTML';
        END

        DROP TABLE #SelectedDBs;
        DROP TABLE #MigrationAudit;

    END TRY
    BEGIN CATCH
        SET @ErrorMessage = ERROR_MESSAGE();
        PRINT 'ERROR: ' + @ErrorMessage;
        
        IF OBJECT_ID('tempdb..#SelectedDBs') IS NOT NULL
            DROP TABLE #SelectedDBs;
        IF OBJECT_ID('tempdb..#MigrationAudit') IS NOT NULL
            DROP TABLE #MigrationAudit;
        
        RETURN;
    END CATCH
END
GO

```
