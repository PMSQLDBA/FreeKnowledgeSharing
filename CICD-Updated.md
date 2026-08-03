# SQL Server CI/CD Pipeline: Execution Location Guide

This guide shows exactly WHERE each command runs and WHICH MACHINE or LOCATION it needs to be executed on.

---

## Machine Setup Overview

You need TWO machines or ONE machine with TWO applications:

Machine 1: Your Windows Computer (Developer Machine)
- Has: Git, PowerShell, SQL Server Management Studio (SSMS)
- Purpose: Write code, commit to GitHub, run GitHub Actions runner

Machine 2: GitHub Cloud (Cloud Service)
- Has: GitHub Actions, GitHub Repository
- Purpose: Store code, trigger automated deployments
- You access it via web browser

Many people use ONE Windows machine for both roles.

---

## PHASE 1: INITIAL SQL SERVER SETUP

These commands run on YOUR WINDOWS MACHINE in SQL Server Management Studio.

### Step 1A: Open SQL Server Management Studio

Location: Your Windows Computer

Application: SQL Server Management Studio (SSMS)

Actions:
1. Click Start > Search "SQL Server Management Studio"
2. Open the application
3. Connect to your local SQL Server (usually "localhost" or "." or your server name)
4. Click "New Query" button

You now see a blank query window. This is where you type SQL commands.

### Step 1B: Create SQL Login for CI/CD

Location: Your Windows Computer
Application: SQL Server Management Studio (SSMS)
Query Window: Where you type SQL

Copy and paste this SQL into the query window:

```sql
CREATE LOGIN GitHub_CICD_User WITH PASSWORD = 'YourStrongPassword123!';
GO
```

Click Execute (or press F5).

Result: You see "Command(s) completed successfully" message.

### Step 1C: Create Database User and Grant Permissions

Location: Your Windows Computer
Application: SQL Server Management Studio (SSMS)
Query Window: Same query window as above

Clear the previous SQL. Copy and paste this SQL:

```sql
USE YourDatabaseName;
GO

CREATE USER Github_CICD_User FOR LOGIN Github_CICD_User;
GO

ALTER ROLE db_ddladmin ADD MEMBER Github_CICD_User;
GO

PRINT 'GitHub CI/CD user created successfully';
GO
```

Click Execute.

Result: Message shows "GitHub CI/CD user created successfully".

### Step 1D: Test the Login Works

Location: Your Windows Computer
Application: PowerShell (not SSMS)
Folder: Any folder (does not matter)

Open PowerShell. Type this command:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U Github_CICD_User -P YourStrongPassword123! -Q "SELECT @@VERSION;"
```

Press Enter.

Result: You see SQL Server version information printed.

If you see an error, the login failed. Go back and check the password is correct.

---

## PHASE 2: CREATE PROJECT FOLDER ON YOUR WINDOWS COMPUTER

### Step 2A: Create Project Folder

Location: Your Windows Computer
Application: PowerShell
Folder: You create a new one

Open PowerShell. Type these commands:

```powershell
mkdir F:\CICDDEMO
cd F:\CICDDEMO
```

Press Enter.

Result: Folder F:\CICDDEMO is created and you are now inside it.

Verify: Type `dir` and press Enter. Folder should be empty.

### Step 2B: Create Subfolders

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO (you should already be here from Step 2A)

Type these commands:

```powershell
mkdir Scripts\Database
mkdir .github\workflows
```

Press Enter.

Result: Two subfolders are created.

Verify: Type `dir` and press Enter. You see:

```
Mode    Name
----    ----
d----   Scripts
d----   .github
```

---

## PHASE 3: CREATE SQL SCRIPTS

### Step 3A: Create First SQL Script File

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO (the folder you created in Step 2)

Type this command:

```powershell
notepad Scripts\Database\001_CreateTables.sql
```

Press Enter.

Result: Notepad opens with a blank file named 001_CreateTables.sql.

### Step 3B: Add SQL Code to Script

Location: Your Windows Computer
Application: Notepad
File: 001_CreateTables.sql (which is now open)

Copy and paste this SQL code into Notepad:

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

Click File > Save.

Close Notepad.

Result: File 001_CreateTables.sql now exists in F:\CICDDEMO\Scripts\Database folder.

### Step 3C: Test SQL Script Manually

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Before pushing to GitHub, test the script to make sure it works.

Type this command:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U Github_CICD_User -P YourStrongPassword123! -i Scripts\Database\001_CreateTables.sql
```

Press Enter.

Result: You see SQL output:

```
Table Users created successfully

(1 rows affected)
UserID      UserName    Email               CreatedDate
----------- ----------- ------------------- -----------------------
1           TestUser    test@example.com    2024-01-15 10:30:00.000
```

This means the script works. Now you can push it to GitHub.

---

## PHASE 4: INITIALIZE GIT ON YOUR WINDOWS COMPUTER

### Step 4A: Initialize Git Repository

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO (make sure you are here)

Type these commands:

```powershell
cd F:\CICDDEMO
git init
```

Press Enter.

Result: Message shows "Initialized empty Git repository in F:/CICDDEMO/.git/".

### Step 4B: Configure Git Identity

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type these commands:

```powershell
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

Press Enter after each command.

Result: No output. Configuration is saved silently.

### Step 4C: Stage All Files for Commit

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
git add .
```

Press Enter.

Result: No output. All files in the folder are now staged.

Verify: Type `git status` and press Enter. You see:

```
On branch master

No commits yet

Changes to be committed:
  new file:   Scripts/Database/001_CreateTables.sql
```

### Step 4D: Create First Commit

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
git commit -m "Initial commit with SQL scripts"
```

Press Enter.

Result: Message shows:

```
[master (root-commit) 783532b] Initial commit with SQL scripts
 1 file changed, 15 insertions(+)
 create mode 100644 Scripts/Database/001_CreateTables.sql
```

This means your commit is saved locally.

---

## PHASE 5: CREATE GITHUB REPOSITORY

### Step 5A: Create Repository on GitHub

Location: GitHub Website (Cloud)
Application: Web Browser
Website: https://github.com

Actions:
1. Go to https://github.com and sign in
2. Click + icon (top right corner)
3. Click "New repository"
4. Repository name: SQL-CICD-DEMO
5. Description: Automated SQL Server deployment pipeline
6. Visibility: Private
7. Click "Create repository" button

Result: GitHub shows you setup instructions with a URL like:
```
https://github.com/YourUsername/SQL-CICD-DEMO.git
```

Copy this URL. You will use it next.

### Step 5B: Connect Local Folder to GitHub

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type these commands (replace YourUsername with your actual GitHub username):

```powershell
git remote add origin https://github.com/YourUsername/SQL-CICD-DEMO.git
git branch -M main
git push -u origin main
```

Press Enter after each command.

Result: Files upload to GitHub. You see:

```
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 203 bytes | 203.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
...
 * [new branch]      main -> main
```

This means your code is now on GitHub.

---

## PHASE 6: CREATE GITHUB ACTIONS WORKFLOW FILE

### Step 6A: Create Workflow File

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
notepad .github\workflows\deploy-sql.yml
```

Press Enter.

Result: Notepad opens with a blank file named deploy-sql.yml.

### Step 6B: Add Workflow Configuration

Location: Your Windows Computer
Application: Notepad
File: deploy-sql.yml (which is now open)

Copy and paste this YAML code into Notepad:

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

Click File > Save.

Close Notepad.

Result: File deploy-sql.yml now exists in F:\CICDDEMO\.github\workflows folder.

### Step 6C: Commit and Push Workflow File

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type these commands:

```powershell
git add .github/workflows/deploy-sql.yml
git commit -m "Add GitHub Actions workflow"
git push origin main
```

Press Enter after each command.

Result: Workflow file is now on GitHub.

---

## PHASE 7: ADD GITHUB SECRETS

These settings store your SQL Server credentials securely on GitHub.

### Step 7A: Open GitHub Secrets Page

Location: GitHub Website (Cloud)
Application: Web Browser

Actions:
1. Go to https://github.com/YourUsername/SQL-CICD-DEMO
2. Click "Settings" tab
3. In left sidebar, click "Secrets and variables"
4. Click "Actions"

You now see the Secrets page.

### Step 7B: Add SQL Server Host Secret

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Secrets and variables > Actions

Actions:
1. Click "New repository secret" button
2. Name: SQL_SERVER_HOST
3. Value: localhost (or your SQL Server name)
4. Click "Add secret" button

Secret is saved.

### Step 7C: Add Database Name Secret

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Secrets and variables > Actions

Actions:
1. Click "New repository secret" button
2. Name: SQL_DATABASE
3. Value: YourDatabaseName
4. Click "Add secret" button

Secret is saved.

### Step 7D: Add SQL Username Secret

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Secrets and variables > Actions

Actions:
1. Click "New repository secret" button
2. Name: SQL_USERNAME
3. Value: Github_CICD_User
4. Click "Add secret" button

Secret is saved.

### Step 7E: Add SQL Password Secret

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Secrets and variables > Actions

Actions:
1. Click "New repository secret" button
2. Name: SQL_PASSWORD
3. Value: YourStrongPassword123!
4. Click "Add secret" button

Secret is saved.

Verify: On the Secrets page, you now see four secrets listed:
- SQL_SERVER_HOST
- SQL_DATABASE
- SQL_USERNAME
- SQL_PASSWORD

---

## PHASE 8: DOWNLOAD GITHUB ACTIONS RUNNER

### Step 8A: Go to Runners Page

Location: GitHub Website (Cloud)
Application: Web Browser

Actions:
1. Go to https://github.com/YourUsername/SQL-CICD-DEMO
2. Click "Settings" tab
3. In left sidebar, click "Actions" > "Runners"
4. Click "New self-hosted runner" button

You now see the Runner setup page.

### Step 8B: Select Runner Configuration

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Runner setup

Actions:
1. Operating System: Select "Windows"
2. Architecture: Select "x64"

You now see a download button.

### Step 8C: Download Runner ZIP File

Location: GitHub Website (Cloud)
Application: Web Browser

Actions:
1. Click the download link
2. Save the ZIP file to Downloads folder

File name: actions-runner-win-x64-2.336.0.zip (version number may be different)

### Step 8D: Extract Runner to Folder

Location: Your Windows Computer
Application: File Explorer or PowerShell

Actions using PowerShell:

```powershell
cd C:\
mkdir GitHub-Runner
cd GitHub-Runner
```

Then extract the ZIP file into C:\GitHub-Runner folder.

You can also use File Explorer:
1. Open File Explorer
2. Go to C:\ drive
3. Create new folder "GitHub-Runner"
4. Extract the ZIP file into this folder

Result: Folder C:\GitHub-Runner contains runner files.

---

## PHASE 9: CONFIGURE GITHUB ACTIONS RUNNER

### Step 9A: Get Configuration Token

Location: GitHub Website (Cloud)
Application: Web Browser
Page: Runner setup

After downloading, GitHub shows you a token like:

```
AAAA...BBBB...CCCC
```

Copy this token. You will use it in the next step.

### Step 9B: Run Configuration Script

Location: Your Windows Computer
Application: PowerShell
Folder: C:\GitHub-Runner

Type these commands:

```powershell
cd C:\GitHub-Runner
.\config.cmd
```

Press Enter.

Result: Configuration wizard starts. You see prompts:

```
|GitHub Actions Runner Configuration|

Enter name of work folder: (press Enter for default)
```

Actions:
1. GitHub URL prompt: Type https://github.com and press Enter
2. Token prompt: Paste the token from GitHub and press Enter
3. Runner name prompt: Type sql-server-runner and press Enter
4. Work folder prompt: Press Enter to accept default

Result: Configuration completes with message "Settings Saved".

### Step 9C: Start the Runner

Location: Your Windows Computer
Application: PowerShell
Folder: C:\GitHub-Runner

Type this command:

```powershell
.\run.cmd
```

Press Enter.

Result: Runner starts and shows:

```
|GitHub Actions Runner|

Listening for Jobs
```

Keep this PowerShell window open. Do not close it.

This window must stay open for GitHub Actions to send jobs to your machine.

---

## PHASE 10: TEST THE PIPELINE

### Step 10A: Create a New SQL Script

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
notepad Scripts\Database\002_CreateOrders.sql
```

Press Enter.

Result: Notepad opens.

### Step 10B: Add SQL Code

Location: Your Windows Computer
Application: Notepad
File: 002_CreateOrders.sql

Copy and paste this SQL:

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

Click File > Save.

Close Notepad.

### Step 10C: Test Script Manually (Optional)

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U Github_CICD_User -P YourStrongPassword123! -i Scripts\Database\002_CreateOrders.sql
```

Press Enter.

Result: You see "Table Orders created successfully". Script works.

### Step 10D: Commit and Push to GitHub

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type these commands:

```powershell
git add Scripts\Database\002_CreateOrders.sql
git commit -m "Add Orders table"
git push origin main
```

Press Enter after each command.

Result: File is pushed to GitHub.

### Step 10E: Watch GitHub Actions Run

Location: GitHub Website (Cloud)
Application: Web Browser

Actions:
1. Go to https://github.com/YourUsername/SQL-CICD-DEMO
2. Click "Actions" tab
3. You see a workflow run with your commit message "Add Orders table"
4. Click on the workflow to see progress

Result: You see three steps running:
- Checkout code (green checkmark)
- Test SQL Connection (green checkmark)
- Execute SQL Scripts (green checkmark with script names)

### Step 10F: Monitor Runner Window

Location: Your Windows Computer
Application: PowerShell (the runner window)
Window: The one showing "Listening for Jobs"

You should see output:

```
Running job: deploy
Executing: 001_CreateTables.sql
Executing: 002_CreateOrders.sql
Job deploy completed with result: Succeeded
```

### Step 10G: Verify in SQL Server

Location: Your Windows Computer
Application: SQL Server Management Studio

Actions:
1. Open SSMS
2. Connect to your SQL Server
3. Expand Databases > YourDatabaseName > Tables
4. Refresh (press F5)
5. You should see:
   - dbo.Users table (from script 1)
   - dbo.Orders table (from script 2)

Both tables were automatically created by GitHub Actions.

---

## PHASE 11: DAILY WORKFLOW

### Step 11A: Create New SQL Script Locally

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
notepad Scripts\Database\003_AddStoredProcedures.sql
```

Press Enter.

### Step 11B: Write Your SQL Code

Location: Your Windows Computer
Application: Notepad

Write your SQL code. Example:

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
GO
```

Save the file.

### Step 11C: Test Locally (Optional but Recommended)

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type this command:

```powershell
sqlcmd -S localhost -d YourDatabaseName -U Github_CICD_User -P YourStrongPassword123! -i Scripts\Database\003_AddStoredProcedures.sql
```

Press Enter.

If it works locally, it will work in GitHub Actions.

### Step 11D: Commit and Push

Location: Your Windows Computer
Application: PowerShell
Folder: F:\CICDDEMO

Type these commands:

```powershell
git add Scripts\Database\003_AddStoredProcedures.sql
git commit -m "Add stored procedures"
git push origin main
```

Press Enter after each command.

### Step 11E: GitHub Actions Automatically Runs

Location: GitHub Website (Cloud)
Application: Web Browser
Action: Automatic (no manual action needed)

When you push code, GitHub Actions automatically:
1. Detects the push
2. Sends the job to your runner
3. Runner executes the SQL scripts
4. Results appear in the Actions tab

You just watch it happen.

---

## SUMMARY: LOCATION MAPPING

Here is where each phase runs:

| Phase | Name | Location | Application | Machine |
|-------|------|----------|-------------|---------|
| 1 | SQL Setup | SSMS + PowerShell | SQL Server Management Studio, PowerShell | Your Windows Computer |
| 2 | Create Folders | PowerShell | PowerShell | Your Windows Computer |
| 3 | Create SQL Scripts | Notepad + PowerShell | Notepad, PowerShell | Your Windows Computer |
| 4 | Initialize Git | PowerShell | PowerShell | Your Windows Computer |
| 5 | GitHub Repository | Web Browser | GitHub Website | Cloud (Internet) |
| 6 | Workflow File | Notepad + PowerShell | Notepad, PowerShell | Your Windows Computer |
| 7 | GitHub Secrets | Web Browser | GitHub Website | Cloud (Internet) |
| 8 | Download Runner | Web Browser | GitHub Website | Cloud (Internet) |
| 9 | Configure Runner | PowerShell | PowerShell | Your Windows Computer |
| 10 | Test Pipeline | PowerShell + Web Browser + SSMS | Multiple | Your Windows Computer + Cloud |
| 11 | Daily Usage | PowerShell + Web Browser | PowerShell, Web Browser | Your Windows Computer + Cloud |

---

## KEY POINTS TO REMEMBER

Your Windows Computer runs:
- SQL Server Management Studio (SSMS) for database work
- PowerShell for Git commands
- GitHub Actions Runner (always listening)

GitHub Cloud runs:
- Your code repository
- GitHub Actions automation
- Workflow execution orchestration

When you push code from PowerShell, GitHub detects it, sends a job to your runner, which executes SQL scripts on your local SQL Server database.

The runner must stay running 24/7 for automatic deployments. If you close the PowerShell window, GitHub Actions cannot run jobs.
