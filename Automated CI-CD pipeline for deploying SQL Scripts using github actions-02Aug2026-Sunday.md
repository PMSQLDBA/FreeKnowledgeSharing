# SQL-CICD-DEMO

Automated CI/CD pipeline for deploying SQL Server scripts using GitHub Actions.

## What This Project Does

This project automates SQL Server database deployments. 
When you push SQL scripts to GitHub, they automatically execute on your SQL Server without manual intervention.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Quick Start](#quick-start)
4. [Installation Steps](#installation-steps)
5. [Configuration](#configuration)
6. [Usage](#usage)
7. [Folder Organization](#folder-organization)
8. [Troubleshooting](#troubleshooting)
9. [Architecture](#architecture)

## Prerequisites

Before you begin, ensure you have installed:

- Git (https://git-scm.com/download/win)
- Visual Studio Community (https://visualstudio.microsoft.com/downloads/)
- SQL Server (2019 or later)
- SQL Server Management Studio (SSMS)
- PowerShell 5.0 or later (usually pre-installed on Windows)

## Project Structure

```
SQL-CICD-DEMO
│
├── Scripts
│   └── Database
│       ├── 001_CreateTables.sql
│       ├── 002_CreateStoredProcedures.sql
│       └── [Your SQL scripts here]
│
├── .github
│   └── workflows
│       └── deploy-sql.yml
│
├── README.md
└── [Other project files]
```

### Folder Descriptions

**Scripts/Database/**
- Contains all SQL scripts for deployment
- Scripts execute in alphabetical order (001_, 002_, etc.)
- Each script should be idempotent (safe to run multiple times)

**.github/workflows/**
- Contains GitHub Actions workflow configuration
- deploy-sql.yml defines what happens when you push code
- This file tells GitHub to execute SQL scripts on your SQL Server

## Quick Start

For experienced developers who want to get started immediately:

```bash
# 1. Clone this repository
git clone https://github.com/YourUsername/SQL-CICD-DEMO.git
cd SQL-CICD-DEMO

# 2. Add SQL scripts to Scripts/Database folder
# (Use version prefix: 001_, 002_, etc.)

# 3. Configure GitHub Secrets (see Configuration section)

# 4. Install and run GitHub Self-Hosted Runner
# (See Installation Steps section)

# 5. Push changes to main branch
git add .
git commit -m "Add SQL scripts"
git push origin main

# 6. Watch GitHub Actions automatically deploy your scripts
```

## Installation Steps

### Step 1: Create Local Project Folder

Create a folder on your computer:
```
C:\Users\YourName\Documents\GitHub\SQL-CICD-DEMO
```

### Step 2: Create Required Subfolders

Inside SQL-CICD-DEMO, create these folders:

```
Scripts/
  └── Database/

.github/
  └── workflows/
```

### Step 3: Create Your First SQL Script

In `Scripts/Database/`, create file: `001_CreateTables.sql`

Add this sample SQL:
```sql
CREATE TABLE dbo.Users
(
    UserID INT PRIMARY KEY IDENTITY(1,1),
    UserName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME DEFAULT GETDATE()
);
GO

INSERT INTO dbo.Users (UserName, Email)
VALUES ('TestUser', 'test@example.com');
GO

SELECT * FROM dbo.Users;
GO
```

### Step 4: Initialize Git Repository

Open PowerShell in your project folder:

```powershell
# Initialize Git
git init

# Configure your identity
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Add all files
git add .

# Create first commit
git commit -m "Initial commit with SQL scripts"
```

### Step 5: Create GitHub Repository

1. Go to https://github.com
2. Click + icon (top right) > New repository
3. Repository name: `SQL-CICD-DEMO`
4. Description: `Automated SQL Server deployment pipeline`
5. Visibility: Private (only you can see)
6. Click Create repository

### Step 6: Connect Local to GitHub

```powershell
# Add remote repository (replace YourUsername)
git remote add origin https://github.com/YourUsername/SQL-CICD-DEMO.git

# Set default branch
git branch -M main

# Push to GitHub
git push -u origin main
```

### Step 7: Create GitHub Workflow File

In `.github/workflows/`, create file: `deploy-sql.yml`

Add this content:
```yaml
name: Deploy SQL Scripts

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

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
              -i "$($script.FullName)"
            if ($LASTEXITCODE -ne 0) {
              Write-Error "Script failed: $($script.Name)"
              exit 1
            }
          }
```

Push this file to GitHub:
```powershell
git add .
git commit -m "Add GitHub Actions workflow"
git push origin main
```

## Configuration

### Step 1: Create SQL Server Login

Open SQL Server Management Studio:

1. Expand Security > Logins
2. Right-click > New Login
3. Login name: `GitHub_CICD_User`
4. Select SQL Server authentication
5. Set strong password
6. Click OK

Grant database permissions:

1. Expand Databases > YourDatabase > Security > Users
2. Right-click > New User
3. User name: `GitHub_CICD_User`
4. Select login from dropdown: `GitHub_CICD_User`
5. Click Database role membership
6. Check `db_ddladmin` (for DDL operations)
7. Click OK

### Step 2: Add GitHub Secrets

GitHub Secrets store your SQL Server credentials securely.

1. Go to: https://github.com/YourUsername/SQL-CICD-DEMO
2. Click Settings tab
3. Left sidebar: Secrets and variables > Actions
4. Click New repository secret

Add these four secrets:

**Secret 1: SQL Server Host**
- Name: `SQL_SERVER_HOST`
- Value: `localhost` (or your SQL Server name)

**Secret 2: Database Name**
- Name: `SQL_DATABASE`
- Value: `YourDatabaseName`

**Secret 3: SQL Username**
- Name: `SQL_USERNAME`
- Value: `GitHub_CICD_User`

**Secret 4: SQL Password**
- Name: `SQL_PASSWORD`
- Value: `Your SQL login password`

### Step 3: Install GitHub Self-Hosted Runner

Download the runner:

1. Go to: https://github.com/YourUsername/SQL-CICD-DEMO/settings/actions/runners
2. Click New self-hosted runner
3. Select Windows and x64
4. Click Download
5. Save to: `C:\GitHub-Runner`

Extract and configure:

```powershell
# Extract files
cd C:\GitHub-Runner
# (Extract zip file here)

# Configure
.\config.cmd

# When prompted:
# Runner name: sql-server-runner
# Labels: (leave blank)
# Work directory: (accept default)
```

Run the runner:

```powershell
cd C:\GitHub-Runner
.\run.cmd
```

Wait for message: `Listening for Jobs`

Keep this window open. The runner is now active.

## Usage

### Adding a New SQL Script

1. Create file in `Scripts/Database/` with version prefix
   - Example: `003_CreateIndexes.sql`

2. Write your SQL code

3. Test manually in SQL Server Management Studio

4. Commit and push:
   ```powershell
   git add .
   git commit -m "Add new index script"
   git push origin main
   ```

5. GitHub Actions automatically executes your script

### Monitoring Deployments

1. Go to: https://github.com/YourUsername/SQL-CICD-DEMO
2. Click Actions tab
3. Watch workflow execution
4. Green checkmark = Success
5. Red X = Failed (click to see logs)

### Checking Execution Logs

After deployment completes:

1. Go to Actions tab
2. Click the workflow run
3. Click the job name
4. Review output for each step

## Folder Organization

### Scripts/Database Folder

Store SQL scripts here with version prefixes:

```
001_CreateTables.sql          (runs first)
002_CreateStoredProcedures.sql (runs second)
003_InsertInitialData.sql     (runs third)
004_CreateIndexes.sql         (runs fourth)
...
```

Scripts execute in alphabetical order. Name them with leading numbers to control execution order.

### Script Best Practices

Each script should:

- Be idempotent (safe to run multiple times)
- Include error handling
- Use GO statements between batches
- Include comments explaining purpose

Example:
```sql
-- Script: 001_CreateTables.sql
-- Purpose: Create initial database schema
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
```

## Troubleshooting

### Workflow Does Not Run

**Symptoms:** You pushed code but workflow did not execute

**Solutions:**
- Check runner status: Green dot means online
- Verify you pushed to main branch
- Confirm workflow file is in .github/workflows/
- Wait 30 seconds and refresh Actions page

### SQL Connection Failed

**Symptoms:** Workflow fails with connection error

**Solutions:**
- Test manually: `sqlcmd -S localhost -d YourDB -U GitHub_CICD_User -P YourPassword`
- Verify secrets are correct (exact spelling)
- Check firewall allows SQL connections
- Ensure SQL Server is running

### Permission Denied Error

**Symptoms:** Workflow fails with "permission denied"

**Solutions:**
- Open SSMS
- Verify GitHub_CICD_User has db_ddladmin role
- Re-grant permissions if needed
- Restart SQL Server (sometimes helps)

### Runner Offline

**Symptoms:** Actions tab shows "No runners found" or red dot

**Solutions:**
- Check if .\\run.cmd PowerShell window is still open
- Restart runner: `cd C:\GitHub-Runner` then `.\\run.cmd`
- Check Windows Event Viewer for errors
- Verify internet connection

### Script Execution Error

**Symptoms:** Workflow runs but SQL script fails

**Solutions:**
- Download workflow logs from Actions
- Review SQL error messages
- Test script manually in SSMS
- Check for missing database objects
- Verify database exists and is online

## Architecture

### How It Works

```
Developer makes changes to SQL scripts
                    |
                    v
Developer commits and pushes to GitHub
                    |
                    v
GitHub detects push to main branch
                    |
                    v
GitHub Actions workflow triggered
                    |
                    v
GitHub sends job to self-hosted runner
                    |
                    v
Runner (your Windows machine) receives job
                    |
                    v
Runner downloads latest code from GitHub
                    |
                    v
Runner executes SQL scripts using sqlcmd
                    |
                    v
sqlcmd connects to SQL Server using secrets
                    |
                    v
SQL scripts execute on database
                    |
                    v
Results logged to GitHub Actions
                    |
                    v
Developer reviews logs in GitHub
```

### Component Breakdown

**Visual Studio**
- Where you edit SQL scripts
- Manages local file changes
- Provides terminal for git commands

**Git**
- Tracks file changes locally
- Stores version history
- Manages commits and branches

**GitHub**
- Cloud repository (backup)
- Stores your code permanently
- Triggers workflows automatically

**GitHub Actions**
- Automation platform
- Runs workflows on events (push, pull request, schedule)
- Executes jobs on runners

**Self-Hosted Runner**
- Your Windows machine
- Executes GitHub Actions workflows
- Has direct access to SQL Server

**SQL Server**
- Database server
- Receives and executes SQL scripts
- Returns results to runner

## Security Considerations

- Never commit passwords to Git
- Use GitHub Secrets for all credentials
- Use strong, unique passwords
- Create dedicated SQL login for deployments
- Restrict SQL login permissions (db_ddladmin minimum)
- Rotate passwords every 90 days
- Monitor GitHub Actions logs for suspicious activity

## Common Git Commands

```bash
# Check status
git status

# Stage changes
git add .

# Commit changes
git commit -m "Your message"

# Push to GitHub
git push origin main

# Pull latest from GitHub
git pull origin main

# View commit history
git log

# Create new branch
git checkout -b feature-branch

# Switch branches
git checkout main

# Merge branch
git merge feature-branch
```

## Performance Tips

- Keep SQL scripts small and focused
- Use indexes for large tables
- Batch operations when possible
- Test scripts locally before pushing
- Monitor execution time in logs
- Clean up old scripts periodically

## Support and Documentation

- GitHub Actions Docs: https://docs.github.com/en/actions
- SQL Server Docs: https://learn.microsoft.com/en-us/sql/
- Git Documentation: https://git-scm.com/doc
- GitHub Runners: https://docs.github.com/en/actions/hosting-your-own-runners

## Next Steps

1. Follow Installation Steps section completely
2. Create your first SQL script
3. Test deployment pipeline
4. Monitor GitHub Actions logs
5. Add more SQL scripts as needed
6. Set up notifications (optional)
7. Implement approval gates for production (optional)

## License

This project is public.

## Contact

For questions or issues, review the Troubleshooting section or check GitHub Actions logs for detailed error messages.

**Maintained By:** SQL-CICD-DEMO Project Team
