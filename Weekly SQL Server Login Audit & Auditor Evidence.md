# SOP: Weekly SQL Server Login Audit & Auditor Evidence

**Doc ID:** SDC-SOP-SEC-015 | **Version:** 1.0 | **Classification:** Internal

**Owner:** Praveen Madupu | **Approver:** DBA Lead / IT Security

**Frequency:** Weekly (automated, Mon 06:00) + quarterly submission

**Related artifacts:** `SDC-AUDIT-LOGIN-REVIEW.sql` (v1.1, interactive) · `SDC-AUDIT-LOGIN-REPORT-DEPLOY.sql` (v2.0, scheduled)

**Applies to:** SQL Server 2012–2022 — on-prem, AWS EC2, Azure VM. **Not** Azure SQL DB / Synapse.

---

## 1. Purpose

- Produce recurring, dated evidence of **who can log in, at what privilege, and whether they are used**.
- Satisfy periodic access-review control requests without a manual pull per instance per quarter.
- Detect privilege drift (new sysadmin, role change, re-enabled account) **within 7 days**, not at audit time.

## 2. Scope

**In:** server logins, server roles, explicit server permissions, SQL login password controls, database user mapping, orphaned users, login activity, dormant accounts.

**Out (state explicitly if asked):**
- Application-level / in-app user accounts.
- Active Directory group *membership expansion* — SQL sees the group, not its members.
- Linked server credentials, credentials/proxies, contained DB users in AGs beyond the primary.
- Object-level (table/proc) permissions — separate review.

## 3. Roles

| Role | Responsibility |
|---|---|
| DBA on-call | Monday triage of the report; remediate HIGH within SLA |
| DBA Lead | Approves standard values (§10); signs off quarterly submission |
| IT Security | Consumes HIGH findings; owns exception register |
| App owner | Confirms business need for dormant / non-expiring service accounts |

**RACI note:** DBA is *Responsible* for evidence production, **not** *Accountable* for access decisions. Do not drop a login on the strength of this report alone — see §7.

---

## 4. Prerequisites

- [ ] Utility DB exists (default `DBA_Admin`); SIMPLE recovery is fine, but **include it in backups** — it is the evidence store.
- [ ] Database Mail configured; profile name known. Verify: `EXEC msdb.dbo.sysmail_help_profile_sp;`
- [ ] SQL Agent running and set to auto-start.
- [ ] Agent service account is **sysadmin** on the instance. Needed for: `xp_instance_regread`, `xp_readerrorlog`, all-DB loop, `PWDCOMPARE`. Without it the script does not fail — it emits `NOTE` findings and produces *quietly incomplete evidence*, which is worse. Confirm before trusting output.
- [ ] DBA distribution list address confirmed (not an individual).
- [ ] Change ticket raised (§12) — deploying to prod even though read-only.

## 5. Object inventory

| Object | Type | Notes |
|---|---|---|
| `dbo.LoginAudit_Run` | table | 1 row/run + summary counts. Parent; cascades on delete |
| `dbo.LoginAudit_Login` | table | Inventory snapshot per run — **this is what makes deltas possible** |
| `dbo.LoginAudit_Finding` | table | Findings per run |
| `dbo.LoginAudit_Html` | table | Rendered report, 1 row per line. Feeds the file attachment |
| `dbo.fn_LoginAudit_HtmlEncode` | scalar fn | Encodes `&` first, then `<`, `>`, `"` |
| `dbo.usp_LoginAudit_Collect` | proc | All checks; read-only vs instance; returns `@RunId` |
| `dbo.usp_LoginAudit_Report` | proc | Builds HTML + sends mail; re-runnable for any `RunId` |
| `SDC - Weekly Login Audit` | Agent job | Mon 06:00, 1 step, 1 retry |

**Sizing:** ~1 KB/login/run. 200 logins × 52 wks ≈ 10 MB/yr/instance. Retention 26 months, auto-purged in `Collect`.

---

## 6. Deployment procedure

### Step 1 — Stage the script
```
Open SDC-AUDIT-LOGIN-REPORT-DEPLOY.sql in SSMS, connect to target instance.
Edit line: USE [DBA_Admin];      -- utility DB name
```
**Expected:** nothing yet. **If it fails:** DB missing → uncomment the `CREATE DATABASE` guard at the top.

### Step 2 — Run SECTIONS 1–5 only (stop before SECTION 7)
**Expected:** 4 tables, 1 function, 2 procs created. **If it fails:** check for an existing `LoginAudit_*` object from a prior version — the `IF OBJECT_ID ... IS NULL` guards will skip creation and leave a stale schema. Drop and re-run.

### Step 3 — Smoke test, no mail (§6.1 in script)
```sql
DECLARE @NewRun INT;
EXEC dbo.usp_LoginAudit_Collect @RunId = @NewRun OUTPUT;
EXEC dbo.usp_LoginAudit_Report  @RunId = @NewRun, @SendMail = 0;
```
**Expected:** one `Html` cell + a size row. Copy the cell out, save as `.html`, open in a browser.
**Check before proceeding:** table cells line up; no `NOTE` findings about registry/ERRORLOG (= permission gap); sysadmin list matches reality.
**If it fails:** see §9.

### Step 4 — Mail test to **yourself**, not the DL
```sql
DECLARE @LastRun INT = (SELECT MAX(RunId) FROM dbo.LoginAudit_Run);
EXEC dbo.usp_LoginAudit_Report @RunId = @LastRun, @SendMail = 1,
     @MailProfile = N'DBA_Mail_Profile', @Recipients = N'you@domain.com';
```
**Expected:** email with rendered body **and** a `LoginAudit_<instance>_<yyyymmdd>.html` attachment that opens standalone.
**Verify delivery:**
```sql
SELECT TOP 5 sent_status, sent_date, subject FROM msdb.dbo.sysmail_allitems ORDER BY mailitem_id DESC;
SELECT TOP 5 * FROM msdb.dbo.sysmail_event_log ORDER BY log_id DESC;
```
**If it fails:** §9.

### Step 5 — Install the job (SECTION 7)
Edit `@AuditDb`, `@MailProfile`, `@Recipients`, `@JobOwner`. Run. Section is drop-and-recreate → safe to redeploy.

### Step 6 — Prove the whole path
```sql
EXEC msdb.dbo.sp_start_job @job_name = N'SDC - Weekly Login Audit';
WAITFOR DELAY '00:00:30';
EXEC msdb.dbo.sp_help_jobactivity @job_name = N'SDC - Weekly Login Audit';
```
**Expected:** step 1 succeeds; DL receives mail; new `RunId` present.

### Step 7 — Record
- Add instance to the audit population register.
- Note the **baseline RunId** — the first run has no deltas by definition; say so if an auditor asks.

**Rollout order:** dev → one prod instance → wait one full weekly cycle → remainder. Do not mass-deploy on day 1; the first week's report on a legacy instance is usually long and needs triage capacity.

---

## 7. Weekly operation (DBA on-call, Monday)

1. **Read "Changes since the previous run" first.** It is the only section that is time-sensitive.
2. Triage per §8. Log each HIGH in the ticket system — *the ticket is the evidence of the review, not the email.*
3. Anything accepted → exception register with owner + expiry. An unexplained finding that persists for 12 weekly reports is an audit finding in itself.
4. If no mail arrived by 07:00 → §9 (silence is not a pass).

**Rule:** never remediate off the report alone.
- New sysadmin → confirm against the change ticket **before** revoking. Legitimate emergency access is common.
- Dormant login → confirm with the app owner. "No evidence of use" ≠ unused; it means not seen in the retained window (§11).
- Drop nothing. Disable, wait one full business cycle (incl. month-end), then drop.

## 8. Findings triage

| Finding | Severity | Action | Target |
|---|---|---|---|
| Blank/weak password | HIGH | Reset now. Names deliberately **not** in the report — run `SDC-AUDIT-LOGIN-REVIEW.sql` interactively, result set `[04]` | Same day |
| Failed login auditing off | HIGH | Set *Both failed and successful* → **requires instance restart** to take effect. Schedule it | Next window |
| Excessive sysadmin | HIGH | Confirm each vs. approved list; revoke unapproved | 2 business days |
| BUILTIN\Administrators as login | HIGH | Replace with a named DBA AD group; drop BUILTIN | Next change window |
| `guest` with roles in user DB | HIGH | `REVOKE CONNECT FROM guest` | 2 business days |
| New sysadmin (delta) | HIGH | Match to change ticket; revoke if none | 2 business days |
| `CHECK_POLICY = OFF` | HIGH | Cannot be turned on without a password reset — coordinate with app owner | 10 business days |
| Mixed Mode auth | MEDIUM | Document justification once; recurs every run by design | Register once |
| `CHECK_EXPIRATION = OFF` | MEDIUM | Service accounts: document rotation as compensating control | Register |
| Orphaned users | MEDIUM | `ALTER USER x WITH LOGIN = y` or drop — evidence of incomplete deprovisioning | 10 business days |
| Dormant login | MEDIUM | App owner confirmation → disable → drop | 30 days |
| ≥10 failed attempts | MEDIUM | Usually a stale app credential, occasionally not. Check source host | 5 business days |
| `sa` enabled | MEDIUM | Rename + disable | Next window |
| `NOTE` (registry/ERRORLOG/DB skipped) | — | **Permission or access gap. Fix before the next run — the evidence is incomplete** | Same day |

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Msg 537 Invalid length parameter` | Pre-v1.1 ERRORLOG parse: `SUBSTRING` guarded only in `WHERE`; optimizer evaluates the projection before the filter | Use v1.1+. Parse is staged into `#LoginEvents` and the *length argument* is guarded. **Do not "simplify" it back to one SELECT** |
| `Null value is eliminated by an aggregate` | `MAX()` over logins with no activity rows | Benign — it is what produces "no evidence". Do not suppress with `SET ANSI_WARNINGS OFF` |
| Report has empty/blank attachment | `@query_result_no_padding`/`@query_result_header` altered, or `LoginAudit_Html` empty for that RunId | Restore mail params; re-run `usp_LoginAudit_Report` |
| Attachment missing entirely | DB Mail's attachment query runs on a **separate connection** — it cannot see temp tables | Do not refactor `LoginAudit_Html` to a `#temp` |
| Table cells shifted left | A `td` expression returned NULL — `FOR XML PATH` omits the element entirely | Wrap **every** `td` in `ISNULL()` |
| Mail `sent_status = failed` | Profile/account/relay | `sysmail_event_log`; test with `sp_send_dbmail` plain text |
| No mail, no job failure | Job succeeded, DL wrong/filtered | Check `sysmail_allitems`; check spam quarantine — HTML+attachment from a server is a classic false positive |
| Job fails: `EXECUTE permission denied on xp_readerrorlog` | Agent account not sysadmin | Grant, or set `@ScanErrorLogs = 0` and accept degraded activity evidence (document it) |
| `NOTE: Database skipped` | DB offline / AG secondary not readable / encrypted & inaccessible | Expected for AG secondaries. Note in submission; collect from the primary |
| Dormant list is huge on first run | ERRORLOG has just been recycled | Expected. Evidence strengthens weekly (§11) |
| Job duration climbing | Many DBs / large ERRORLOG set | Lower `@ErrorLogArchivesToScan` |

## 10. Standard values (approved by DBA Lead — edit to match your standard)

| Parameter | Default | Notes |
|---|---|---|
| `@StaleLoginDays` | 90 | Align to the access-review cycle |
| `@PasswordMaxAgeDays` | 90 | Must match the org password standard |
| `@MaxSysadminCount` | 3 | Per-instance; raise for clustered/tooling-heavy hosts |
| `@ErrorLogArchivesToScan` | 6 | With weekly recycle ≈ 6 weeks of activity |
| `@RetentionMonths` | 26 | ≥ 2 audit cycles |
| Schedule | Mon 06:00 | Before business hours, after weekend maintenance |

## 11. Known limitations — **disclose these proactively**

An auditor who finds a limitation you did not declare discounts the whole report.

1. **Activity evidence is retention-bound.** Sources: `dm_exec_sessions` (point-in-time) + ERRORLOG (bounded by recycling *and* by the login audit level) + last-known-login carried forward from retained runs. Nothing before the first retained run can be evidenced. Persisted SQL Server Audit is the only durable fix.
2. **"No evidence of use" ≠ "never used."** Wording in the report is deliberate. Do not restate it as "unused" in the submission.
3. **AD groups are not expanded.** A single `DOMAIN\SQL_Admins` row may be 40 people. Effective-access population requires the AD-side extract. This is the most common gap raised on review.
4. **Point-in-time.** Access granted and revoked between Monday runs is invisible. Only a persisted Audit closes this.
5. **Weak-password names are not stored or emailed** — by design (§8).
6. **Windows password policy is not read** — `CHECK_POLICY = ON` proves enforcement is *delegated* to the OS policy, not that the policy is strong. Pair with the GPO extract.
7. **One instance per run.** Population completeness is a manual control — reconcile against the CMDB.

## 12. Change control & rollback

- **Deploy:** standard change. Read-only against the instance; creates objects only in the utility DB. No restart, no config change.
- **Rollback:**
```sql
EXEC msdb.dbo.sp_delete_job @job_name = N'SDC - Weekly Login Audit';
-- retain the tables: they are audit evidence. Drop only with Compliance sign-off.
```
- **Disable temporarily:** disable the job. Do **not** drop the tables — a retention gap is a finding.
- **Version bump:** re-run SECTIONS 3–5 (procs are drop/create) then SECTION 7. Tables are preserved by the `IF OBJECT_ID ... IS NULL` guards.

## 13. Quarterly auditor submission

1. Pick the run closest to period end:
```sql
SELECT RunId, RunDateTime, HighFindings, MediumFindings, SysadminCount, DormantCount
FROM dbo.LoginAudit_Run ORDER BY RunId DESC;
```
2. Re-issue unchanged (report is rebuilt from stored data, not re-collected — same RunId = same content):
```sql
EXEC dbo.usp_LoginAudit_Report @RunId = <n>, @SendMail = 0;
```
3. Package per instance: HTML report + weekly delta history + ticket references for each HIGH + exception register.
4. State the §11 limitations in the cover note.
5. **Never edit the HTML output.** If it is wrong, fix the script and re-run with a new RunId; keep both. An edited evidence artifact is worthless.

---

## Revision history

| Ver | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-07-15 | Praveen Madupu | Initial. Covers REVIEW v1.1 + DEPLOY v2.0 |

## Open items

- [ ] Confirm §10 values with DBA Lead
- [ ] Decide: SQL Server Audit (`SUCCESSFUL_LOGIN_GROUP` / `FAILED_LOGIN_GROUP`) to close limitations 1 & 4 — cost is log volume
- [ ] Add AD group expansion (PowerShell + `Get-ADGroupMember`) to close limitation 3
- [ ] Central collection: push each instance's `LoginAudit_*` to one repository → one population-wide report
