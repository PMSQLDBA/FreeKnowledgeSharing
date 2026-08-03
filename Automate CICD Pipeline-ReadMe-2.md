# SQL Server CI/CD Pipeline - Simple README

## What is This?

Automated SQL Server database deployments using GitHub. Push SQL scripts to GitHub, and they automatically run on your SQL Server database.

## How It Works

1. You write SQL scripts and push them to GitHub
2. GitHub automatically triggers a workflow
3. A Windows runner machine picks up the job
4. Your SQL scripts run on your SQL Server database
5. Done. Your database is updated.

## What You Need

- Windows Server or Windows 10 Pro
- SQL Server 2019 or later
- GitHub account
- Git installed
- PowerShell

## Quick Setup (10 Minutes)

### 1. Create SQL User

Open SQL Server Management Studio and run:

```sql
CREATE LOGIN Github_CICD_User WITH PASSWORD = 'MyPassword123!'
CREATE USER Github_CICD_User FOR LOGIN Github_CICD_User
ALTER ROLE db_owner ADD MEMBER Github_CICD_User
```

Write down the password.

### 2. Add GitHub Secrets

Go to: `https://github.com/YOUR_USERNAME/YOUR_REPO/settings/secrets/actions`

Click "New repository secret" and add these:

- Name: `SQL_SERVER` → Value: `pmsqldba` (your server name)
- Name: `SQL_DATABASE` → Value: `DBAScripts` (your database name)
- Name: `SQL_USERNAME` → Value: `Github_CICD_User`
- Name: `SQL_PASSWORD` → Value: `MyPassword123!`

### 3. Create Workflow File

Create file: `.github/workflows/deploy-sql.yml`

Copy-paste this:

```yaml
name: SQL Server Deployment

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
      - uses: actions/checkout@v4
      - name: Test Connection
        shell: powershell
        run: sqlcmd -S "${{ secrets.SQL_SERVER }}" -d "${{ secrets.SQL_DATABASE }}" -U "${{ secrets.SQL_USERNAME }}" -P "${{ secrets.SQL_PASSWORD }}" -C -Q "SELECT @@VERSION;"
      - name: Deploy Scripts
        shell: powershell
        run: |
          Get-ChildItem '.\Scripts' -Filter '*.sql' | ForEach-Object {
            Write-Host "Executing: $($_.Name)"
            sqlcmd -S "${{ secrets.SQL_SERVER }}" -d "${{ secrets.SQL_DATABASE }}" -U "${{ secrets.SQL_USERNAME }}" -P "${{ secrets.SQL_PASSWORD }}" -C -i $_.FullName
            if ($LASTEXITCODE -ne 0) { exit 1 }
          }
```

Commit and push:

```powershell
git add .github/workflows/deploy-sql.yml
git commit -m "Add deployment workflow"
git push origin main
```

### 4. Set Up Runner

Download runner:

```powershell
cd F:\
mkdir runners
cd runners
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.336.0/actions-runner-win-x64-2.336.0.zip -OutFile runner.zip
Expand-Archive runner.zip
```

Get token from: `https://github.com/YOUR_USERNAME/YOUR_REPO/settings/actions/runners`

Register runner:

```powershell
cmd /c config.cmd --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_TOKEN_HERE
```

Start runner:

```powershell
.\run.cmd
```

Keep this window open. It listens for deployment jobs.

### 5. Test It

Create SQL script:

```powershell
cd F:\GITHUB\SQL-CICD-DEMO
notepad Scripts\Users.sql
```

Add SQL:

```sql
CREATE TABLE Users (
    UserID INT PRIMARY KEY IDENTITY(1,1),
    UserName NVARCHAR(100) NOT NULL
)
```

Push to GitHub:

```powershell
git add Scripts\Users.sql
git commit -m "Add Users table"
git push origin main
```

Watch the runner window. You should see:

```
Running job: deploy
Job deploy completed with result: Succeeded
```

Check SQL Server - Users table should exist.

## Daily Usage

Add SQL files to Scripts folder:

```powershell
notepad Scripts\Orders.sql
```

Write SQL and push:

```powershell
git add Scripts\Orders.sql
git commit -m "Add Orders table"
git push origin main
```

Done. Your script runs automatically on SQL Server.

## Folder Structure

```
Your-Repository/
├── .github/
│   └── workflows/
│       └── deploy-sql.yml
├── Scripts/
│   ├── Users.sql
│   └── Orders.sql
└── README.md
```

Only SQL files go in Scripts folder. They run in alphabetical order.

## Best Practices

**Use numbered prefixes:**

- `001-CreateUsersTable.sql`
- `002-CreateOrdersTable.sql`
- `003-AddStoredProcedures.sql`

This ensures correct execution order.

**Write safe scripts:**

```sql
-- Good - Can run multiple times
IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'Users')
CREATE TABLE Users (UserID INT PRIMARY KEY)

-- Bad - Fails on second run
CREATE TABLE Users (UserID INT PRIMARY KEY)
```

**Test first:**

```powershell
sqlcmd -S pmsqldba -d DBAScripts -U Github_CICD_User -P MyPassword123! -C -i Scripts\Users.sql
```

## Troubleshooting

### Runner is Offline

Start it:

```powershell
cd F:\runners
.\run.cmd
```

Keep the window open.

### Connection Error

Test manually:

```powershell
sqlcmd -S pmsqldba -d DBAScripts -U Github_CICD_User -P MyPassword123! -C -Q "SELECT @@VERSION;"
```

Check GitHub secrets are correct.

### Script Didn't Run

Check GitHub Actions logs: `https://github.com/YOUR_USERNAME/YOUR_REPO/actions`

Common issues:
- Scripts folder is empty
- SQL has errors
- Wrong database name in secrets

## That's It

You now have automated SQL deployments. Every SQL script you push to GitHub automatically runs on your database.

No manual deployments. No mistakes. Just push and deploy.

## For More Details

See the `SQL-CICD-Technical-SOP.docx` file for complete step-by-step instructions, troubleshooting, and best practices.

## Questions?

Check the troubleshooting section above or review the detailed SOP document.
