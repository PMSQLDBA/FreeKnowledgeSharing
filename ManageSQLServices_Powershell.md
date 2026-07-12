# Technical SOP: Managing SQL Server Services via PowerShell

## Overview

This Standard Operating Procedure (SOP) outlines the standardized method for safely starting and stopping multiple SQL Server instances and their associated SQL Server Agent services. 
Using this script ensures that services are handled in the correct order to maintain system stability.

## Scope

This procedure applies to all staff managing local workstations that host multiple SQL Server instances.

## Prerequisites

* **Permissions:** You must run the PowerShell environment as an **Administrator**.
* **Script Content:** Ensure the `SQL_Service_Control.ps1` file contains the logic provided below.

---

## The PowerShell Script (`SQL_Service_Control.ps1`)

Save the following code into a file named `SQL_Service_Control.ps1`:

```powershell
param(
    [Parameter(Mandatory=$true)]
    [ValidateSet("Start","Stop")]
    [string]$Action
)

# Helper function to process services
function Invoke-SQLServiceAction {
    param([string]$Action, [array]$Services)
    foreach ($svc in $Services) {
        try {
            if ($Action -eq "Stop" -and $svc.Status -eq "Running") {
                Stop-Service -Name $svc.Name -Force -ErrorAction Stop
                Write-Host "Stopped: $($svc.Name)" -ForegroundColor Yellow
            }
            elseif ($Action -eq "Start" -and $svc.Status -ne "Running") {
                Start-Service -Name $svc.Name -ErrorAction Stop
                Write-Host "Started: $($svc.Name)" -ForegroundColor Green
            }
            else {
                Write-Host "$($svc.Name) is already in the target state." -ForegroundColor Gray
            }
        }
        catch {
            Write-Error "Failed to $Action $($svc.Name): $($_.Exception.Message)"
        }
    }
}

# Identify services
$SQLAgent = Get-Service | Where-Object { $_.Name -like "SQLSERVERAGENT*" }
$SQLDatabaseEngine = Get-Service | Where-Object { $_.Name -like "MSSQL$*" -or $_.Name -eq "MSSQLSERVER" }

# Execute based on action
if ($Action -eq "Stop") {
    Write-Host "--- Stopping SQL Services ---" -ForegroundColor Cyan
    Invoke-SQLServiceAction -Action "Stop" -Services $SQLAgent
    Invoke-SQLServiceAction -Action "Stop" -Services $SQLDatabaseEngine
}
else {
    Write-Host "--- Starting SQL Services ---" -ForegroundColor Cyan
    Invoke-SQLServiceAction -Action "Start" -Services $SQLDatabaseEngine
    Invoke-SQLServiceAction -Action "Start" -Services $SQLAgent
}

Write-Host "Operation Completed." -ForegroundColor Green

```

---

## Procedure

### 1. Handling Execution Policy

If you receive a security error, bypass the restriction for the current terminal process by running:

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

```

### 2. Executing the Script

Navigate to the directory containing the script and execute the desired action.

* **To Stop All SQL Services:**
```powershell
.\SQL_Service_Control.ps1 Stop

```

* *Note:* The script stops SQL Server Agent services first, followed by the SQL Database Engine to ensure a clean shutdown.

* **To Start All SQL Services:**
```powershell
.\SQL_Service_Control.ps1 Start

```

* *Note:* The script initiates the SQL Database Engine first, followed by the SQL Server Agent.


*For further information on PowerShell security, refer to the [Microsoft documentation on Execution Policies](https://go.microsoft.com/fwlink/?LinkID=135170).*

Would you like me to include instructions on how to create a desktop shortcut that automatically runs this script with the correct parameters?
