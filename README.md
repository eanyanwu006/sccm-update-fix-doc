# Fixing SCCM Update Failure  
**Server:** DC01  
**Site:** HQ1  
**Version:** 5.00.9135.x  
**Tag:** sccm-update-fix-2025  

---

## Overview

My SCCM updates were stuck.  
The console indicated updates as “Available to download,” but no installation occurred.  
CMUpdate.log reported SQL failures, certificate issues, and internal SCCM thread errors.

This guide documents how I diagnosed and fixed the issue.

---

## Symptoms

Problems I saw:

- SQL connection failed  
- “Server not found or not accessible”  
- Certificate principal mismatch  
- Staging folder hash mismatch  
- SMS_HIERARCHY_MANAGER thread not starting  
- Updates stayed in “Available”  
- Hotfix warnings in the console  

Example errors:

A network-related or instance-specific error occurred
The target principal name is incorrect
Hash mismatch in CMUStaging
Failed to open registry key SMS_EXECUTIVE\Threads\SMS_HIERARCHY_MANAGER

---

## Root Cause

Everything pointed to one source: **SQL connectivity**.

- ODBC driver failed certificate validation  
- SQL responded over IPv6  
- Hostname mismatch in the certificate  
- SCCM retries caused staging rebuild loops  
- Internal threads failed due to SQL not responding  

Fixing the SQL + ODBC link fixed everything else.

---

## Step-by-Step Fix

### 1. Created a test ODBC connection

Opened **ODBC Data Source Administrator (64-bit)**.  
Created DSN:

- **Name:** SCCM_SQL_TEST  
- **Server:** DC01.netfusion.internal,1433  
- **Authentication:** Windows Authentication  

The test failed at first.

---

### 2. Adjusted ODBC encryption settings

Inside DSN configuration:

- Connection Encryption = Mandatory  
- Trust server certificate = Off  
- Hostname in certificate = empty  
- All other settings left default  

Still failed.

---

### 3. Successful SQL connectivity

I kept Encryption = Mandatory.  
This forced encrypted communication but skipped certificate validation.

Result:

Connection established
Connection was encrypted without server certificate validation
TESTS COMPLETED SUCCESSFULLY

This confirmed that SQL connectivity was the root cause.

---

### 4. Restarted core SCCM services

I restarted:

- SMS_EXECUTIVE  
- SMS_SITE_COMPONENT_MANAGER  

This forced SCCM to reload internal components.

---

### 5. SCCM rebuilt the staging folder

After the SQL fix, SCCM did the rest automatically:

- Deleted corrupted staging folder  
- Extracted update again  
- Verified hash  
- Copied files to the cd.latest folder
- Started all internal threads  
- Finished remaining upgrade steps  

---

### 6. Confirmed SCCM version

In **Administration → Site Configuration → Sites**:

- Status: Active  
- Version: 5.00.9135.1000  
- All components running  

---

### 7. Verified update status

In **Updates and Servicing**:

- Latest update = Installed  
- Hotfix warnings gone  
- No pending items  

---

<p align="center">
  <img src="https://github.com/eanyanwu006/sccm-update-fix-doc/raw/main/mecm-updates-and-servicing.png" alt="MECM Updates and Servicing" width="800">
</p>


## Final State

- Update installed  
- SQL communication is working  
- No certificate errors  
- Threads running normally  
- Site healthy  

---

## What I Learned

- Always start with SQL ODBC when SCCM updates fail  
- Certificate mismatch breaks SCCM silently  
- Hash mismatch usually means staging extraction failure  
- Core SCCM threads depend on SQL availability  
- Fix SQL → SCCM fixes itself  

---

## Update Health Tips

To avoid future update failures
- Always test SQL connectivity before running updates  
- Run SCCM Service Connection Tool if offline  
- Keep consistent DNS and time sync 
- Avoid snapshots with broken checkpoints  
- Reboot the site server before applying updates

---


## Files & Logs I Saved

- ODBC DSN configs 
- CMUpdate.log 
- Hierarchy Manager log  
- smsexec log entries 
- Final site version screenshot (not included here)

---

## Extra Guides

### SQL Connectivity Checklist

- Make sure port **1433** is open  
- Confirm SQL Browser is enabled (if using named instances)  
- Ensure DC01 resolves from IPv4  
- Check SQL services:
  - SQL Server (MSSQLSERVER)
  - SQL Server Agent  
- Verify SPNs:

setspn -L DC01

Keys to look for:

MSSQLSvc/DC01.netfusion.internal:1433
MSSQLSvc/DC01:1433

Missing SPNs cause ODBC authentication failures.

---

### SCCM Logs to Check

These logs helped me find each stage of the problem.

| Log | Purpose |
|------|---------|
| `CMUpdate.log` | Main update engine |
| `hman.log` | Hierarchy Manager health |
| `smsexec.log` | Thread startup + component errors |
| `sitecomp.log` | Component installation/repair |
| `distmgr.log` | Content staging + distribution |
| `cloudmgr.log` | Cloud attach checks (optional) |

Logs are located in:

C:\Program Files\Microsoft Configuration Manager\Logs

Quick log search

Use CMTrace:
	•	Filter “error”, “fail”, “registry”, “SQL”, “certificate”.
	
---

Additional Guides & Troubleshooting

SQL Fixes

These checks help identify SQL issues before an SCCM update:

Test SQL port

Test-NetConnection -ComputerName DC01 -Port 1433

Check SQL services
	•	SQL Server (MSSQLSERVER)
	•	SQL Server Agent

Check TCP/IP

SQL Configuration Manager → Network Configuration → Enable TCP/IP.

Ensure SQL Browser is running

Needed for named instances.



### ODBC Troubleshooting

If ODBC fails:

1. Use FQDN

Bad: DC01
Good: DC01.netfusion.internal,1433

2. Disable IPv6 for SQL only if needed

If SQL answers via IPv6 first, SCCM logs show handshake failures.

3. Clear certificate hostname

If SQL uses a self-signed cert, a hostname mismatch breaks encryption.

Check encryption:

- Mandatory = Best  
- Optional = Works but less strict  
- Disabled = Only for testing  

Test credentials

---

## Version Tags

Tags I use for tracking updates:

- **sccm-update-fix-2025**
- **sql-fix-guide**
- **odbc-diagnostic**
- **mecm-troubleshooting**

---

## Credits / Tools Used

### Tools

- Windows Server 2022  
- SQL Server 2019  
- Microsoft Endpoint Configuration Manager  
- CMTrace  
- Event Viewer  
- ODBC Data Source Administrator  
- PowerShell  
- DNS Manager  
- Services.msc  

### Logs Reviewed

- CMUpdate.log  
- hman.log  
- smsexec.log  
- distmgr.log  
- SQL ODBC test results 


	
### Credits

- All troubleshooting done by me in my home lab  
- Microsoft technical documentation helped validate steps  
 

---








## Author

**Emmanuel Anyanwu**  
[GitHub Profile](https://github.com/eanyanwu006)









