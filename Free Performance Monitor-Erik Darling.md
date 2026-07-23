Source: 

https://www.linkedin.com/posts/erik-darling-data_sqlserver-dba-monitoring-share-7485695308400738304-_w-o/?utm_source=share&utm_medium=member_desktop&rcm=ACoAACKCDlIBwCyHwl-zdzyASa1BIP491KNZCLE

SQL Server Performance Monitor v3.2.0 is out. This is the release where Darling grows up.

Darling is the new flagship edition. 
It's a free, headless monitoring service: install it once, point it at your fleet, and it collects around the clock into a bundled, managed Postgres/TimescaleDB store. 
Nothing gets installed on your SQL Servers for its storage. It scales from 5 servers to 500.

What you get:

• A web dashboard and a desktop viewer

• Custom Views you compose yourself, plus investigation notebooks

• An MCP server your AI agent can drive: build views, tune alerts, and onboard whole fleets from a conversation

• Connection-loss alerts, bulk server onboarding, and 9 new collectors shared across both editions


Darling replaces the Full Dashboard as the flagship. Still free. Still open source. Your data never leaves your network.

Github Repo: https://github.com/erikdarlingdata/PerformanceMonitor/releases/tag/v3.2.0

Owner: Erik Darling

# What is Performance Monitor Darling?

Think of it as a **free alternative** to products like:

* Redgate SQL Monitor
* SentryOne SQL Sentry
* SolarWinds DPA
* Idera SQL Diagnostic Manager

The major difference is:

* Free
* Open Source
* No SQL monitoring database installed on monitored SQL Servers
* Centralized monitoring
* AI/MCP integration
* Web dashboard
* Desktop viewer
* PostgreSQL/TimescaleDB backend

Your monitored SQL Servers remain almost untouched except for read-only monitoring queries and Extended Events where applicable. ([GitHub][2])

---

# Overall Architecture

```
                    +----------------+
                    | Web Dashboard  |
                    +-------+--------+
                            |
                    +-------v--------+
                    | Desktop Viewer |
                    +-------+--------+
                            |
                    +-------v--------+
                    | Darling Service|
                    | Windows Service|
                    +-------+--------+
                            |
         -----------------------------------------
         |                  |                    |
         |                  |                    |
 +-------v------+   +-------v------+    +--------v------+
 | SQL Server 1 |   | SQL Server 2 |    | SQL Server N  |
 +--------------+   +--------------+    +---------------+

                |
                |
        PostgreSQL + TimescaleDB
        (Installed automatically)
```

---

# Step 1 — Download

Download the latest Darling release:

**[PerformanceMonitor Releases](https://github.com/erikdarlingdata/PerformanceMonitor/releases?utm_source=chatgpt.com)**

Download:

```
PerformanceMonitorDarling-3.2.0.zip
```

Do **not** download Dashboard unless you have an existing deployment.

---

# Step 2 — Choose a Monitoring Server

Create a dedicated monitoring VM.

Example:

```
MONITOR01

Windows Server 2022

16 GB RAM

8 vCPU

200 GB SSD

```

This server will host:

* Darling Service
* PostgreSQL
* TimescaleDB
* Web Dashboard
* Desktop Viewer

Not your production SQL Servers.

---

# Step 3 — Extract

Example

```
D:\PerformanceMonitor\
```

You'll see files such as:

```
Darling.exe

DarlingViewer.exe

darling.sample.json

InstallService.ps1

```

---

# Step 4 — Configure

Copy

```
darling.sample.json
```

Rename it

```
darling.json
```

---

Inside you'll configure

```
Servers

Credentials

SMTP

Alerts

Ports

Web Dashboard

```

Example

```json
{
   "Servers":[
      {
         "Name":"SQLPROD01",
         "Server":"SQLPROD01"
      },
      {
         "Name":"SQLPROD02",
         "Server":"SQLPROD02"
      }
   ]
}
```

---

# Step 5 — Install Windows Service

Run PowerShell as Administrator.

```
InstallService.ps1
```

or

```
sc create DarlingService ...
```

Once installed

```
services.msc
```

You should see

```
Darling Monitoring Service

Running
```

---

# Step 6 — Internal Database

Unlike previous versions,

Darling automatically installs and manages:

```
PostgreSQL

+

TimescaleDB
```

No manual PostgreSQL installation is needed for a standard deployment. ([GitHub][2])

---

# Step 7 — Add SQL Servers

There are multiple ways.

### Method 1

Desktop Viewer

```
Add Server

Server Name

Authentication

Encrypt

Trust Certificate

Save
```

---

### Method 2

Bulk Import

Paste

```
SQLPROD01

SQLPROD02

SQLPROD03

SQLTEST01

SQLDEV01
```

The tool automatically creates all monitoring connections. ([GitHub][1])

---

### Method 3

Using MCP AI

You can ask

```
Add every SQL Server in Production OU
```

or

```
Add all servers beginning SQLPRD
```

The MCP tools can perform bulk onboarding. ([GitHub][1])

---

# Step 8 — Required SQL Permissions

For monitoring, create a dedicated login rather than using `sa`. The exact permissions depend on the collectors you enable, but they generally require visibility into DMVs and Extended Events. Review the project's documentation and grant only the minimum required rights for your environment. ([GitHub][2])

---

# Step 9 — Collectors

Darling continuously gathers telemetry such as:

| Collector             | Purpose                    |
| --------------------- | -------------------------- |
| Wait Stats            | Bottleneck analysis        |
| Blocking              | Blocking chains            |
| Deadlocks             | XML deadlock graphs        |
| CPU                   | CPU utilization            |
| Memory                | Memory pressure            |
| TempDB                | TempDB usage               |
| File I/O              | Read/write latency         |
| Query Store           | Query performance          |
| Running Queries       | Active requests            |
| Job History           | SQL Agent jobs             |
| Plan Cache            | Cached execution plans     |
| Scheduler             | Runnable queue             |
| Latch Stats           | Latch contention           |
| Spinlocks             | Spinlock contention        |
| System Health         | Extended Events            |
| Configuration Changes | SQL configuration tracking |

Version 3.2 adds nine shared collectors across Lite and Darling, including latch stats, spinlock stats, CPU scheduler stats, plan cache stats, system health events, job history, agent status, and session summary statistics. ([GitHub][1])

---

# Step 10 — Web Dashboard

Open the configured web URL (default depends on your configuration).

You'll see:

* Fleet overview
* Server health
* Alerts
* Wait statistics
* Blocking
* CPU
* Memory
* TempDB
* Query performance
* Alert history
* Custom views

---

# Step 11 — Desktop Viewer

The desktop application offers richer investigation features, including:

* Query drill-down
* Execution plans
* Deadlock viewer
* Wait statistics
* Blocking chains
* Historical analysis
* Performance timelines

---

# Step 12 — Alerts

Configure thresholds for:

* CPU
* Memory
* Blocking
* Deadlocks
* Long-running queries
* Connection loss
* SQL Agent failures

Alerts can be delivered via:

* SMTP email
* Webhooks

Version 3.2 also adds connection-lost and connection-restored alerts. ([GitHub][1])

---

# Step 13 — Custom Views

One of the biggest new capabilities is building your own dashboards.

Examples:

```
Production Health

Morning DBA Dashboard

Storage Dashboard

AlwaysOn Dashboard

Security Dashboard

Performance Dashboard

Capacity Dashboard
```

Each can combine metrics, filters, charts, thresholds, and annotations. ([GitHub][1])

---

# Step 14 — MCP AI Integration

Darling exposes an MCP server so AI clients can interact with the monitoring environment.

Examples include:

* Build a dashboard for TempDB usage.
* Increase CPU alert threshold to 90%.
* Add ten new SQL Servers to monitoring.
* Show the top wait types across the fleet.
* Summarize last night's blocking incidents.

The MCP server supports both analysis and administrative tasks such as managing custom views and server onboarding. ([GitHub][1])

---

# Step 15 — Scaling

Typical guidance:

| Environment     | Recommended Edition |
| --------------- | ------------------- |
| 1–5 servers     | Lite                |
| 5–50 servers    | Darling             |
| 50–500+ servers | Darling             |

Darling is intended as the scalable, always-on edition. ([GitHub][2])

# How I would deploy it in a real enterprise

For a production environment with 100+ SQL Servers:

* Create a dedicated Windows monitoring VM.
* Install Darling as a Windows service.
* Use a dedicated least-privilege SQL login for monitoring.
* Enable SMTP and webhook notifications.
* Bulk onboard all SQL Servers.
* Organize servers into Production, UAT, QA, and Development groups using custom views.
* Retain historical performance data in the bundled PostgreSQL/TimescaleDB store.
* Use the web dashboard for operations teams and the desktop viewer for deep performance investigations.

With large on-premises SQL Server estates, this architecture aligns well with centralized monitoring while avoiding the overhead of installing a monitoring database on every SQL Server.


[1]: https://github.com/erikdarlingdata/PerformanceMonitor/releases?utm_source=chatgpt.com "Releases · erikdarlingdata/PerformanceMonitor"
[2]: https://github.com/erikdarlingdata/performancemonitor?utm_source=chatgpt.com "Free, open-source SQL Server performance monitoring — ..."
