# SQL Server CI/CD Pipeline with GitHub Actions

## What This Project Does

This project automates SQL Server database deployments. When you push SQL scripts to GitHub, they automatically execute on your SQL Server without manual intervention.

## How It Works

1. You write SQL scripts and push them to GitHub
2. GitHub detects the push to the main branch
3. GitHub Actions workflow triggers automatically
4. Your self-hosted runner (Windows machine) receives the job
5. SQL scripts execute on your SQL Server database
6. Results appear in GitHub Actions logs

## Prerequisites

Before starting, install these tools on your Windows machine:

- Windows 10 Pro or Windows Server 2019 or later
- Git (https://git-scm.com/download/win)
- SQL Server 2019 or later
- SQL Server Management Studio (SSMS)
- PowerShell 5.0 or later (built into Windows)
- GitHub account (free tier works)

## Step 1: Create SQL Server Login for Deployments

Open SQL Server Management Studio and run this script:

```sql
CREATE LOGIN GitHub_CICD_User WITH PASSWORD = 'YourStrongPassword123!';
GO

USE YourDatabaseName;
GO

CREATE USER GitHub_CICD_User FOR LOGIN GitHub_CICD_User;
GO

ALTER ROLE db_ddladmin ADD MEMBER GitHub_CICD_User;
GO

PRINT 'GitHub CI/CD user created successfully';
GO
```

Replace 'YourStrongPassword123!' with a strong password. Write it down, you will need it later.

Verify the login works:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U GitHub_CICD_User -P YourStrongPassword123! -Q "SELECT @@VERSION;"
```

## Step 2: Create Project Folder Structure

Create these folders on your Windows machine:

```
F:\CICDDEMO
├── Scripts
│   └── Database
├── .github
│   └── workflows
└── README.md
```

Use PowerShell to create folders:

```powershell
mkdir F:\CICDDEMO
cd F:\CICDDEMO
mkdir Scripts\Database
mkdir .github\workflows
```

## Step 3: Create Your First SQL Script

Create file: `Scripts\Database\001_CreateTables.sql`

Add this sample SQL:

```sql
-- Script: 001_CreateTables.sql
-- Purpose: Create initial Users table
-- Date: 2024-01-15

IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'Users')
BEGIN
    CREATE TABLE dbo.Users
    (
        UserID INT PRIMARY KEY IDENTITY(1,1),
        UserName NVARCHAR(100) NOT NULL,
        Email NVARCHAR(255) NOT NULL,
        CreatedDate DATETIME DEFAULT GETDATE()
    );
    PRINT 'Table Users created successfully';
END
ELSE
BEGIN
    PRINT 'Table Users already exists';
END
GO

INSERT INTO dbo.Users (UserName, Email)
VALUES ('TestUser', 'test@example.com');
GO

SELECT * FROM dbo.Users;
GO
```

Key points for your SQL scripts:

- Use `IF NOT EXISTS` checks so scripts run multiple times safely
- Prefix scripts with numbers (001_, 002_, etc.) for execution order
- Use `GO` statements between batches
- Include comments explaining what the script does
- Test scripts manually in SSMS before pushing

## Step 4: Initialize Git Repository

Open PowerShell in your project folder:

```powershell
cd F:\CICDDEMO

git init

git config user.name "Your Name"
git config user.email "your.email@example.com"

git add .

git commit -m "Initial commit with SQL scripts"
```

## Step 5: Create GitHub Repository

Go to https://github.com and sign in.

Click the + icon (top right) and select "New repository".

Configure these settings:

- Repository name: SQL-CICD-DEMO
- Description: Automated SQL Server deployment pipeline
- Visibility: Private
- Click "Create repository"

GitHub shows setup instructions. Copy the repository URL.

## Step 6: Connect Local Folder to GitHub

In PowerShell, add the remote repository:

```powershell
git remote add origin https://github.com/YourUsername/SQL-CICD-DEMO.git

git branch -M main

git push -u origin main
```

Replace YourUsername with your GitHub username.

## Step 7: Create GitHub Actions Workflow

Create file: `.github\workflows\deploy-sql.yml`

Add this content:

```yaml
name: Deploy SQL Scripts

on:
  push:
    branches:
      - main
    paths:
      - 'Scripts/**'
  workflow_dispatch:

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Test SQL Connection
        shell: powershell
        run: |
          sqlcmd -S "${{ secrets.SQL_SERVER_HOST }}" `
            -d "${{ secrets.SQL_DATABASE }}" `
            -U "${{ secrets.SQL_USERNAME }}" `
            -P "${{ secrets.SQL_PASSWORD }}" `
            -C -Q "SELECT @@VERSION;"

      - name: Execute SQL Scripts
        shell: powershell
        run: |
          $scripts = Get-ChildItem -Path '.\Scripts\Database' -Filter '*.sql' | Sort-Object Name
          foreach ($script in $scripts) {
            Write-Host "Executing: $($script.Name)"
            sqlcmd -S "${{ secrets.SQL_SERVER_HOST }}" `
              -d "${{ secrets.SQL_DATABASE }}" `
              -U "${{ secrets.SQL_USERNAME }}" `
              -P "${{ secrets.SQL_PASSWORD }}" `
              -C -i "$($script.FullName)"
            if ($LASTEXITCODE -ne 0) {
              Write-Error "Script failed: $($script.Name)"
              exit 1
            }
          }
```

Commit and push this file:

```powershell
git add .github/workflows/deploy-sql.yml

git commit -m "Add GitHub Actions workflow"

git push origin main
```

## Step 8: Add GitHub Repository Secrets

GitHub Secrets store your SQL Server credentials securely. GitHub automatically masks these values in logs.

Go to your repository on GitHub:

https://github.com/YourUsername/SQL-CICD-DEMO

Click the "Settings" tab.

In the left sidebar, click "Secrets and variables" then "Actions".

Click "New repository secret" and add these four secrets:

**Secret 1: SQL Server Host**

- Name: SQL_SERVER_HOST
- Value: localhost (or your SQL Server name)

**Secret 2: Database Name**

- Name: SQL_DATABASE
- Value: YourDatabaseName

**Secret 3: SQL Username**

- Name: SQL_USERNAME
- Value: GitHub_CICD_User

**Secret 4: SQL Password**

- Name: SQL_PASSWORD
- Value: YourStrongPassword123!

After adding all four secrets, verify they appear in the Secrets list. Click each to confirm the name is correct.

## Step 9: Download and Install GitHub Self-Hosted Runner

GitHub Actions runs workflows on runners. A self-hosted runner is your Windows machine.

Go to your repository settings:

https://github.com/YourUsername/SQL-CICD-DEMO/settings/actions/runners

Click "New self-hosted runner".

Select:
- Operating System: Windows
- Architecture: x64

Download the ZIP file.

Extract the ZIP file to:

```
C:\GitHub-Runner
```

## Step 10: Configure and Start the Runner

Open PowerShell as Administrator:

```powershell
cd C:\GitHub-Runner
```

Run the configuration script:

```powershell
.\config.cmd
```

When prompted, enter:

- GitHub URL: https://github.com
- Token: (paste the token from the GitHub settings page)
- Runner name: sql-server-runner
- Work folder: (press Enter to accept default)

Configuration completes. Now start the runner:

```powershell
.\run.cmd
```

Wait for this message:

```
Listening for Jobs
```

Keep this PowerShell window open. The runner is now active and listening for deployment jobs.

## Step 11: Test the Pipeline

Create a new SQL script:

```powershell
cd F:\CICDDEMO
notepad Scripts\Database\002_CreateOrders.sql
```

Add this SQL:

```sql
IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'Orders')
BEGIN
    CREATE TABLE dbo.Orders
    (
        OrderID INT PRIMARY KEY IDENTITY(1,1),
        UserID INT NOT NULL,
        OrderDate DATETIME DEFAULT GETDATE()
    );
    PRINT 'Table Orders created successfully';
END
ELSE
BEGIN
    PRINT 'Table Orders already exists';
END
GO
```

Commit and push:

```powershell
git add Scripts\Database\002_CreateOrders.sql

git commit -m "Add Orders table"

git push origin main
```

Go to GitHub Actions:

https://github.com/YourUsername/SQL-CICD-DEMO/actions

Watch the workflow run. You should see:

- Checkout code (green checkmark)
- Test SQL Connection (green checkmark)
- Execute SQL Scripts (green checkmark)

In your PowerShell runner window, you will see:

```
Running job: deploy
Executing: 001_CreateTables.sql
Executing: 002_CreateOrders.sql
Job deploy completed with result: Succeeded
```

Open SQL Server Management Studio and verify both tables exist in your database.

## Step 12: Daily Usage

To add new SQL scripts:

1. Create a new file in Scripts\Database with a version prefix

```powershell
notepad Scripts\Database\003_AddStoredProcedures.sql
```

2. Write your SQL code

```sql
IF NOT EXISTS (SELECT * FROM sys.objects WHERE type = 'P' AND name = 'sp_GetUsers')
BEGIN
    EXEC sp_executesql N'CREATE PROCEDURE sp_GetUsers
    AS
    BEGIN
        SELECT * FROM dbo.Users
    END'
    PRINT 'Stored procedure sp_GetUsers created'
END
ELSE
BEGIN
    PRINT 'Stored procedure sp_GetUsers already exists'
END
GO
```

3. Test the script manually in SSMS

4. Commit and push

```powershell
git add Scripts\Database\003_AddStoredProcedures.sql

git commit -m "Add stored procedures"

git push origin main
```

5. GitHub Actions automatically deploys the script

## Folder Organization

Keep your project folder simple and organized:

```
SQL-CICD-DEMO
├── Scripts
│   └── Database
│       ├── 001_CreateTables.sql
│       ├── 002_CreateOrders.sql
│       └── 003_AddStoredProcedures.sql
├── .github
│   └── workflows
│       └── deploy-sql.yml
└── README.md
```

Scripts execute in alphabetical order. Use numbered prefixes to control execution order.

## Script Best Practices

Write scripts that are idempotent (safe to run multiple times):

```sql
-- Good: Uses IF NOT EXISTS
IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'Users')
BEGIN
    CREATE TABLE Users (UserID INT PRIMARY KEY)
END
GO

-- Bad: Fails on second run
CREATE TABLE Users (UserID INT PRIMARY KEY)
GO
```

Include comments explaining the purpose:

```sql
-- Script: 001_CreateTables.sql
-- Purpose: Create base tables for user management
-- Author: Your Name
-- Date: 2024-01-15
```

Use GO statements to separate batches:

```sql
CREATE TABLE Users (UserID INT PRIMARY KEY)
GO

INSERT INTO Users VALUES (1)
GO

SELECT * FROM Users
GO
```

Always test scripts manually before pushing to GitHub:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U GitHub_CICD_User -P YourPassword -i Scripts\Database\001_CreateTables.sql
```

## Monitoring Deployments

Go to your repository on GitHub:

https://github.com/YourUsername/SQL-CICD-DEMO

Click the "Actions" tab.

You see all workflow runs with timestamps and status:

- Green checkmark: Deployment succeeded
- Red X: Deployment failed
- Yellow circle: Deployment in progress

Click a workflow run to see details:

- Checkout code
- Test SQL Connection
- Execute SQL Scripts

Click "Execute SQL Scripts" to see which scripts ran and any SQL output.

## Troubleshooting

### Runner Shows Offline

The self-hosted runner is not running on your Windows machine.

Solution: Open PowerShell and start it:

```powershell
cd C:\GitHub-Runner
.\run.cmd
```

Wait for "Listening for Jobs". Keep this window open.

### SQL Connection Failed

The workflow cannot connect to your SQL Server.

Solutions:

1. Test the connection manually:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U GitHub_CICD_User -P YourPassword -Q "SELECT @@VERSION;"
```

2. Check GitHub secrets:

Go to Settings > Secrets > Actions. Verify all four secrets exist and spellings are correct.

3. Check firewall:

Windows Firewall might block SQL connections. Open Windows Defender Firewall and allow SQL Server.

4. Verify SQL Server is running:

Open SQL Server Management Studio and connect to your server.

### Permission Denied Error

The GitHub_CICD_User account lacks required permissions.

Solution: Grant proper permissions:

```sql
USE YourDatabaseName
GO

ALTER ROLE db_ddladmin ADD MEMBER GitHub_CICD_User
GO
```

### Script Execution Error

A SQL script contains errors or references missing objects.

Solutions:

1. Download the workflow log:

Go to Actions > Click the failed workflow > Click "Execute SQL Scripts" > Scroll down and click the download icon.

2. Review the SQL error message in the log.

3. Test the script manually in SSMS:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U GitHub_CICD_User -P YourPassword -i Scripts\Database\001_CreateTables.sql
```

4. Fix the script and push again.

### No Runners Found

GitHub Actions shows "No runners found" error.

Solutions:

1. Check if the runner PowerShell window is still open.

2. Restart the runner:

```powershell
cd C:\GitHub-Runner
.\run.cmd
```

3. Check Windows Event Viewer for errors.

4. Verify internet connection on your Windows machine.

## Common Git Commands

Check status of your files:

```powershell
git status
```

Stage changes for commit:

```powershell
git add .
```

Commit changes:

```powershell
git commit -m "Your message describing what changed"
```

Push to GitHub:

```powershell
git push origin main
```

Pull latest from GitHub:

```powershell
git pull origin main
```

View commit history:

```powershell
git log
```

Create a new branch:

```powershell
git checkout -b feature-branch
```

Switch branches:

```powershell
git checkout main
```

Merge a branch:

```powershell
git merge feature-branch
```

## Security Considerations

Never commit passwords or credentials to Git. GitHub automatically scans for exposed credentials and disables them.

Use GitHub Secrets for all sensitive information. Secrets are masked in logs and cannot be viewed after creation.

Create a dedicated SQL login (GitHub_CICD_User) for deployments. Do not reuse production database admin accounts.

Use strong passwords. Generate random passwords at least 15 characters long with uppercase, lowercase, numbers, and symbols.

Rotate passwords every 90 days.

Restrict SQL login permissions to db_ddladmin role minimum. Do not grant sysadmin role.

Monitor GitHub Actions logs regularly for failed deployments or suspicious activity.

## Performance Tips

Keep SQL scripts small and focused on one change per script.

Test scripts locally before pushing to GitHub.

Use indexes on large tables to improve query performance.

Batch insert operations when importing large datasets.

Monitor execution time in GitHub Actions logs.

Clean up old SQL scripts periodically.

## Architecture

Component Breakdown

Visual Studio or VS Code: Edit SQL scripts and manage local files.

Git: Tracks file changes, manages commits, maintains version history.

GitHub: Cloud repository stores your code, provides backup, triggers workflows.

GitHub Actions: Automation platform executes workflows when you push code.

Self-Hosted Runner: Your Windows machine executes jobs, has direct access to SQL Server.

SQL Server: Database server receives and executes SQL scripts.

Data Flow

1. You create or modify SQL scripts locally
2. You commit and push changes to GitHub
3. GitHub detects push to main branch
4. GitHub Actions workflow triggers automatically
5. Job is sent to self-hosted runner on your Windows machine
6. Runner downloads latest code from GitHub
7. PowerShell executes SQL scripts using sqlcmd utility
8. sqlcmd connects to SQL Server using stored credentials
9. SQL scripts execute on your database
10. Results and logs return to GitHub Actions
11. You review results in GitHub Actions tab

## Next Steps

1. Complete all setup steps above
2. Create your first SQL script and test
3. Monitor GitHub Actions logs after each push
4. Add more SQL scripts as your project grows
5. Set up email notifications in GitHub (optional)
6. Implement approval gates for production deployments (advanced)

## Support

For GitHub Actions documentation: https://docs.github.com/en/actions

For SQL Server documentation: https://learn.microsoft.com/en-us/sql/

For Git documentation: https://git-scm.com/doc

For GitHub self-hosted runners: https://docs.github.com/en/actions/hosting-your-own-runners

## Questions or Issues

1. Review the Troubleshooting section above
2. Check GitHub Actions logs for detailed error messages
3. Test SQL connections manually using sqlcmd
4. Verify all GitHub secrets are correctly spelled
5. Ensure runner is online and listening for jobs

---

Maintained By: SQL-CICD-DEMO Project Team
