# Git SOP for SQL Server DBA

## Daily Support, Change Management & CI/CD Deployments

---

# 1. Why Git for SQL Server DBA?

Without Git

```
Developer sends SQL script over email
        ↓
DBA saves on Desktop
        ↓
Script modified multiple times
        ↓
Nobody knows latest version
        ↓
Production issue
        ↓
Rollback becomes difficult
```

With Git

```
Developer
     ↓
Git Repository
     ↓
Code Review
     ↓
GitHub Actions
     ↓
QA
     ↓
Production
```

Everything is

* Version Controlled
* Auditable
* Rollback Available
* Automated

---

# Daily Folder Structure

```
SQL-CICD-DEMO

│
├── Scripts
│      001_Create_Table.sql
│      002_Create_Index.sql
│      003_StoredProcedure.sql
│
├── Rollback
│      Rollback_001.sql
│
├── Releases
│      Release_2026_08_05.sql
│
├── Documentation
│      README.md
│
└── .github
       workflows
            deploy.yml
```

---

# 1. Initialize Repository

Command

```bash
git init
```

Purpose

Creates Git repository.

Real-Time DBA Use

You created

```
F:\SQLDeployments
```

Start version controlling deployment scripts.

---

# 2. Clone Repository

```bash
git clone https://github.com/company/SQLDeployments.git
```

Use Case

Joining a new support project.

Instead of copying folders manually

Clone repository.

---

# 3. Configure Username

```bash
git config --global user.name "Praveen Kumar"
```

Use Case

Every deployment should show

```
Author

Praveen Kumar
```

---

# 4. Configure Email

```bash
git config --global user.email "praveen@email.com"
```

Useful for audit.

---

# 5. Check Status

```bash
git status
```

Most important command.

Shows

```
Modified

Deleted

New files

Staged files
```

Real-Time Example

You modified

```
CreateIndex.sql
```

Before deployment

```
git status
```

Output

```
modified:

Scripts/CreateIndex.sql
```

---

# 6. Add File

Single file

```bash
git add Script.sql
```

All files

```bash
git add .
```

Use Case

Today's deployment includes

```
3 Procedures

2 Indexes

1 Table
```

Stage them.

---

# 7. Commit

```bash
git commit -m "Added Customer Index Optimization"
```

Good Commit

```
Fixed deadlock in SalesOrder Procedure

Added missing index

Updated SQL Agent Job

Created Monthly Archive Job
```

Bad Commit

```
Updated

Changes

Final

Latest
```

---

# 8. View History

```bash
git log
```

Real-Time

Production issue occurred yesterday.

Find

```
Who changed procedure?

When?

Commit ID?
```

---

# 9. Branch

Current branches

```bash
git branch
```

---

# 10. Create Branch

```bash
git branch Release_Aug2026
```

Real-Time

Monthly Release

```
Main

↓

Release_Aug2026
```

---

# 11. Switch Branch

```bash
git switch Release_Aug2026
```

or

```bash
git checkout Release_Aug2026
```

---

# 12. Create + Switch

```bash
git checkout -b Hotfix_Deadlock
```

Real-Time

Production deadlock.

Create Hotfix branch.

---

# 13. Merge

```bash
git merge Hotfix_Deadlock
```

After testing

Merge into Main.

---

# 14. Compare

```bash
git diff
```

Real-Time

Compare

```
Old Procedure

vs

New Procedure
```

---

# 15. Push

```bash
git push origin main
```

Uploads local commits to GitHub.

In your SQL CI/CD

This triggers

```
GitHub Actions

↓

Deploy SQL Script

↓

SQL Server
```

---

# 16. Push First Time

```bash
git push -u origin main
```

Used during first deployment.

---

# 17. Pull Latest

```bash
git pull
```

Always execute

Before starting work.

Otherwise

```
Push Rejected

Fetch First
```

Exactly the error you recently encountered.

---

# 18. Fetch

```bash
git fetch
```

Downloads latest changes

Doesn't merge.

Good for production repositories.

---

# 19. List Remote

```bash
git remote -v
```

Shows

```
origin

https://github...
```

Useful when troubleshooting push failures.

---

# 20. Add Remote

```bash
git remote add origin https://github.com/PMSQLDBA/CICDDEMO.git
```

You already executed this.

---

# 21. Remove Remote

```bash
git remote remove origin
```

Use when repository URL changes.

---

# 22. Clone Specific Branch

```bash
git clone -b Release https://github.com/company/repo.git
```

Useful

Support only Release branch.

---

# 23. Push Tags

```bash
git push --tags
```

Release Example

```
Version

v1.0

v1.1

v2.0
```

---

# 24. Discard Changes

```bash
git restore Script.sql
```

Real-Time

Mistakenly modified

```
Customer Procedure
```

Discard changes.

---

# 25. Reset Soft

```bash
git reset --soft HEAD~1
```

Undo commit

Keep code.

---

# 26. Reset Mixed

```bash
git reset --mixed HEAD
```

Remove staging.

---

# 27. Reset Hard

```bash
git reset --hard HEAD
```

Danger

Deletes all local modifications.

---

# 28. Revert

```bash
git revert CommitID
```

Production Best Practice

Never delete production history.

Instead

Revert.

---

# 29. Stash

```bash
git stash
```

Real-Time

Working on

```
Index Optimization
```

Production issue arrives.

Stash work.

Switch branch.

Fix issue.

Return later.

---

# 30. Stash Pop

```bash
git stash pop
```

Restore previous work.

---

# 31. Clean

```bash
git clean -fd
```

Deletes

```
Temp files

Backup files

Unused scripts
```

---

# 32. Reflog

```bash
git reflog
```

DBA Life Saver

Recover deleted commits.

---

# 33. Abort Merge

```bash
git merge --abort
```

When merge conflicts occur.

---

# Complete Daily DBA Workflow

```
Morning

git pull

↓

Receive CRQ

↓

Create Branch

git checkout -b CRQ_4589

↓

Modify SQL Scripts

↓

git status

↓

git add .

↓

git commit

↓

git push

↓

GitHub Actions

↓

Validation

↓

Deploy QA

↓

Testing

↓

Merge

↓

Production Approval

↓

git pull

↓

Deploy

↓

Tag Release

↓

Archive
```

---

# Monthly Release Workflow

```
Sprint

↓

Developer SQL Scripts

↓

Git Branch

↓

DBA Review

↓

Performance Review

↓

Syntax Validation

↓

Merge

↓

GitHub Actions

↓

QA Deployment

↓

User Testing

↓

Production Deployment

↓

Tag Release

↓

Rollback Script Stored
```

---

# Best Practices for SQL Server DBAs

| Best Practice                                                           | Why It Matters                                              |
| ----------------------------------------------------------------------- | ----------------------------------------------------------- |
| One change request (CRQ) per branch                                     | Isolates changes and simplifies rollback.                   |
| Always include rollback scripts                                         | Enables rapid recovery if deployment fails.                 |
| Never commit passwords or connection strings                            | Store secrets in GitHub Secrets or Azure Key Vault.         |
| Keep scripts idempotent where possible (`IF EXISTS`, `CREATE OR ALTER`) | Safe re-execution during deployments.                       |
| Use meaningful commit messages                                          | Improves auditability and troubleshooting.                  |
| Pull before you push                                                    | Prevents non-fast-forward push errors.                      |
| Review `git diff` before committing                                     | Avoids accidental deployment of unrelated changes.          |
| Tag production releases                                                 | Makes it easy to identify and restore a known-good version. |
| Separate DDL, DML, and rollback scripts                                 | Improves deployment control and review.                     |
| Automate deployments through GitHub Actions                             | Reduces manual errors and provides consistent deployments.  |

# Typical End-to-End SQL Server CI/CD Flow

```text
Developer
    │
    ▼
Write T-SQL Script
    │
    ▼
git status
git add .
git commit
git push
    │
    ▼
GitHub Repository (main / release branch)
    │
    ▼
GitHub Actions Workflow
    │
    ├── Validate SQL syntax
    ├── Check script naming standards
    ├── Execute pre-deployment checks
    ├── Run sqlcmd / SqlPackage / Flyway (as applicable)
    ├── Execute deployment on QA
    ├── Run post-deployment validation
    └── Notify Teams/Email
    │
    ▼
User Acceptance Testing
    │
    ▼
Production Approval
    │
    ▼
Production Deployment
    │
    ▼
Tag Release (e.g., v2026.08.04)
    │
    ▼
Store Rollback Script and Deployment Logs
```

This SOP aligns well with daily SQL Server DBA support, controlled database releases, and GitHub Actions–based deployment automation, providing a repeatable and auditable process from script development through production deployment.
