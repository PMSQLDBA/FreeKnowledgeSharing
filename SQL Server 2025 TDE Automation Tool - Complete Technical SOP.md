# SQL Server 2025 TDE Automation Tool - Complete Technical SOP
## End-to-End Standard Operating Procedure

**Document Version:** 1.0  
**Created:** July 18, 2026  
**Tool Version:** 3.3.1 Professional Edition  
**Author:** SQL DBA Champs (Praveen Madupu)  
**Audience:** Database Administrators, Infrastructure Team

---

## TABLE OF CONTENTS

1. [System Requirements](#system-requirements)
2. [Pre-Implementation Checklist](#pre-implementation-checklist)
3. [Installation & Setup](#installation--setup)
4. [Connection Configuration](#connection-configuration)
5. [TDE Implementation Workflow](#tde-implementation-workflow)
6. [Certificate Management](#certificate-management)
7. [Monitoring & Maintenance](#monitoring--maintenance)
8. [Troubleshooting](#troubleshooting)
9. [Emergency Procedures](#emergency-procedures)
10. [Appendices](#appendices)

---

## 1. SYSTEM REQUIREMENTS

### Hardware Requirements
```
Minimum:
  - Processor: Intel Core i5 or equivalent (2+ GHz)
  - RAM: 4 GB minimum, 8 GB recommended
  - Disk Space: 500 MB for tool + space for certificate backups
  - Network: 100 Mbps connection to SQL Server

Recommended:
  - Processor: Intel Core i7 or higher
  - RAM: 16 GB+
  - Disk Space: 1 TB+ for backups
  - Network: 1 Gbps connection
```

### Software Requirements
```
Operating System:
  ✓ Windows Server 2016 or later
  ✓ Windows 10/11 Professional or Enterprise
  ✓ Windows Server 2019, 2022, 2025

SQL Server Versions:
  ✓ SQL Server 2016 SP3 or later
  ✓ SQL Server 2017
  ✓ SQL Server 2019
  ✓ SQL Server 2022
  ✓ SQL Server 2025

Python:
  ✓ Python 3.10 or later
  ✓ Python 3.11 recommended
  ✓ Python 3.12 supported

Required Python Packages:
  - PyQt5 (GUI framework)
  - pyodbc (SQL Server connectivity)
  - pyperclip (clipboard operations)

Required System Components:
  ✓ ODBC Driver 18 for SQL Server
  ✓ .NET Framework 4.7.2+
  ✓ Visual C++ Redistributable 2019+
```

### Network Requirements
```
Connectivity:
  ✓ TCP/IP connection to SQL Server (port 1433 default)
  ✓ Windows Authentication (preferred) OR SQL Server Authentication
  ✓ Network latency < 500ms for optimal performance

Firewall Rules:
  ✓ Allow outbound TCP port 1433 to SQL Server
  ✓ Allow outbound TCP port 445 (SMB) for file backups
  ✓ Allow named pipes (if configured)

Permissions:
  ✓ sysadmin role on SQL Server
  ✓ File system access to backup location (write permissions)
  ✓ Local administrator rights on tool execution machine
```

---

## 2. PRE-IMPLEMENTATION CHECKLIST

### 2.1 Database Readiness Assessment

**Check List:**
```
□ Database exists and is online
□ Database is not in read-only mode
□ Database is not a system database (master, model, msdb, tempdb)
□ Database has no ongoing TDE operation
□ Database has valid backup
□ Database is not in the middle of log backup
□ Database has sufficient free disk space (>10 GB minimum)
□ Database has sufficient transaction log space
```

**Verification Script:**
```sql
-- Run on target database
SELECT 
    DATABASEPROPERTYEX('YourDatabaseName', 'Status') AS DatabaseStatus,
    DATABASEPROPERTYEX('YourDatabaseName', 'IsReadOnly') AS IsReadOnly,
    DATABASEPROPERTYEX('YourDatabaseName', 'Recovery') AS RecoveryModel,
    (SELECT SUM(size * 8 / 1024) FROM sys.database_files) AS DatabaseSizeMB,
    (SELECT SUM(free_space_in_bytes / 1024 / 1024) FROM sys.dm_os_volume_info) AS FreeSpaceMB

-- Check for existing TDE
SELECT database_id, encryption_state, encryption_scan_state
FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID('YourDatabaseName')
```

### 2.2 Environment Preparation

**Pre-Deployment Tasks:**
```
□ Notify development team (if applicable)
□ Schedule maintenance window
□ Prepare rollback plan
□ Test tool in non-production first
□ Document baseline encryption status
□ Backup all databases to be encrypted
□ Test restore procedures
□ Prepare communication to users
□ Have DBA on standby
```

### 2.3 Security Preparation

**Certificate & Key Management:**
```
□ Define certificate naming convention
  Format: {DatabaseName}_TDE_Cert
  Example: Karthikeya_TDE_Cert

□ Define backup location
  Recommendation: E:\TDE_Backups\{DatabaseName}\
  
□ Define key encryption password
  Requirement: Strong password (12+ chars, special chars, numbers)
  Storage: Secure password manager (NOT in code)
  
□ Plan key rotation schedule
  Recommended: Every 1 year
  Document in calendar
```

### 2.4 Access & Permissions

**Verify User Permissions:**
```sql
-- Check current user roles
SELECT dp.name, dp.type_desc, dp.permission_name
FROM sys.database_principals dp
WHERE dp.name = USER_NAME()

-- Verify sysadmin role
SELECT IS_SRVROLEMEMBER('sysadmin') AS IsSysAdmin

-- Verify file system permissions
-- (Manual check: verify write access to backup location)
```

---

## 3. INSTALLATION & SETUP

### 3.1 Python Installation (Windows)

**Step 1: Download Python**
```
1. Go to https://www.python.org/downloads/
2. Download Python 3.11 or 3.12
3. Choose "Windows Installer"
4. Download to C:\Installers\python-3.x.x-amd64.exe
```

**Step 2: Install Python**
```
1. Run installer as Administrator
2. Check: "Add Python to PATH"
3. Check: "Install pip"
4. Choose: "Install for all users"
5. Installation directory: C:\Python311\ (or latest version)
6. Complete installation
```

**Step 3: Verify Installation**
```bash
# Open Command Prompt (cmd.exe)
python --version
# Expected: Python 3.11.x or higher

pip --version
# Expected: pip 23.x.x from C:\Python311\lib\site-packages\pip

python -c "import sys; print(sys.executable)"
# Should show: C:\Python311\python.exe
```

### 3.2 ODBC Driver Installation

**Step 1: Download ODBC Driver 18**
```
1. Go to Microsoft Downloads
2. Search: "ODBC Driver 18 for SQL Server"
3. Download: msodbcsql.msi (latest version)
4. Save to: C:\Installers\msodbcsql.msi
```

**Step 2: Install ODBC Driver**
```
1. Run msodbcsql.msi as Administrator
2. Accept license terms
3. Choose: "Add ODBC Driver to PATH"
4. Complete installation
5. Restart computer (recommended)
```

**Step 3: Verify Installation**
```bash
# In Command Prompt
odbcad32.exe
# Should open ODBC Data Source Administrator
# Go to "Drivers" tab
# Should see: "ODBC Driver 18 for SQL Server"
```

### 3.3 Required Python Packages

**Step 1: Install Packages**
```bash
# Open Command Prompt as Administrator
pip install PyQt5==5.15.9
pip install pyodbc==5.1.1
pip install pyperclip==1.8.2
```

**Step 2: Verify Installation**
```bash
# Test each package
python -c "import PyQt5; print('PyQt5 OK')"
python -c "import pyodbc; print('pyodbc OK')"
python -c "import pyperclip; print('pyperclip OK')"

# Expected output: All three should print "OK"
```

### 3.4 Tool Installation

**Step 1: Create Directory Structure**
```bash
# Create main directory
mkdir C:\TDE_Tool
mkdir C:\TDE_Tool\backups
mkdir C:\TDE_Tool\logs
mkdir C:\TDE_Tool\config
```

**Step 2: Copy Tool File**
```
1. Download: tde_automation_gui_v3_1_final.py
2. Copy to: C:\TDE_Tool\
3. Verify file size: ~90 KB
4. Verify permissions: Read/Execute for current user
```

**Step 3: Create Batch Launcher**
```batch
# Create file: C:\TDE_Tool\launch_tde_tool.bat
@echo off
REM TDE Automation Tool Launcher
cd /d C:\TDE_Tool
python tde_automation_gui_v3_1_final.py
pause
```

**Step 4: Create Shortcut**
```
1. Right-click launch_tde_tool.bat
2. Create shortcut
3. Rename: "SQL Server TDE Automation Tool"
4. Right-click shortcut → Properties
5. Shortcut tab → Advanced → Check "Run as administrator"
6. Move to Desktop or Taskbar
```

---

## 4. CONNECTION CONFIGURATION

### 4.1 First-Time Launch

**Step 1: Launch Tool**
```
1. Double-click shortcut or batch file
2. Tool window opens (1600x950 pixels)
3. You'll see: "Status: Not Connected"
4. Connection section is visible at top
```

**Step 2: Connection Parameters**

**For Windows Authentication:**
```
Authentication Type: Windows Authentication
Server: PHSQLDBA (your SQL Server instance name)
Database: master (or specific database)
Username: Not required
Password: Not required
```

**For SQL Server Authentication:**
```
Authentication Type: SQL Server Authentication
Server: PHSQLDBA
Database: master
Username: sa (or SQL user)
Password: (your SQL Server password)
```

**Step 3: Connect to SQL Server**
```
1. Fill in Server name: PHSQLDBA
2. Set Authentication Type
3. Fill Database: master
4. Click "✓ Connect" button
5. Wait for connection confirmation
6. Status should show: "✓ Connected to PHSQLDBA.master"
```

**Step 4: Verify Connection**
```
If successful:
  ✓ Status bar shows green "✓ Connected"
  ✓ All buttons become enabled
  ✓ Ready to proceed

If failed:
  ✗ Error message appears
  → Check server name spelling
  → Check SQL Server is running
  → Check network connectivity
  → Check credentials
  → Retry connection
```

### 4.2 Connection Troubleshooting

**Common Connection Errors:**

| Error | Cause | Solution |
|-------|-------|----------|
| "Named instance not found" | Wrong server name | Verify with: `sqlcmd -S servername -E` |
| "Login failed" | Wrong credentials | Re-enter username/password |
| "Connection timeout" | Network issue | Check firewall, ping server |
| "ODBC Driver not found" | Driver not installed | Install ODBC Driver 18 |

**Test Connection Script:**
```bash
# Test with sqlcmd (SQL Server command-line tool)
# If SQL Server Tools installed:
sqlcmd -S PHSQLDBA -E
# Should connect successfully
# Type: EXIT to quit
```

---

## 5. TDE IMPLEMENTATION WORKFLOW

### 5.1 Pre-Implementation Steps

**Step 1: Fill Operation Parameters**
```
Target Database: SampleDB (or your database name)
Backup Path: E:\path\TDE_Testing\SampleDB_1_PRE.bak
Certificate Name (Auto): SampleDB_TDE_Cert
Master Key Password: ••••••••••• (strong password)
Certificate Backup Path (Auto): E:\path\TDE_Testing\SampleDB_TDE_Cert.cer
Private Key Backup Path (Auto): E:\path\TDE_Testing\SampleDB_TDE_Key.pvk
Key Encryption Password: ••••••••••• (strong password)
```

**Step 2: Auto-Generate Parameters**
```
1. Enter Target Database: SampleDB
2. Enter Backup Path: E:\path\TDE_Testing\SampleDB.bak
3. Click "Auto-Generate Parameters"
4. Verify generated values:
   - Certificate Name: SampleDB_TDE_Cert ✓
   - Cert Backup: E:\path\TDE_Testing\SampleDB_TDE_Cert.cer ✓
   - Key Backup: E:\path\TDE_Testing\SampleDB_TDE_Key.pvk ✓
5. Verify all paths use backslashes (\) not forward slashes (/)
```

### 5.2 Execute 8-Phase Workflow

**Phase 1: Backup Database**
```
Button: "1️⃣ Backup Database"

What Happens:
  - Creates copy-only backup (doesn't affect backups)
  - Uses path specified: E:\path\TDE_Testing\SampleDB.bak
  - Takes 1-5 minutes depending on database size
  
Expected Output:
  [timestamp] Starting copy-only backup of SampleDB...
  [timestamp] Backup path: E:\path\TDE_Testing\SampleDB.bak
  [timestamp] Query executed successfully
  ✓ Backup completed successfully at E:\path\TDE_Testing\SampleDB.bak
  
If Fails:
  - Check backup path exists and is writable
  - Check disk space available (> database size)
  - Retry Phase 1
```

**Phase 2: Create Master Key**
```
Button: "2️⃣ Create Master Key"

What Happens:
  - Creates database master key (required for certificates)
  - Uses password you provided
  - Skips if already exists (idempotent)
  
Expected Output:
  [timestamp] Checking if Database Master Key exists...
  [timestamp] Creating new Database Master Key...
  [timestamp] Query executed successfully
  ✓ Master Key created successfully
  
OR (if already exists):
  [timestamp] ✓ Database Master Key already exists - Skipping creation
  ✓ Master Key already exists (skipped)
```

**Phase 3: Create Certificate**
```
Button: "3️⃣ Create Certificate"

What Happens:
  - Creates TDE certificate in master database
  - Certificate name: SampleDB_TDE_Cert
  - Valid for 1 year from creation
  - Skips if already exists (idempotent)
  
Expected Output:
  [timestamp] Checking if Certificate 'SampleDB_TDE_Cert' exists...
  [timestamp] Creating certificate 'SampleDB_TDE_Cert'...
  [timestamp] Query executed successfully
  ✓ Certificate created successfully
  
OR (if already exists):
  [timestamp] ✓ Certificate 'SampleDB_TDE_Cert' already exists - Skipping creation
  ✓ Certificate already exists (skipped)
```

**Phase 4: Backup Certificate**
```
Button: "4️⃣ Backup Certificate"

What Happens:
  - Exports certificate to .cer file (public)
  - Exports private key to .pvk file (encrypted)
  - Files encrypted with password you provided
  - CRITICAL: These files needed for disaster recovery
  
Expected Output:
  [timestamp] Backing up certificate and private key...
  [timestamp] Query executed successfully
  ✓ Certificate backed up successfully
  
Files Created:
  - E:\path\TDE_Testing\SampleDB_TDE_Cert.cer (public key)
  - E:\path\TDE_Testing\SampleDB_TDE_Key.pvk (encrypted private key)
  
Backup Location Important:
  - Store copies on secure location
  - Off-site backup recommended
  - Document password in secure location
  - Test restore procedures
```

**Phase 5: Create DEK (Database Encryption Key)**
```
Button: "5️⃣ Create DEK"

What Happens:
  - Creates DEK protected by certificate
  - DEK is what actually encrypts database
  - Skips if already exists (idempotent)
  
Expected Output:
  [timestamp] Creating Database Encryption Key for SampleDB...
  [timestamp] Query executed successfully
  ✓ DEK created successfully
  
OR (if already exists):
  [timestamp] ✓ DEK already exists for SampleDB - Skipping creation
  ✓ DEK already exists (skipped)
```

**Phase 6: Enable TDE**
```
Button: "6️⃣ Enable TDE"

What Happens:
  - Activates encryption on database
  - Database remains online (transparent)
  - May require log backup (if TDE was partially started)
  - Encryption starts immediately
  
Expected Output:
  [timestamp] Enabling TDE for SampleDB...
  [timestamp] Query executed successfully
  ✓ TDE enabled successfully
  
If Error (Already Encrypted):
  [timestamp] Cannot enable database encryption because it is already enabled
  → This means TDE already active, proceed to Phase 7
  
If Error (Pending Log Backup):
  [timestamp] Database has changes from previous encryption scans
  → Take transaction log backup manually:
     BACKUP LOG [SampleDB] TO DISK = 'E:\path\SampleDB.trn'
  → Retry Phase 6
```

**Phase 7: Verify Status**
```
Button: "7️⃣ Verify Status"

What Happens:
  - Checks encryption status
  - Reports encryption state and progress
  - Confirms certificate is protecting database
  
Expected Output:
  [timestamp] Verifying TDE status for SampleDB...
  [timestamp] TDE Status: ENCRYPTED, Progress: 0.0%
  ✓ Status retrieved successfully
  
Status Meanings:
  ENCRYPTED = Database is fully encrypted (or was already encrypted)
  Progress: 0.0% = Encryption complete or already was complete
  
Note:
  - If Progress < 100%: Encryption still in progress, wait and check later
  - If ENCRYPTED and Progress 0.0%: Already complete, normal
```

**Phase 8: Monitor Progress**
```
Button: "8️⃣ Monitor Progress"

What Happens:
  - Polls encryption progress every 5 seconds
  - Shows real-time status
  - Continues until encryption complete (or manually stopped)
  - Takes minutes to hours depending on database size
  
Expected Output:
  [timestamp] Status: ENCRYPTED, Progress: 0.0%
  [timestamp+5s] Status: ENCRYPTED, Progress: 0.0%
  [timestamp+10s] Status: ENCRYPTED, Progress: 0.0%
  ... continues polling ...
  
Exit Monitoring:
  - Encryption complete when Progress reaches 100% or remains at 0.0%
  - You can stop at any time (doesn't interrupt encryption)
  - Close output tab or click another operation
```

### 5.3 Post-Implementation Verification

**Verification Checklist:**
```sql
-- Run on SQL Server (in master database)

-- 1. Verify certificate exists
SELECT name, expiry_date FROM sys.certificates 
WHERE name = 'SampleDB_TDE_Cert'
-- Expected: Should return 1 row

-- 2. Verify DEK exists
SELECT database_id, encryption_state, encryptor_thumbprint 
FROM sys.dm_database_encryption_keys 
WHERE database_id = DB_ID('SampleDB')
-- Expected: encryption_state = 3 (ENCRYPTED)

-- 3. Verify database is encrypted
SELECT name, is_encrypted FROM sys.databases 
WHERE name = 'SampleDB'
-- Expected: is_encrypted = 1 (TRUE)

-- 4. Check certificate backing up database
SELECT dek.database_id, c.name, c.expiry_date
FROM sys.dm_database_encryption_keys dek
LEFT JOIN sys.certificates c ON dek.encryptor_thumbprint = c.thumbprint
WHERE dek.database_id = DB_ID('SampleDB')
-- Expected: Shows certificate name and expiry date
```

---

## 6. CERTIFICATE MANAGEMENT

### 6.1 Check Certificate Expiry

**How to Use:**
```
1. Scroll to bottom of tool window
2. Find "Maintenance & Emergency" section
3. Click "📅 Check Expiry" button (blue)
4. Wait 1-2 seconds for result
```

**Expected Output:**
```
[timestamp] Checking certificate expiry for 'SampleDB_TDE_Cert'...

════════════════════════════════════════════════════════════════
CERTIFICATE EXPIRY STATUS
════════════════════════════════════════════════════════════════
Certificate: SampleDB_TDE_Cert
Expiry Date: 2027-07-18 21:30:13
Days Remaining: 365
Status: ✅ OK (> 90 days)
✓ Certificate valid, continue monitoring
════════════════════════════════════════════════════════════════

✓ Certificate expires in 365 days
```

**Status Indicators:**
```
✅ OK (> 90 days)
   Status: Green, no action needed
   Action: Continue monitoring

🟡 WARNING (30-90 days)
   Status: Yellow, plan renewal
   Action: Schedule renewal within 90 days

🔴 URGENT (< 30 days)
   Status: Red, renewal needed
   Action: Renew certificate within 30 days

❌ EXPIRED (< 0 days)
   Status: Critical
   Action: IMMEDIATE - Emergency renewal required
```

### 6.2 Renew Certificate

**Pre-Renewal Checklist:**
```
□ Verified certificate expires within 30 days
□ Scheduled maintenance window
□ Notified users of brief downtime (2-3 seconds)
□ Backup location has sufficient space
□ Backup path is accessible
□ Have strong password ready for new certificate
```

**Renewal Process:**
```
1. Click "🔄 Renew Cert" button (green)
2. Watch progress in output window
3. Process takes 15-30 minutes total:
   - Steps 1-3: ~5 minutes
   - Step 4 (DEK switch): ~8 seconds downtime
   - Steps 5-6: ~2 minutes

Expected Timeline:
[21:28:29] Starting certificate renewal for SampleDB...
[21:28:29] Step 1: Verifying current certificate...
[21:28:29] Current certificate expires in 366 days
[21:28:29] Step 2: Creating new certificate 'SampleDB_TDE_Cert_New'...
[21:28:29] Query executed successfully
[21:28:29] Step 3: Backing up new certificate...
[21:28:29] Query executed successfully
[21:28:29] ✓ New certificate backed up to:
[21:28:29]   .cer: E:\path\TDE_Testing\SampleDB_TDE_Cert_New.cer
[21:28:29]   .pvk: E:\path\TDE_Testing\SampleDB_TDE_Key_New.pvk
[21:28:29] Step 4: Switching DEK to new certificate...
[21:28:29] Query executed successfully
[21:28:29] Step 5: Verifying new certificate...
[21:28:29] ✓ DEK successfully switched to 'SampleDB_TDE_Cert_New'

════════════════════════════════════════════════════════════════
CERTIFICATE RENEWAL SUMMARY
════════════════════════════════════════════════════════════════
Database: SampleDB
Old Certificate: SampleDB_TDE_Cert
New Certificate: SampleDB_TDE_Cert_New
Backup Files:
  - E:\path\TDE_Testing\SampleDB_TDE_Cert_New.cer
  - E:\path\TDE_Testing\SampleDB_TDE_Key_New.pvk
Status: RENEWED & ACTIVE
Next Renewal: 2027-07-18
════════════════════════════════════════════════════════════════

✓ Certificate renewal completed successfully
```

**Post-Renewal Steps:**
```
1. Verify output shows all 6 steps completed
2. Test database access:
   - Run SELECT query
   - Verify data returns normally
   - Check no connection errors
   
3. Confirm encryption still active:
   - Click "7️⃣ Verify Status"
   - Should show: ENCRYPTED
   
4. Document renewal:
   - Record date/time of renewal
   - Note old and new certificate names
   - Save backup file locations
   - Update renewal schedule for next year
   
5. Notify team:
   - Send email: "Certificate renewal completed successfully"
   - Mention: Database encryption remains active
   - Note: Next renewal needed July 18, 2027
```

### 6.3 Certificate Renewal Schedule

**Recommended Schedule:**
```
January: Quarterly check
  □ Click "📅 Check Expiry"
  □ Verify certificate status
  □ Document findings

April: Planning begins (90 days before)
  □ Click "📅 Check Expiry"
  □ Status should show 🟡 WARNING
  □ Start planning renewal
  □ Check backup path space

July 1: Urgent phase begins (30 days before)
  □ Click "📅 Check Expiry"
  □ Status should show 🔴 URGENT
  □ Schedule maintenance window
  □ Notify users
  □ Prepare for renewal

July 15-17: Execution phase
  □ During maintenance window
  □ Click "🔄 Renew Cert"
  □ Monitor completion
  □ Verify database access

July 18: DEADLINE
  □ If not renewed: Emergency procedures
  □ Certificate expires this date
  □ Must not miss this date

August: Cleanup (optional)
  □ Remove old certificate backups
  □ Archive new certificate backups
  □ Document in log
```

---

## 7. MONITORING & MAINTENANCE

### 7.1 Daily Monitoring

**Daily Tasks (Off-Peak):**
```
1. Check Database Availability
   SELECT name, state_desc FROM sys.databases 
   WHERE name = 'SampleDB'
   -- Expected: state_desc = 'ONLINE'

2. Monitor TDE Status
   Click "📊 Get Status" in tool
   -- Expected: Status = ENCRYPTED

3. Check Application Logs
   -- Look for encryption-related errors
   -- Should be none

4. Verify Backup Completion
   -- Confirm daily backups running
   -- Check backup files have reasonable size
```

### 7.2 Weekly Monitoring

**Weekly Tasks (After backups complete):**
```
Monday or Tuesday:
  1. Run full health check
  2. Verify certificate still valid (click "📅 Check Expiry")
  3. Check log file sizes
  4. Review SQL Server error log
  5. Verify backup integrity
  
Report:
  - Database Status: ONLINE ✓
  - Encryption Status: ENCRYPTED ✓
  - Certificate: Valid (X days remaining) ✓
  - No errors in logs ✓
```

### 7.3 Monthly Monitoring

**Monthly Tasks (First week of month):**
```
1. Comprehensive Status Check
   - Run all verification queries
   - Check encryption progress
   - Verify all certificates
   
2. Backup Verification
   - Test restore of database backup
   - Test restore of certificate backups
   - Verify backup file integrity
   
3. Performance Check
   - Monitor query performance (should be unaffected)
   - Check wait statistics
   - Review slow query logs
   
4. Documentation Update
   - Update certificate tracking spreadsheet
   - Record any issues
   - Update renewal calendar
   
5. Security Review
   - Verify certificate access restrictions
   - Check backup file permissions
   - Audit who accessed certificates
```

### 7.4 Quarterly Monitoring

**Quarterly Tasks (End of each quarter):**
```
Comprehensive Review:
  □ Audit all encrypted databases
  □ Verify all certificates active
  □ Check all backup locations
  □ Review performance metrics
  □ Validate disaster recovery procedures
  □ Update documentation
  
Renewal Planning:
  □ 90 days before expiry: Start renewal planning
  □ 60 days before: Finalize renewal window
  □ 30 days before: Final planning & notification
  
Report to Management:
  - Encryption status: All databases encrypted
  - Certificate status: Valid until [date]
  - Backup status: All backups successful
  - Performance impact: Negligible
  - Planned actions: Certificate renewal on [date]
```

### 7.5 Annual Maintenance

**Yearly Tasks:**
```
January (Start of Year):
  □ Update certificate renewal calendar
  □ Review previous year's issues
  □ Plan renewal windows for all databases
  □ Train new team members
  □ Update disaster recovery procedures
  
Throughout Year:
  □ Monitor certificate expiry (monthly)
  □ Execute renewals (as needed)
  □ Document all actions
  
December (End of Year):
  □ Verify all certificates still valid
  □ Archive completed renewals
  □ Plan next year's schedule
  □ Update documentation
  □ Review lessons learned
  □ Plan improvements
```

---

## 8. TROUBLESHOOTING

### 8.1 Connection Issues

**Problem: Cannot connect to SQL Server**
```
Error: "Named instance not found"
Or: "Connection timeout"

Solution:
  1. Verify server name:
     - Open SQL Server Management Studio
     - Check Instance name in Object Explorer
     - Use exact name from SSMS
  
  2. Check SQL Server running:
     - Windows > Services
     - Look for "SQL Server (INSTANCENAME)"
     - Should be "Running"
     - If stopped: Start it
  
  3. Check network connectivity:
     - Open Command Prompt
     - Type: ping PHSQLDBA
     - Should show responses
     - If timeout: Network issue, check firewall
  
  4. Check ODBC Driver:
     - Windows > Control Panel > ODBC Data Source Administrator
     - Go to "Drivers" tab
     - Look for "ODBC Driver 18 for SQL Server"
     - If not found: Reinstall driver
  
  5. Verify credentials:
     - Check username/password correct
     - Try in SQL Server Management Studio first
     - If same error in SSMS: Credentials wrong
     - If SSMS works but tool fails: Check tool settings
  
  6. Check firewall rules:
     - Windows Firewall > Allow app through firewall
     - SQL Server should be allowed
     - Or disable firewall temporarily for testing
```

### 8.2 TDE Implementation Issues

**Problem: Phase 1 fails (Backup fails)**
```
Error: "Failed to create backup directory"
Or: "I/O error in database"

Solution:
  1. Check backup path exists:
     - Example: E:\path\TDE_Testing\
     - Should already exist
     - If not: Create manually in Windows Explorer
  
  2. Check write permissions:
     - Right-click folder > Properties > Security
     - Your user account needs "Modify" permission
     - If not: Add permission (may need admin)
  
  3. Check disk space:
     - Required: At least (database size) free space
     - Example: 50GB database needs 50GB+ free
     - Check with: dir E:\ in Command Prompt
  
  4. Check database not in use:
     - Make sure no one is using the database
     - Check Active Sessions:
       SELECT * FROM sys.dm_exec_sessions 
       WHERE database_id = DB_ID('SampleDB')
     - Kill sessions if necessary
  
  5. Retry Phase 1
```

**Problem: Phase 6 fails (Enable TDE fails)**
```
Error: "Cannot enable database encryption because it is already enabled"

Solution:
  - Database is already encrypted
  - This is OK - proceed to Phase 7
  - You can continue with phases 7-8 or skip to monitoring

Error: "Database has changes from previous encryption scans pending log backup"

Solution:
  1. Take transaction log backup:
     BACKUP LOG [SampleDB] TO DISK = 'E:\path\SampleDB.trn'
  2. Wait for backup to complete
  3. Retry Phase 6

Error: "ALTER DATABASE statement failed"

Solution:
  1. Check database status:
     SELECT state_desc FROM sys.databases 
     WHERE name = 'SampleDB'
     -- Should be ONLINE
  
  2. If not online:
     - Something is wrong
     - Contact infrastructure team
     - Do not proceed
  
  3. Check transaction log:
     - Ensure transaction log file exists
     - Has not grown too large (> 50GB concerning)
     - Has sufficient space
  
  4. Retry Phase 6
```

### 8.3 Certificate Issues

**Problem: Check Expiry shows 0 results**
```
Error: "Could not retrieve certificate expiry information"

Solution:
  1. Verify certificate exists:
     SELECT * FROM sys.certificates 
     WHERE name = 'SampleDB_TDE_Cert'
     -- Should return 1 row
  
  2. If no rows: Certificate doesn't exist
     - Run Phase 3 again to create it
  
  3. Check database context:
     - Tool connects to "master" database
     - Certificates always in master
     - Should be correct
  
  4. Check SQL Server connection:
     - Click "Disconnect" then "Connect" again
     - Retry "Check Expiry"
```

**Problem: Renew Certificate fails at Step 4**
```
Error: "Failed to switch DEK"

Solution:
  1. New certificate created but not yet active
  2. Check if certificate was created:
     SELECT * FROM sys.certificates 
     WHERE name = 'SampleDB_TDE_Cert_New'
     -- Should return 1 row
  
  3. If found: Try renewal again
     - System may have been async
     - Next attempt often succeeds
  
  4. If not found: 
     - Try renewal from beginning
     - Check system resources (CPU, memory)
     - Retry
```

### 8.4 Path Issues

**Problem: Paths show forward slashes (E:/path/...) instead of backslashes**
```
Error: File not found when trying to backup

Solution:
  1. This is already fixed in v3.3.1
  2. If you see forward slashes:
     - Download latest version
     - Replace tool file
     - Restart tool
  
  2. Verify paths use backslashes:
     - Should be: E:\path\TDE_Testing\SampleDB_TDE_Cert.cer
     - NOT: E:/path/TDE_Testing/SampleDB_TDE_Cert.cer
```

---

## 9. EMERGENCY PROCEDURES

### 9.1 Disable TDE (Emergency Rollback)

**When to use:**
```
- TDE causing performance issues
- Database not responding
- Need to troubleshoot
- Testing purposes
```

**How to disable:**
```
1. In tool: Click "↩️ Rollback TDE" button (red)
2. Wait for completion
3. Database remains online
4. Encryption disabled
5. Can be re-enabled later

SQL Command (alternative):
ALTER DATABASE SampleDB SET ENCRYPTION OFF
```

**After Rollback:**
```
1. Monitor performance
2. Check database access
3. Verify queries work normally
4. When ready to re-enable:
   - Run TDE Implementation phases 6-8 again
   - Or use tool to re-enable
```

### 9.2 Certificate Expiry Emergency

**Situation: Certificate expired but not renewed**
```
Status: Database uses expired certificate
Risk: Database may become unavailable
Action: Immediate emergency renewal
```

**Emergency Renewal Procedure:**
```
1. Stay calm - data is still encrypted
2. Immediately create new certificate:
   CREATE CERTIFICATE [SampleDB_TDE_Cert_Emergency]
   WITH SUBJECT = 'Emergency Certificate'
   EXPIRY_DATE = DATEADD(YEAR, 1, GETDATE())

3. Backup immediately:
   BACKUP CERTIFICATE [SampleDB_TDE_Cert_Emergency]
   TO FILE = 'E:\emergency_backup.cer'
   WITH PRIVATE KEY (FILE = 'E:\emergency_backup.pvk', 
                     ENCRYPTION BY PASSWORD = 'strong_password')

4. Switch DEK:
   USE SampleDB
   ALTER DATABASE ENCRYPTION KEY 
   ENCRYPTION BY SERVER CERTIFICATE [SampleDB_TDE_Cert_Emergency]

5. Verify switch:
   SELECT * FROM sys.certificates WHERE name = 'SampleDB_TDE_Cert_Emergency'

6. Test database access

7. Contact backup team with certificate details

8. Keep emergency certificate active until proper renewal done

9. Schedule proper renewal immediately
```

### 9.3 Backup File Loss

**Situation: Certificate backup files (.cer and .pvk) deleted or lost**
```
Risk: Cannot restore database on new server
Action: Immediate action needed
```

**Recovery Procedure:**
```
1. If database still online:
   - Backup certificate again immediately
   
   SQL:
   BACKUP CERTIFICATE [SampleDB_TDE_Cert]
   TO FILE = 'E:\emergency_cert_backup.cer'
   WITH PRIVATE KEY (FILE = 'E:\emergency_key_backup.pvk',
                     ENCRYPTION BY PASSWORD = 'strong_password')

2. If database offline:
   - Contact Microsoft Support
   - Provide:
     * SQL Server backup files
     * Service Master Key
     * Master database backup
   - Recovery possible but complex

3. Prevention going forward:
   - Backup certificate files weekly
   - Store off-site
   - Document passwords separately
   - Test restore procedures
```

### 9.4 SQL Server Crash

**Situation: SQL Server went down with encrypted database**
```
Status: Cannot access database
Action: Recovery procedure
```

**Recovery Steps:**
```
1. Restart SQL Server:
   - Windows > Services
   - Find "SQL Server (INSTANCENAME)"
   - Start service
   - Wait for database to come online

2. Verify database is accessible:
   SELECT * FROM sys.databases WHERE name = 'SampleDB'
   -- Check state_desc = 'ONLINE'

3. Test encryption still active:
   SELECT encryption_state FROM sys.dm_database_encryption_keys
   WHERE database_id = DB_ID('SampleDB')
   -- Should show 3 (ENCRYPTED)

4. If encryption_state = 1 or 2:
   - Encryption suspended
   - Need to enable again
   - Database was using encryption, will resume
   - May take time to decrypt/re-encrypt

5. Do NOT shut down SQL Server during encryption changes

6. Monitor progress:
   - Use tool: "8️⃣ Monitor Progress"
   - Or query: 
     SELECT * FROM sys.dm_database_encryption_keys
     WHERE database_id = DB_ID('SampleDB')
     -- Shows current progress
```

---

## 10. APPENDICES

### Appendix A: Useful SQL Queries

**Check Overall Encryption Status:**
```sql
-- Current database encryption status
SELECT 
    db.name AS DatabaseName,
    dek.encryption_state,
    CASE dek.encryption_state
        WHEN 0 THEN 'No Database Encryption Key'
        WHEN 1 THEN 'Unencrypted'
        WHEN 2 THEN 'Encryption in progress'
        WHEN 3 THEN 'Encrypted'
        WHEN 4 THEN 'Key change in progress'
        WHEN 5 THEN 'Decryption in progress'
        WHEN 6 THEN 'Protection change in progress'
    END AS EncryptionStatus,
    c.name AS CertificateName,
    c.expiry_date AS CertificateExpiry,
    DATEDIFF(DAY, GETDATE(), c.expiry_date) AS DaysUntilExpiry
FROM sys.dm_database_encryption_keys dek
INNER JOIN sys.databases db ON dek.database_id = db.database_id
LEFT JOIN sys.certificates c ON dek.encryptor_thumbprint = c.thumbprint
ORDER BY db.name
```

**Find All Certificates:**
```sql
-- All certificates in instance
SELECT 
    name AS CertificateName,
    expiry_date AS ExpiryDate,
    DATEDIFF(DAY, GETDATE(), expiry_date) AS DaysRemaining,
    CASE 
        WHEN DATEDIFF(DAY, GETDATE(), expiry_date) < 0 THEN 'EXPIRED'
        WHEN DATEDIFF(DAY, GETDATE(), expiry_date) < 30 THEN 'URGENT'
        WHEN DATEDIFF(DAY, GETDATE(), expiry_date) < 90 THEN 'WARNING'
        ELSE 'OK'
    END AS Status
FROM sys.certificates
WHERE name LIKE '%TDE%'
ORDER BY expiry_date
```

**Monitor Encryption Progress:**
```sql
-- Encryption progress by database
SELECT 
    db.name AS DatabaseName,
    dek.encryption_state,
    CASE dek.encryption_state
        WHEN 3 THEN 'Encrypted'
        ELSE 'Not Encrypted'
    END AS Status,
    (SELECT SUM(size * 8 / 1024 / 1024) FROM sys.master_files 
     WHERE database_id = db.database_id) AS SizeMB
FROM sys.dm_database_encryption_keys dek
INNER JOIN sys.databases db ON dek.database_id = db.database_id
ORDER BY db.name
```

**Check Active Sessions:**
```sql
-- Who is connected to database
SELECT 
    session_id,
    login_name,
    database_id,
    DB_NAME(database_id) AS DatabaseName,
    status,
    connect_time,
    last_request_start_time
FROM sys.dm_exec_sessions
WHERE DB_ID('SampleDB') IS NOT NULL
ORDER BY connect_time
```

**Backup Status:**
```sql
-- Recent backups
SELECT 
    database_name,
    type,
    backup_start_date,
    backup_finish_date,
    expiration_date,
    backup_size
FROM msdb.dbo.backupset
WHERE database_name = 'SampleDB'
ORDER BY backup_start_date DESC
LIMIT 10
```

### Appendix B: Tool Keyboard Shortcuts

**Coming in future versions - currently use mouse/clicking**

### Appendix C: Settings File (tde_tool_settings.json)

**Location:**
```
C:\TDE_Tool\tde_tool_settings.json
```

**Format:**
```json
{
    "server": "PHSQLDBA",
    "database": "master",
    "username": "",
    "auth_type": "Windows Authentication"
}
```

**What's Saved:**
```
- Server name
- Default database
- Username (if SQL Auth)
- Authentication type
- NOT passwords (for security)
```

**How to Reset:**
```
1. Delete tde_tool_settings.json
2. Restart tool
3. Settings reset to defaults
```

### Appendix D: Troubleshooting Checklist

**Before contacting support:**
```
□ Tool version: 3.3.1
□ Python version: (python --version)
□ ODBC Driver version: 18+
□ SQL Server version: (SELECT @@VERSION)
□ Database affected: (specify name)
□ Error message: (full error text)
□ When did it start: (specific time)
□ What were you doing: (specific operation)
□ Have you tried: (connection reset, restart tool)
□ Recent changes: (any server changes recently)
□ Backup status: (is backup available)
□ Certificate status: (CHECK EXPIRY)
```

### Appendix E: Contact & Support

**For Issues:**
```
1. Check Troubleshooting section (above)
2. Review logs in C:\TDE_Tool\logs\
3. Contact DBA team with:
   - Error message
   - Database name
   - Checklist above completed
   - Proposed timeline

Emergency:
   - Contact Infrastructure on-call DBA
   - Have database backups verified
   - Have certificate backups available
```

### Appendix F: Training Checklist

**For New DBAs:**
```
□ Understand what TDE does
□ Review this entire SOP
□ Watch tool demonstration
□ Practice on test database
□ Implement TDE on test database (all 8 phases)
□ Monitor encryption progress
□ Test certificate renewal
□ Learn troubleshooting procedures
□ Understand emergency procedures
□ Sign off on competency
```

---

## SIGN-OFF

**Document Approval:**

| Role | Name | Date | Signature |
|------|------|------|-----------|
| DBA Lead | | | |
| Infrastructure Manager | | | |
| Security Officer | | | |
| Database Owner | | | |

---

## VERSION HISTORY

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2026-07-18 | Initial release | Praveen Madupu |

---

**END OF DOCUMENT**

---

*This SOP is a living document. Update as needed with operational experience. Distribute to all DBAs with encryption responsibilities.*

