# SQL Server 2025 TDE Automation GUI Tool - Setup & User Guide

## Overview

Professional GUI automation tool for SQL Server 2025 Transparent Data Encryption (TDE) with real-time parameters and full operational control.

**Version:** 1.0  
**Author:** Praveen Madupu (SQL DBA Champs)  
**Status:** Production Ready

---

## System Requirements

### Operating System
- Windows Server 2019 or later
- Windows 10 / 11 (for client deployment)

### Software Prerequisites
- Python 3.8 or higher
- SQL Server 2025 (Enterprise or Standard Edition)
- SQL Server Native Client ODBC Driver 17

### Network Requirements
- Network connectivity to target SQL Server instance
- Port 1433 (default) open for SQL communication
- Administrative privileges on SQL Server

---

## Installation Guide

### Step 1: Install Python Dependencies

```bash
# Navigate to application directory
cd C:\path\to\tde_automation_gui

# Install required packages
pip install -r requirements.txt
```

### Step 2: Install ODBC Driver (if not present)

```bash
# Download from Microsoft: Microsoft ODBC Driver 17 for SQL Server
# Or use chocolatey:
choco install odbc17

# Or download from:
# https://www.microsoft.com/en-us/download/details.aspx?id=56567
```

### Step 3: Verify Installation

```bash
python tde_automation_gui.py --version
```

### Step 4: Launch Application

```bash
python tde_automation_gui.py
```

Or create a batch file:

```batch
@echo off
python C:\path\to\tde_automation_gui.py
pause
```

---

## Application Features

### Tab 1: Connection
**Purpose:** Establish and manage SQL Server connections

**Parameters:**
- **SQL Server:** Server name or IP address (e.g., SERVERNAME or 192.168.1.100)
- **Database:** Target database (default: master)
- **Username:** SQL Server login (e.g., sa, domain\user)
- **Password:** SQL Server password (encrypted)

**Operations:**
- ✓ Connect to SQL Server
- ✓ Test Connection (Ping)
- ✓ Display Server Information
- ✓ Disconnect

**Real-Time Status Indicators:**
- Connection state (Connected/Not Connected)
- Color-coded status (Green/Red)
- Server version and edition
- Current database info

---

### Tab 2: Implementation
**Purpose:** Execute TDE implementation in 4 phases

#### Phase 1: Pre-Implementation Backup
- **Parameters:**
  - Target Database
  - Backup Path
  - Backup Type (Copy-Only)
  - Compression (ON/OFF)
  
- **Operations:**
  - Create copy-only full backup
  - Verify backup integrity
  - Auto-validate before proceeding

#### Phase 2: Master Key & Certificate Setup
- **Parameters:**
  - Master Key Password (secure input)
  - Certificate Name Prefix
  - Certificate Backup Path
  - Private Key Password (secure input)
  
- **Operations:**
  - Create Database Master Key
  - Create TDE Server Certificate
  - Backup Certificate & Private Key
  - Auto-generate dated names

#### Phase 3: Enable TDE Encryption
- **Parameters:**
  - Target Database
  - Certificate Name
  
- **Operations:**
  - Create Database Encryption Key (AES-256)
  - Enable TDE on Database
  - Start async encryption scan
  - Confirmation dialogs for critical operations

#### Phase 4: Validation & Monitoring
- **Parameters:**
  - Database to Validate
  
- **Operations:**
  - Verify encryption status
  - Check encryption progress
  - Display encryption metrics
  - Confirm successful implementation

---

### Tab 3: Monitoring
**Purpose:** Real-time TDE monitoring and diagnostics

**Parameters:**
- **Database to Monitor:** Target database name
- **Check Interval:** 10-300 seconds (default: 30)

**Operations:**
- **Verify TDE Status:** One-time status check
  - Encryption state (0-6 levels)
  - Progress percentage
  - Certificate expiry dates
  - Days until renewal

- **Monitor Encryption Progress:** Continuous monitoring
  - Automatic refresh at set intervals
  - Progress bar and percentage
  - Scan state tracking
  - Estimated completion time

**Monitoring Table:**
| Column | Description |
|--------|-------------|
| Database | Database name |
| State | Encryption state (Encrypted/In Progress) |
| Progress % | Encryption completion percentage |
| Scan Status | Scan status (Scanning/Complete) |
| Certificate | TDE certificate name |
| Days Until Expiry | Days remaining before cert expires |

**Progress Log:**
- Timestamped entries
- Real-time updates
- Detailed status messages
- Error logging

---

### Tab 4: Maintenance
**Purpose:** Certificate lifecycle management and renewal

#### Certificate Expiry Check
- **Operations:**
  - Check all TDE certificates
  - Display expiry status
  - Color-coded alerts (🔴 Red: Immediate action required)
  - Days remaining calculation

#### Certificate Renewal
- **Parameters:**
  - Old Certificate Name
  - New Certificate Name
  - Target Database
  - New Expiry Date (date picker)
  
- **8-Step Renewal Process:**
  1. Backup existing certificate
  2. Create new certificate
  3. Backup new certificate
  4. Re-protect DEK with new certificate
  5. Validate thumbprint match
  6. Test restore (optional)
  7. Keep old certificate (recommended period)
  8. Remove old certificate (optional)

**Installed Certificates:**
- Live list of all TDE certificates
- Status indicators per certificate
- Expiry information
- Click-to-manage functionality

---

### Tab 5: Rollback
**Purpose:** Emergency procedures for encryption reversal

**⚠️ WARNING:** Critical operations with explicit confirmations required

#### Encryption Rollback
- **Parameters:**
  - Target Database
  
- **Operations:**
  - Cancel ongoing encryption
  - Initiate decryption process
  - Monitor decryption progress
  - Drop DEK after completion
  
- **Safety Features:**
  - Mandatory confirmation dialog
  - Warning message display
  - Operation logging
  - Rollback status tracking

#### Restore from Pre-TDE Backup
- **Parameters:**
  - Database Name
  - Backup File Path (with browse dialog)
  
- **Operations:**
  - Take database offline
  - Execute full restore
  - Drop TDE components
  - Bring database online
  
- **Safety Features:**
  - Triple confirmation required
  - Explicit data loss warning
  - Backup validation
  - Pre-restore checks

**Rollback Log:**
- All rollback operations logged
- Timestamp tracking
- Success/failure status
- Error details and recovery steps

---

### Tab 6: Scripts
**Purpose:** Pre-built T-SQL script library

**Available Scripts:**
1. **Full Implementation (All Phases)**
   - Complete 4-phase implementation
   - ~300 lines of T-SQL
   - All error handling included

2. **Phase 1: Backup Only**
   - Copy-only backup with verification
   - Compression enabled

3. **Phase 2: Certificate Setup**
   - Master key creation
   - Certificate generation
   - Certificate backup

4. **Phase 3: Enable TDE**
   - DEK creation
   - TDE enablement
   - Async scan initiation

5. **Phase 4: Validation**
   - Status verification
   - Progress monitoring
   - Certificate validation

6. **Monitoring Queries**
   - TDE status checks
   - Certificate expiry monitoring
   - Blocking session detection
   - Performance diagnostics

7. **Certificate Renewal**
   - New certificate creation
   - DEK re-protection
   - Validation queries

8. **Rollback Procedures**
   - Encryption cancellation
   - Decryption monitoring
   - Database restoration
   - Cleanup procedures

9. **Troubleshooting**
   - Common issue detection
   - Diagnostic queries
   - Resolution steps
   - Validation scripts

**Script Operations:**
- **Copy to Clipboard:** One-click copy for SSMS execution
- **Save Script:** Export to .sql file for version control
- **Font:** Courier New, 9pt (optimized for readability)

---

## Real-Time Parameters Reference

### Connection Parameters
| Parameter | Type | Default | Range | Required |
|-----------|------|---------|-------|----------|
| SQL Server | String | localhost | Any hostname/IP | Yes |
| Database | String | master | Any database | Yes |
| Username | String | sa | Any login | Yes |
| Password | Password | (empty) | N/A | Yes |

### Implementation Parameters
| Parameter | Type | Default | Range | Required |
|-----------|------|---------|-------|----------|
| Target Database | String | BETA | DB_NAME | Yes |
| Backup Path | String | E:\SQLBackups\ | Valid path | Yes |
| Master Key Password | Password | (shown) | Min 12 chars | Yes |
| Certificate Name | String | SQLDBAChamps_TDE_Cert_ | Custom | Yes |
| Private Key Password | Password | (shown) | Min 12 chars | Yes |

### Monitoring Parameters
| Parameter | Type | Default | Range | Required |
|-----------|------|---------|-------|----------|
| Database | String | BETA | DB_NAME | Yes |
| Check Interval | Integer | 30 | 10-300 seconds | No |

### Maintenance Parameters
| Parameter | Type | Default | Range | Required |
|-----------|------|---------|-------|----------|
| Old Certificate Name | String | SQLDBAChamps_TDE_Cert_ | Certificate name | Yes |
| New Certificate Name | String | SQLDBAChamps_TDE_Cert_2027_ | Certificate name | Yes |
| Target Database | String | BETA | DB_NAME | Yes |
| New Expiry Date | Date | Current date + 2 years | Any future date | Yes |

### Rollback Parameters
| Parameter | Type | Default | Range | Required |
|-----------|------|---------|-------|----------|
| Database | String | BETA | DB_NAME | Yes |
| Backup Path | String | (empty) | Valid .bak file | Yes (for restore) |
| Confirmation | Checkbox | Unchecked | True/False | Yes (for restore) |

---

## Operation Workflows

### Complete TDE Implementation (Step-by-Step)

1. **Connection Tab**
   - Enter SQL Server credentials
   - Click "Connect"
   - Verify "Connected ✓" status

2. **Implementation Tab → Phase 1**
   - Review database and backup path
   - Click "Execute Backup"
   - Wait for backup verification

3. **Implementation Tab → Phase 2**
   - Review Master Key password
   - Click "Create Master Key"
   - Click "Create Certificate"
   - Click "Backup Certificate & Key"

4. **Implementation Tab → Phase 3**
   - Verify database name
   - Click "Create Database Encryption Key"
   - Click "Enable TDE (Start Encryption Scan)"
   - Read confirmation and proceed

5. **Monitoring Tab**
   - Click "Verify TDE Status"
   - Click "Monitor Encryption Progress" (for async monitoring)
   - Wait until encryption_state = 3 (100% complete)

6. **Implementation Tab → Phase 4**
   - Click "Verify TDE Status"
   - Confirm successful encryption
   - Implementation complete ✓

---

### Certificate Renewal (8-Step Process)

1. **Maintenance Tab → Certificate Check**
   - Click "Check Certificate Expiry"
   - Identify certificate(s) to renew

2. **Maintenance Tab → Renewal**
   - Enter old and new certificate names
   - Set new expiry date (2 years out)
   - Click "Renew Certificate"
   - Observe step-by-step progress

3. **Monitoring Tab**
   - Verify encryption is using new certificate
   - Confirm thumbprint matches

4. **Maintenance Tab**
   - Keep old certificate for backup retention period
   - Monitor expiry dates

---

### Emergency Rollback

1. **Rollback Tab**
   - Read warning message carefully
   - If canceling encryption:
     - Check "Cancel ongoing encryption"
     - Click "Cancel Encryption & Start Decryption"
     - Monitor decryption in logs

2. **Rollback Tab → Restore**
   - Check "I confirm..." checkbox
   - Browse to pre-TDE backup file
   - Click "Restore Database from Backup"
   - Wait for restore completion

3. **Connection Tab**
   - Re-connect to verify database online
   - Confirm TDE is disabled

---

## Logging & Audit

### Operation Logs
- **Location:** Console output + application logs
- **Format:** Timestamped entries with [HH:MM:SS]
- **Retention:** Session-based (cleared on restart)
- **Export:** Copy from output pane or save via right-click

### Log Levels
- ✓ Success messages (green)
- ⚠️ Warning messages (orange)
- ❌ Error messages (red)
- ℹ️ Information messages (blue)

### Audit Information Captured
- Connection attempts (success/failure)
- All SQL operations executed
- Operation start/end times
- Success/failure status
- Error messages and stack traces
- Certificate details
- Encryption progress percentages

---

## Troubleshooting

### Connection Issues

**Problem:** "Connection refused"
- **Solution:** Verify SQL Server is running and accessible
- **Check:** `ping <servername>` and telnet to port 1433
- **ODBC:** Install Microsoft ODBC Driver 17

**Problem:** "Login failed for user"
- **Solution:** Verify credentials (case-sensitive password)
- **Check:** Test login in SQL Server Management Studio first
- **Alternative:** Use Windows authentication (domain\user)

### Operation Issues

**Problem:** "Backup verification failed"
- **Solution:** Verify backup path exists and is accessible
- **Check:** File permissions and disk space
- **Retry:** Ensure database is online

**Problem:** "Encryption stuck at X%"
- **Solution:** Check blocking sessions in Monitoring tab
- **Check:** Long-running transactions
- **Wait:** Large databases can take hours
- **Log:** Review SQL Server error log

**Problem:** "Certificate not found"
- **Solution:** Run "Check Certificate Expiry" to list available certs
- **Check:** Correct certificate name in parameters
- **Create:** Generate new certificate if missing

### GUI Issues

**Problem:** Application crashes
- **Solution:** Check Python version (3.8+ required)
- **Reinstall:** `pip install -r requirements.txt --force-reinstall`
- **Clean:** Delete `tde_tool_settings.json` and restart

**Problem:** Script display is blank
- **Solution:** Ensure "Scripts" tab is fully loaded
- **Refresh:** Click script selector dropdown again
- **Restart:** Close and reopen application

---

## Best Practices

### Pre-Implementation
- ☑️ Schedule maintenance window
- ☑️ Backup database to pre-TDE backup
- ☑️ Disable auto-failover (if cluster)
- ☑️ Notify users of maintenance window
- ☑️ Test scripts in non-prod first

### During Implementation
- ☑️ Monitor encryption progress constantly
- ☑️ Keep application window open
- ☑️ Do NOT restart SQL Server
- ☑️ Do NOT modify database
- ☑️ Keep backup files secure

### Post-Implementation
- ☑️ Verify encryption_state = 3
- ☑️ Test database restores
- ☑️ Update documentation
- ☑️ Set certificate renewal reminder
- ☑️ Archive logs and scripts

### Certificate Management
- ☑️ Renew 30 days before expiry
- ☑️ Keep old certificate 1 year (or backup retention period)
- ☑️ Store backups in secure vault (HSM)
- ☑️ Test certificate restore quarterly
- ☑️ Monitor expiry dates monthly

### Security
- ☑️ Use strong passwords (12+ chars, special chars)
- ☑️ Secure certificate files (encrypt at rest)
- ☑️ Audit certificate access
- ☑️ Restrict GUI tool access (network/firewall)
- ☑️ Use Windows Authentication when possible

---

## Support & Documentation

### Resources
- **SQL DBA Champs:** sqldbachamps.com
- **Microsoft Docs:** docs.microsoft.com/sql/relational-databases/security/encryption/transparent-data-encryption
- **Deployment:** See included 32-page SOP document
- **Scripts:** Pre-built in Scripts tab

### Getting Help
1. Review logs in application output panes
2. Check "Scripts" tab for solutions
3. Refer to "Troubleshooting & Diagnostics" script
4. Consult included SOP documentation
5. Contact: Praveen Madupu (SQL DBA Champs)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Jan 2025 | Initial release - Complete feature set |

---

## License & Attribution

**SQL Server 2025 TDE Automation GUI Tool**  
**Author:** Praveen Madupu (SQL DBA Champs)  
**Version:** 1.0  
**Status:** Production Ready  
**© 2025 SQL DBA Champs. All rights reserved.**

---

**Last Updated:** January 2025  
**Document Version:** 1.0
