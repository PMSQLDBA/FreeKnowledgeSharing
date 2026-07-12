# Understanding Erik Darling's Free SQL Server Performance Monitor

Source: https://erikdarling.com/free-sql-server-performance-monitoring/

## A Technical Evaluation of Features, Benefits, and Considerations

### Introduction

Monitoring SQL Server performance is essential for maintaining database availability, diagnosing performance bottlenecks, and proactively identifying issues before they impact business operations. Traditionally, enterprise-grade monitoring solutions have required significant licensing investments, making them less accessible for smaller organizations or environments with many SQL Server instances.

**Erik Darling's Free SQL Server Performance Monitor** offers an alternative by providing a comprehensive monitoring solution that is **free to use**, **open source under the MIT License**, and **community supported**. It includes many capabilities typically found in commercial monitoring products, such as performance data collection, alerting, execution plan analysis, and historical performance reporting.

This article provides a technical overview of the product, explains its licensing model, and discusses its advantages and limitations.

---

# What is Erik Darling's SQL Server Performance Monitor?

The SQL Server Performance Monitor is a lightweight monitoring solution designed specifically for Microsoft SQL Server environments. It collects performance metrics using T-SQL collectors and stores historical performance data within SQL Server, allowing DBAs to analyze workloads, identify bottlenecks, and troubleshoot performance problems.

The solution supports:

* SQL Server 2016 through SQL Server 2025
* Azure SQL Managed Instance
* Azure SQL Database (Lite Edition)
* Amazon RDS for SQL Server

Two deployment models are available:

| Edition          | Description                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| **Full Edition** | Installed on SQL Server and uses SQL Server Agent for scheduled data collection.                            |
| **Lite Edition** | Agentless desktop application that connects remotely without installing components on the monitored server. |

---

# Key Features

The monitoring solution includes numerous enterprise-level capabilities.

## Performance Data Collection

More than 30 collectors gather information about:

* Wait statistics
* CPU utilization
* Memory usage
* Blocking sessions
* Deadlocks
* Query Store
* TempDB utilization
* Disk I/O
* Active sessions
* Long-running queries

Historical data enables trend analysis instead of relying solely on real-time troubleshooting.

---

## Alerting

Supports notifications through:

* Email
* Microsoft Teams
* Slack
* PagerDuty
* Generic Webhooks

This allows administrators to receive alerts immediately when predefined thresholds are exceeded.

---

## Execution Plan Analysis

One of the strongest features is the integrated execution plan viewer.

Capabilities include:

* Graphical execution plans
* Automatic plan analysis
* Detection of common performance issues
* More than 30 optimization rules

This assists DBAs in identifying inefficient query execution plans more quickly.

---

## AI Integration

The product includes support for an MCP (Model Context Protocol) Server, enabling AI assistants to analyze SQL Server performance data.

Potential use cases include:

* Query performance analysis
* Wait statistics interpretation
* Troubleshooting recommendations
* Root cause analysis

Although AI can accelerate investigations, recommendations should always be validated before implementing changes in production.

---

## Security

The monitoring solution follows several security best practices:

* TLS encrypted communications
* DPAPI encrypted credentials
* Digitally signed binaries
* Parameterized SQL queries

Unlike cloud-based monitoring solutions, performance data remains within the organization's infrastructure.

---

## Reporting

Historical performance data is stored in SQL Server tables.

This enables organizations to build custom reports using:

* Power BI
* SQL Server Reporting Services (SSRS)
* Excel
* Custom T-SQL queries

Unlike many commercial products, organizations retain complete ownership of their monitoring data.

---

# Understanding the Licensing Model

The software is described as:

* Free
* Open Source (MIT License)
* Community Supported

Although these terms are often used together, they represent different concepts.

---

# Free Software

"Free" refers to the licensing cost.

Users can:

* Download the software
* Install it
* Use it
* Deploy it across multiple SQL Servers

without paying licensing fees.

### Benefits

* No purchase cost
* No subscription fees
* No per-server licensing
* Ideal for organizations managing many SQL Server instances

### Limitation

Free software does **not** automatically include:

* Technical support
* Service Level Agreements (SLAs)
* Product warranties

---

# Open Source (MIT License)

The software is released under the **MIT License**, one of the most permissive open-source licenses available.

The source code is publicly accessible, allowing organizations to understand exactly how the software works.

## Organizations may:

* View the source code
* Modify the code
* Fix bugs
* Add custom features
* Redistribute modified versions
* Use the software commercially

The only significant requirement is that the original copyright notice and MIT license remain included when redistributing the software.

### Advantages

* Full transparency
* Security audits are possible
* No vendor lock-in
* Unlimited customization
* Long-term sustainability

### Considerations

Custom modifications become the organization's responsibility.

When new software versions are released, locally modified code may require manual merging.

---

# Community Supported

Community support differs significantly from commercial enterprise support.

Instead of guaranteed vendor assistance, help is generally obtained through:

* Documentation
* GitHub issues
* Blog articles
* User discussions
* Community forums
* Optional consulting services (if offered)

### Benefits

* Active knowledge sharing
* Rapid community contributions
* Direct interaction with developers
* Lower operating costs

### Limitations

There are no guaranteed:

* Response times
* Resolution times
* 24×7 support
* Enterprise SLAs

Organizations with mission-critical production environments may prefer commercial support agreements.

---

# Technical Advantages

| Advantage                         | Explanation                                                               |
| --------------------------------- | ------------------------------------------------------------------------- |
| No licensing cost                 | Can monitor unlimited SQL Servers without additional licensing expenses.  |
| Open source                       | Complete visibility into the implementation and security.                 |
| Historical performance repository | Enables long-term trend analysis.                                         |
| Lightweight collectors            | Designed to minimize monitoring overhead.                                 |
| Execution plan analysis           | Simplifies SQL performance tuning.                                        |
| Multiple alerting options         | Integrates with common operational workflows.                             |
| AI integration                    | Accelerates performance investigations.                                   |
| Secure architecture               | Credentials remain encrypted and monitoring data stays on-premises.       |
| Flexible reporting                | Historical data can be queried directly using SQL or visualization tools. |

---

# Technical Considerations

| Consideration           | Explanation                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| No enterprise SLA       | Community support may not meet strict operational requirements.                             |
| Self-maintenance        | DBAs must manage upgrades, retention, backups, and monitoring database maintenance.         |
| Storage growth          | Historical monitoring data requires retention planning.                                     |
| Alert tuning            | Default thresholds may generate excessive notifications if not configured properly.         |
| Windows client          | Organizations seeking browser-based dashboards may prefer web-centric monitoring solutions. |
| AI recommendations      | Suggested optimizations should always be reviewed before production deployment.             |
| Custom code maintenance | Local modifications must be maintained during future software upgrades.                     |

---

# Example Deployment Scenario

Consider an organization managing **100 SQL Server instances**.

### Commercial Monitoring Solution

* Annual licensing fees
* Per-server licensing
* Vendor support
* Enterprise SLAs
* Closed-source software

### Erik Darling's Performance Monitor

* No licensing fees
* Unlimited server deployment
* Complete source code access
* Internal data ownership
* Optional consulting instead of mandatory support contracts

The organization can significantly reduce software costs while maintaining comprehensive SQL Server performance visibility, provided it has sufficient DBA expertise to operate and maintain the solution.

---

# When This Solution Is a Good Fit

This monitoring platform is particularly suitable for:

* Organizations seeking to eliminate SQL Server monitoring licensing costs
* SQL Server DBAs who require direct access to raw monitoring data
* Database consultants troubleshooting client environments
* Organizations with strict security requirements that prohibit cloud telemetry
* Teams comfortable maintaining open-source software

---

# When a Commercial Solution May Be Preferred

Commercial monitoring platforms may be more appropriate when organizations require:

* Guaranteed vendor support
* 24×7 technical assistance
* Enterprise Service Level Agreements (SLAs)
* Executive dashboards
* Centralized monitoring across heterogeneous database platforms
* Compliance requirements mandating commercial vendor support

---

# Conclusion

Erik Darling's Free SQL Server Performance Monitor demonstrates that a modern SQL Server monitoring solution does not necessarily require expensive licensing. By combining comprehensive performance data collection, execution plan analysis, historical reporting, alerting, and AI integration with an MIT open-source license, it delivers capabilities that are often associated with commercial products.

Its greatest strengths are **cost efficiency, transparency, flexibility, and data ownership**. Organizations gain unrestricted access to the source code, retain complete control over their monitoring data, and avoid recurring licensing costs.

The primary trade-off is the operational responsibility that accompanies open-source software. Unlike commercial platforms, organizations must rely on community support or internal DBA expertise for deployment, maintenance, customization, and troubleshooting.

For SQL Server-focused environments with experienced database administrators, this solution offers a compelling balance between functionality and cost, making it a strong alternative to traditional enterprise monitoring products.
