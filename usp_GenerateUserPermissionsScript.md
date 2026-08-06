# usp_GenerateUserPermissionsScript

A SQL Server stored procedure (compatible with SQL Server 2016–2025) that **generates a T-SQL script** to recreate database users, role memberships, and permissions across one or more databases. It doesn't change anything itself — it *prints* the DDL/DCL statements you'd need to run elsewhere (e.g. to migrate security to a new server, rebuild a DR instance, or document current access).

## What it does

For each database you pass in, it walks through four stages and `PRINT`s the corresponding script:

1. **User creation** — For every database principal (excluding fixed/system principals with `principal_id <= 4`) of type SQL user, Windows user, or Windows group, it prints an idempotent `CREATE USER` statement:
   - Wrapped in `IF NOT EXISTS (...)` so it's safe to re-run.
   - Uses `FOR LOGIN [...]` when the user is mapped to a server login, or `WITHOUT LOGIN` otherwise.
   - Includes `WITH DEFAULT_SCHEMA = [...]` when a default schema is set.

2. **Role memberships** — Prints `ALTER ROLE [role] ADD MEMBER [user];` for every user who belongs to a database role.

3. **Database-level grants** — Prints `GRANT`/`DENY`/`REVOKE` statements for permissions granted at the database scope (e.g. `CREATE TABLE`, `CONNECT`), including `WITH GRANT OPTION` where applicable.

4. **Object/column-level grants** — Prints `GRANT`/`DENY`/`REVOKE` statements for permissions on individual objects (tables, views, procedures, etc.) and, where relevant, specific columns.

Each section is preceded by `GO` and `USE [DatabaseName];` so the output is a ready-to-run script that switches context correctly between databases and statement batches.

## How to use it

### 1. Create the procedure
Run the script above once against your instance (it creates the proc in `master` — `CREATE PROCEDURE` runs in whatever database context is active, and the script starts with `USE [master];`).

### 2. Execute it
```sql
EXEC master.dbo.usp_GenerateUserPermissionsScript @DatabaseNames = 'SalesDB, HRDB, Reporting';
```
- `@DatabaseNames` is a **comma-separated list** of database names (whitespace around names is trimmed).
- Passing an empty or all-whitespace string raises an error (`RAISERROR`) and exits.

### 3. Capture the output
The procedure only `PRINT`s the script — it does not return a result set or write to a file. To use the generated script you need to capture the "Messages" output:
- **SSMS**: Set output mode to "Results to Text" (Ctrl+T) before running, then save/copy the Messages pane content. `PRINT` output can also be truncated in SSMS for very large scripts — check your `Tools > Options > Query Results > Text` output width settings if the output looks cut off.
- **sqlcmd**: Run via `sqlcmd -S server -d master -Q "EXEC dbo.usp_GenerateUserPermissionsScript @DatabaseNames='SalesDB'" -o output.sql`, which redirects `PRINT` output straight to a file.

### 4. Review and run the generated script
The captured output is itself a valid T-SQL script (starts with `GO`/`USE` blocks per database/section). Review it, then execute it against the target server to recreate the users, roles, and permissions.

## Notes & caveats
- **Logins are not scripted.** This only handles database-level users/permissions. If a user is `FOR LOGIN`, the corresponding server login must already exist (or be created separately) on the target server, and its SID should match for orphaned-user issues to be avoided.
- **Passwords are never included** — there's nothing to script for login credentials since only database users are covered.
- **Fixed/system principals are skipped** (`principal_id > 4` filters out `dbo`, `guest`, `INFORMATION_SCHEMA`, `sys`, etc.).
- **Only SQL/Windows users and Windows groups** are scripted (`type_desc IN ('SQL_USER','WINDOWS_USER','WINDOWS_GROUP')`) — contained users mapped via Azure AD or certificate/asymmetric-key users, for example, are excluded from the role/permission sections (though the user-creation section's `type IN ('S','U','G')` filter is similarly scoped).
- **Order matters**: run the four generated sections in order (users → roles → database grants → object grants), since roles/grants reference users that must already exist.
- **Idempotency**: only the `CREATE USER` statements are wrapped in existence checks; re-running the role/grant sections is generally safe since `ALTER ROLE ... ADD MEMBER` and `GRANT` are themselves idempotent in SQL Server, but duplicate `DENY`/`REVOKE` ordering nuances are worth a quick review before bulk-applying.

============================================================================================

USE [master];

GO

SET ANSI_NULLS ON;

GO

SET QUOTED_IDENTIFIER ON;

GO

IF OBJECT_ID('dbo.usp_GenerateUserPermissionsScript', 'P') IS NOT NULL

    DROP PROCEDURE dbo.usp_GenerateUserPermissionsScript;
GO

CREATE PROCEDURE dbo.usp_GenerateUserPermissionsScript

    @DatabaseNames NVARCHAR(MAX)
    
AS

BEGIN
    SET NOCOUNT ON;

    IF LTRIM(RTRIM(ISNULL(@DatabaseNames, ''))) = ''
    BEGIN
        RAISERROR('The @DatabaseNames parameter cannot be empty.', 16, 1);
        RETURN;
    END

    DECLARE @TargetDBs TABLE (DbName SYSNAME PRIMARY KEY);

    INSERT INTO @TargetDBs (DbName)
    SELECT LTRIM(RTRIM(value))
    FROM STRING_SPLIT(@DatabaseNames, ',')
    WHERE LTRIM(RTRIM(value)) <> '';

    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- CONSOLIDATED USER & PERMISSION SCRIPT GENERATOR (SQL Server 2016 - 2025)';
    PRINT '-- Target Databases: ' + @DatabaseNames;
    PRINT '----------------------------------------------------------------------------------';
    PRINT '';

    DECLARE @CurrentDB SYSNAME;
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @ScriptLine NVARCHAR(MAX);

    -- =========================================================================
    -- 1. GENERATE USER CREATION SCRIPTS
    -- =========================================================================
    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- 1. DATABASE USERS CREATION SCRIPTS';
    PRINT '----------------------------------------------------------------------------------';

    DECLARE db_cursor_users CURSOR LOCAL FAST_FORWARD FOR
        SELECT DbName FROM @TargetDBs ORDER BY DbName;

    OPEN db_cursor_users;
    FETCH NEXT FROM db_cursor_users INTO @CurrentDB;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'GO';
        PRINT 'USE [' + @CurrentDB + '];';
        PRINT 'GO';

        SET @SQL = N'
        USE [' + @CurrentDB + '];
        
        DECLARE script_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT 
            ''IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = '''''' + u.name + '''''')'' + CHAR(13) + CHAR(10) +
            ''    CREATE USER '' + QUOTENAME(u.name) + 
            CASE 
                WHEN u.type IN (''S'', ''U'', ''G'') AND l.name IS NOT NULL THEN '' FOR LOGIN '' + QUOTENAME(l.name)
                ELSE '' WITHOUT LOGIN''
            END +
            ISNULL('' WITH DEFAULT_SCHEMA = '' + QUOTENAME(u.default_schema_name), '''') + '';''
        FROM sys.database_principals u
        LEFT JOIN sys.server_principals l ON u.sid = l.sid
        WHERE u.principal_id > 4 
          AND u.type IN (''S'', ''U'', ''G'');';

        -- Use dynamic execution with an inner cursor to PRINT lines cleanly
        DECLARE @InnerExec NVARCHAR(MAX) = @SQL + N'
        DECLARE @Line NVARCHAR(MAX);
        OPEN script_cursor;
        FETCH NEXT FROM script_cursor INTO @Line;
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT @Line;
            FETCH NEXT FROM script_cursor INTO @Line;
        END
        CLOSE script_cursor;
        DEALLOCATE script_cursor;';

        EXEC sp_executesql @InnerExec;

        FETCH NEXT FROM db_cursor_users INTO @CurrentDB;
    END

    CLOSE db_cursor_users;
    DEALLOCATE db_cursor_users;

    -- =========================================================================
    -- 2. GENERATE ROLE MEMBERSHIPS
    -- =========================================================================
    PRINT '';
    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- 2. DATABASE ROLE MEMBERSHIPS (ALTER ROLE syntax)';
    PRINT '----------------------------------------------------------------------------------';

    DECLARE db_cursor_roles CURSOR LOCAL FAST_FORWARD FOR
        SELECT DbName FROM @TargetDBs ORDER BY DbName;

    OPEN db_cursor_roles;
    FETCH NEXT FROM db_cursor_roles INTO @CurrentDB;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'GO';
        PRINT 'USE [' + @CurrentDB + '];';
        PRINT 'GO';

        SET @SQL = N'
        USE [' + @CurrentDB + '];

        DECLARE script_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT 
            ''ALTER ROLE '' + QUOTENAME(USER_NAME(rm.role_principal_id)) + 
            '' ADD MEMBER '' + QUOTENAME(usr.name) + '';''
        FROM sys.database_role_members AS rm
        JOIN sys.database_principals AS usr ON rm.member_principal_id = usr.principal_id
        WHERE usr.principal_id > 4 
          AND usr.type_desc IN (''SQL_USER'', ''WINDOWS_USER'', ''WINDOWS_GROUP'')
        ORDER BY rm.role_principal_id ASC;';

        SET @InnerExec = @SQL + N'
        DECLARE @Line NVARCHAR(MAX);
        OPEN script_cursor;
        FETCH NEXT FROM script_cursor INTO @Line;
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT @Line;
            FETCH NEXT FROM script_cursor INTO @Line;
        END
        CLOSE script_cursor;
        DEALLOCATE script_cursor;';

        EXEC sp_executesql @InnerExec;

        FETCH NEXT FROM db_cursor_roles INTO @CurrentDB;
    END

    CLOSE db_cursor_roles;
    DEALLOCATE db_cursor_roles;

    -- =========================================================================
    -- 3. GENERATE DATABASE LEVEL GRANTS
    -- =========================================================================
    PRINT '';
    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- 3. DATABASE LEVEL PERMISSIONS (GRANTS)';
    PRINT '----------------------------------------------------------------------------------';

    DECLARE db_cursor_db_perms CURSOR LOCAL FAST_FORWARD FOR
        SELECT DbName FROM @TargetDBs ORDER BY DbName;

    OPEN db_cursor_db_perms;
    FETCH NEXT FROM db_cursor_db_perms INTO @CurrentDB;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'GO';
        PRINT 'USE [' + @CurrentDB + '];';
        PRINT 'GO';

        SET @SQL = N'
        USE [' + @CurrentDB + '];

        DECLARE script_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT 
            CASE WHEN perm.state <> ''W'' THEN perm.state_desc ELSE ''GRANT'' END + 
            '' '' + perm.permission_name + 
            '' TO '' + QUOTENAME(usr.name) COLLATE database_default + 
            CASE WHEN perm.state <> ''W'' THEN '''' ELSE '' WITH GRANT OPTION'' END + '';''
        FROM sys.database_permissions AS perm
        JOIN sys.database_principals AS usr ON perm.grantee_principal_id = usr.principal_id
        WHERE usr.principal_id > 4 
          AND usr.type_desc IN (''SQL_USER'', ''WINDOWS_USER'', ''WINDOWS_GROUP'')
          AND perm.class_desc = ''DATABASE''
        ORDER BY perm.permission_name ASC, perm.state_desc ASC;';

        SET @InnerExec = @SQL + N'
        DECLARE @Line NVARCHAR(MAX);
        OPEN script_cursor;
        FETCH NEXT FROM script_cursor INTO @Line;
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT @Line;
            FETCH NEXT FROM script_cursor INTO @Line;
        END
        CLOSE script_cursor;
        DEALLOCATE script_cursor;';

        EXEC sp_executesql @InnerExec;

        FETCH NEXT FROM db_cursor_db_perms INTO @CurrentDB;
    END

    CLOSE db_cursor_db_perms;
    DEALLOCATE db_cursor_db_perms;

    -- =========================================================================
    -- 4. GENERATE OBJECT OR COLUMN LEVEL GRANTS
    -- =========================================================================
    PRINT '';
    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- 4. OBJECT OR COLUMN LEVEL PERMISSIONS';
    PRINT '----------------------------------------------------------------------------------';

    DECLARE db_cursor_obj_perms CURSOR LOCAL FAST_FORWARD FOR
        SELECT DbName FROM @TargetDBs ORDER BY DbName;

    OPEN db_cursor_obj_perms;
    FETCH NEXT FROM db_cursor_obj_perms INTO @CurrentDB;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        PRINT 'GO';
        PRINT 'USE [' + @CurrentDB + '];';
        PRINT 'GO';

        SET @SQL = N'
        USE [' + @CurrentDB + '];

        DECLARE script_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT 
            CASE WHEN perm.state <> ''W'' THEN perm.state_desc ELSE ''GRANT'' END + 
            '' '' + perm.permission_name + 
            '' ON '' + QUOTENAME(SCHEMA_NAME(obj.schema_id)) + ''.'' + QUOTENAME(obj.name) + 
            ISNULL(''('' + QUOTENAME(cl.name) + '')'', '''') + 
            '' TO '' + QUOTENAME(usr.name) COLLATE database_default + 
            CASE WHEN perm.state <> ''W'' THEN '''' ELSE '' WITH GRANT OPTION'' END + '';''
        FROM sys.database_permissions AS perm
        JOIN sys.objects AS obj ON perm.major_id = obj.object_id
        JOIN sys.database_principals AS usr ON perm.grantee_principal_id = usr.principal_id
        LEFT JOIN sys.columns AS cl ON cl.column_id = perm.minor_id AND cl.object_id = perm.major_id
        WHERE usr.principal_id > 4 
          AND usr.type_desc IN (''SQL_USER'', ''WINDOWS_USER'', ''WINDOWS_GROUP'')
        ORDER BY perm.permission_name ASC, perm.state_desc ASC;';

        SET @InnerExec = @SQL + N'
        DECLARE @Line NVARCHAR(MAX);
        OPEN script_cursor;
        FETCH NEXT FROM script_cursor INTO @Line;
        WHILE @@FETCH_STATUS = 0
        BEGIN
            PRINT @Line;
            FETCH NEXT FROM script_cursor INTO @Line;
        END
        CLOSE script_cursor;
        DEALLOCATE script_cursor;';

        EXEC sp_executesql @InnerExec;

        FETCH NEXT FROM db_cursor_obj_perms INTO @CurrentDB;
    END

    CLOSE db_cursor_obj_perms;
    DEALLOCATE db_cursor_obj_perms;

    PRINT '';
    PRINT '----------------------------------------------------------------------------------';
    PRINT '-- SCRIPT GENERATION COMPLETE';
    PRINT '----------------------------------------------------------------------------------';
END;
GO
