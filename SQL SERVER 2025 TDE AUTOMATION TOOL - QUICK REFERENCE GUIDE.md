╔════════════════════════════════════════════════════════════════════════════╗
║                                                                            ║
║     SQL SERVER 2025 TDE AUTOMATION TOOL - QUICK REFERENCE GUIDE            ║
║                                                                            ║
║                  Complete End-to-End SOP - One Page                        ║
║                                                                            ║
╚════════════════════════════════════════════════════════════════════════════╝

INSTALLATION (First Time Only)
════════════════════════════════════════════════════════════════════════════

1. Install Python 3.11+
   Download: https://www.python.org/downloads/
   Run installer, check "Add Python to PATH"

2. Install ODBC Driver 18
   Download: Microsoft ODBC Driver 18 for SQL Server
   Run installer

3. Install Python Packages
   pip install PyQt5==5.15.9 pyodbc==5.1.1 pyperclip==1.8.2

4. Copy Tool
   Copy: tde_automation_gui_v3_1_final.py to C:\TDE_Tool\
   Create shortcut on desktop

5. Launch Tool
   Double-click shortcut
   Tool window opens (1600x950)

════════════════════════════════════════════════════════════════════════════

STEP-BY-STEP WORKFLOW
════════════════════════════════════════════════════════════════════════════

BEFORE YOU START
┌─────────────────────────────────────────────────────────────────────────┐
│ □ Database is online and accessible                                     │
│ □ Database is not encrypted yet (or you know this)                      │
│ □ Have backup location ready: E:\path\TDE_Testing\                      │
│ □ Have strong passwords ready (Master Key + Cert Key)                   │
│ □ Have admin permissions on SQL Server                                  │
│ □ Scheduled maintenance window                                          │
│ □ Notified users of brief 2-3 second downtime (at step 4)               │
└─────────────────────────────────────────────────────────────────────────┘

STEP 1: LAUNCH & CONNECT
────────────────────────────────────────────────────────────────────────────
1. Double-click "SQL Server TDE Automation Tool" shortcut
2. Window opens: "Status: Not Connected"
3. Fill Connection Details:
   - Server: PHSQLDBA
   - Database: master
   - Auth Type: Windows Authentication (or SQL Server Auth)
4. Click "✓ Connect" button
5. Wait for: "✓ Connected to PHSQLDBA.master"

STEP 2: PREPARE PARAMETERS
────────────────────────────────────────────────────────────────────────────
1. Fill "Operation Parameters (Auto-Generated)" section:
   - Target Database: SampleDB
   - Backup Path: E:\TDE_Testing\SampleDB.bak
   - Master Key Password: (strong password)
   - Key Encryption Password: (strong password)
2. Click "Auto-Generate Parameters"
3. Verify:
   - Certificate Name: SampleDB_TDE_Cert
   - Cert Backup: E:\TDE_Testing\SampleDB_TDE_Cert.cer
   - Key Backup: E:\TDE_Testing\SampleDB_TDE_Key.pvk
   - All paths use backslashes (\) not forward slashes (/)

STEP 3: EXECUTE 8-PHASE TDE IMPLEMENTATION
────────────────────────────────────────────────────────────────────────────

Phase 1: 1️⃣ Backup Database
  • Click button
  • Wait for: "✓ Backup completed successfully"
  • Time: 1-5 minutes (depends on database size)

Phase 2: 2️⃣ Create Master Key
  • Click button
  • Expected: "✓ Master Key created" OR "✓ Already exists (skipped)"
  • Time: <1 second

Phase 3: 3️⃣ Create Certificate
  • Click button
  • Expected: "✓ Certificate created" OR "✓ Already exists (skipped)"
  • Time: <1 second

Phase 4: 4️⃣ Backup Certificate
  • Click button
  • Expected: "✓ Certificate backed up to:"
  • Shows: .cer and .pvk file paths
  • IMPORTANT: Save these backup files securely!
  • Time: <1 second

Phase 5: 5️⃣ Create DEK
  • Click button
  • Expected: "✓ DEK created" OR "✓ Already exists (skipped)"
  • Time: <1 second

Phase 6: 6️⃣ Enable TDE
  • Click button
  • Expected: "✓ TDE enabled"
  • If error "Already enabled": Database already encrypted - OK, continue
  • If error "Pending log backup": Run BACKUP LOG and retry
  • Time: <1 second (but may need log backup first)

Phase 7: 7️⃣ Verify Status
  • Click button
  • Expected: "✓ Status retrieved successfully"
  • Should show: "Encryption Status: Encrypted"
  • Time: <1 second

Phase 8: 8️⃣ Monitor Progress
  • Click button (optional - encryption likely already complete)
  • Shows progress every 5 seconds
  • Exit when Progress shows 100% or 0.0% (complete)
  • Time: 5 seconds to 1 hour (depending on size)

STEP 4: VERIFY COMPLETION
────────────────────────────────────────────────────────────────────────────
1. Click "📊 Get Status" in Maintenance section
2. Output should show:
   ✓ Database: SampleDB
   ✓ Status: ENCRYPTED
   ✓ Certificate: SampleDB_TDE_Cert
3. Test database access:
   - Open SQL Server Management Studio
   - Connect to database
   - Run: SELECT * FROM sys.tables
   - Should work normally
4. Database is now encrypted! ✅

════════════════════════════════════════════════════════════════════════════

ONGOING MAINTENANCE - DO THIS REGULARLY
════════════════════════════════════════════════════════════════════════════

MONTHLY (First business day)
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Click "📅 Check Expiry" button                                      │
│ 2. Note: Days remaining                                                 │
│    ✅ OK (> 90 days): Continue monitoring                              │
│    🟡 WARNING (30-90): Start planning renewal                          │
│    🔴 URGENT (< 30): Schedule renewal immediately                      │ 
│ 3. Record in spreadsheet                                               │
│ 4. No action needed unless status changes                              │
└────────────────────────────────────────────────────────────────────────┘

WHEN CERTIFICATE HAS < 30 DAYS LEFT
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Click "📅 Check Expiry" - Verify < 30 days                           │
│ 2. Schedule maintenance window (during off-hours)                       │
│ 3. Notify users of brief downtime (2-3 seconds during Step 4)           │
│ 4. Click "🔄 Renew Cert" (green button)                                │
│ 5. Watch output - should complete in 15-30 seconds                      │
│ 6. Output should show:                                                  │
│    ✓ Step 1: Verified ✓                                                │
│    ✓ Step 2: Certificate created ✓                                     │
│    ✓ Step 3: Backed up ✓                                               │
│    ✓ Step 4: DEK switched ✓ (Users experience brief pause here)       │
│    ✓ Step 5: Verified ✓                                               │
│    ✓ Summary: RENEWED & ACTIVE ✓                                      │
│ 7. New certificate active, next renewal in 1 year                      │
│ 8. Database remains encrypted throughout - no data loss                │
└─────────────────────────────────────────────────────────────────────────┘

════════════════════════════════════════════════════════════════════════════

CERTIFICATE RENEWAL TIMELINE (Example)
════════════════════════════════════════════════════════════════════════════

Certificate Created: July 18, 2026 (expires July 18, 2027)

Timeline:
  January 2027:    📅 Check Expiry → ~180 days → Status OK ✓
  April 2027:      📅 Check Expiry → ~90 days → Status WARNING 🟡
                   Start planning renewal
  June 2027:       📅 Check Expiry → ~30 days → Status URGENT 🔴
                   Schedule maintenance window
  July 15-17, 2027: 🔄 Renew Cert (off-peak hours)
                   Certificate renewed
  July 18, 2027:   DEADLINE - Certificate expires
                   (Should be renewed by this date)
  July 18, 2028:   New certificate expires
                   Repeat cycle

════════════════════════════════════════════════════════════════════════════

EMERGENCY PROCEDURES
════════════════════════════════════════════════════════════════════════════

DISABLE TDE (Emergency)
  • Click "↩️ Rollback TDE" (red button)
  • Database remains online
  • Encryption disabled
  • Use only in emergency
  • Can be re-enabled later

CERTIFICATE EXPIRED
  • Do NOT let this happen! Monitor monthly
  • If it happens: Create new certificate immediately
  • Contact Microsoft Support if having issues

════════════════════════════════════════════════════════════════════════════

USEFUL SQL QUERIES
════════════════════════════════════════════════════════════════════════════

Check Encryption Status:
  SELECT db.name, dek.encryption_state, c.name, c.expiry_date
  FROM sys.dm_database_encryption_keys dek
  JOIN sys.databases db ON dek.database_id = db.database_id
  LEFT JOIN sys.certificates c ON dek.encryptor_thumbprint = c.thumbprint
  WHERE db.name = 'SampleDB'

List All Certificates:
  SELECT name, expiry_date, DATEDIFF(DAY, GETDATE(), expiry_date) AS DaysLeft
  FROM sys.certificates
  WHERE name LIKE '%TDE%'
  ORDER BY expiry_date

════════════════════════════════════════════════════════════════════════════

TROUBLESHOOTING
════════════════════════════════════════════════════════════════════════════

Cannot Connect:
  ✓ Check server name (verify in SQL Server Management Studio)
  ✓ Check SQL Server is running (Services > SQL Server)
  ✓ Check network connectivity (ping server)
  ✓ Check ODBC Driver installed (Control Panel > ODBC)
  ✓ Check credentials (try in SSMS first)

Phase Fails:
  ✓ Check database is online
  ✓ Check backup path exists and is writable
  ✓ Check disk space available
  ✓ Retry the phase
  ✓ Check error message for details

Check Expiry Returns Nothing:
  ✓ Verify certificate exists: 
    SELECT * FROM sys.certificates WHERE name = 'SampleDB_TDE_Cert'
  ✓ Disconnect then reconnect tool
  ✓ Retry Check Expiry

Renew Certificate Fails:
  ✓ Most common: Needs retry (async operation)
  ✓ Verify new certificate created:
    SELECT * FROM sys.certificates WHERE name = 'SampleDB_TDE_Cert_New'
  ✓ If exists: Retry renewal (usually works second time)
  ✓ If not exists: Check error message, fix issue, retry

════════════════════════════════════════════════════════════════════════════

KEYBOARD SHORTCUTS
════════════════════════════════════════════════════════════════════════════

Coming in future versions - currently use mouse to click buttons

════════════════════════════════════════════════════════════════════════════

IMPORTANT REMINDERS
════════════════════════════════════════════════════════════════════════════

✓ BACKUP CERTIFICATE FILES SECURELY
  - Save .cer and .pvk files off-site
  - Cannot restore database without these
  - Test restore procedures regularly

✓ DOCUMENT PASSWORD
  - Keep Master Key and Key passwords secure
  - Use password manager
  - DO NOT commit to code repository

✓ MONITOR MONTHLY
  - Set calendar reminder
  - Click "📅 Check Expiry" monthly
  - Record results
  - Plan renewal 90 days before expiry

✓ TEST RESTORE
  - Monthly: Test database backup restore
  - Quarterly: Test certificate restore
  - Ensures you can recover if needed
  - Don't wait until emergency!

✓ NOTIFY USERS
  - Before maintenance: Send email
  - During maintenance: Brief 2-3 second downtime
  - After maintenance: Confirmation email

✓ KEEP DOCUMENTATION
  - Record all renewals
  - Note certificate names and dates
  - Update renewal calendar annually
  - Keep runbook current

════════════════════════════════════════════════════════════════════════════

CONTACT & SUPPORT
════════════════════════════════════════════════════════════════════════════

For Issues:
  1. Check Troubleshooting section above
  2. Review complete SOP: TDE_AUTOMATION_TOOL_COMPLETE_SOP.md
  3. Contact DBA team with error message and database name

Tool Version: 3.3.1
Documentation Version: 1.0
Last Updated: July 18, 2026

════════════════════════════════════════════════════════════════════════════

That's it! You're ready to encrypt databases! 🔐

For detailed procedures, see the complete SOP document.

════════════════════════════════════════════════════════════════════════════
