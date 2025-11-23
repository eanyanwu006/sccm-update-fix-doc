Fixing SCCM Update Failure

Server: DC01
Site: HQ1
Version: 5.00.9135.x


Overview

My SCCM updates were stuck.
The console showed updates as “Available to download,” but nothing was installed.
CMUpdate.log showed SQL connection failures, certificate issues, and SCCM threads that would not start.

This README documents the steps I used to fix the problem.


Symptoms

I noticed the following:
	•	SQL connection failed
	•	“Server not found or not accessible”
	•	Certificate principal mismatch
	•	Staging folder hash mismatch
	•	SMS_HIERARCHY_MANAGER thread failing
	•	Updates not installing
	•	Yellow security warning bar in the console

Sample log entries:

A network-related or instance-specific error occurred
The target principal name is incorrect
Failed to open registry key SMS_EXECUTIVE\Threads\SMS_HIERARCHY_MANAGER
Hash mismatch in CMUStaging



Root Cause

The failure came from SQL connectivity:
	•	ODBC could not validate the SQL certificate
	•	SQL responded using IPv6
	•	ODBC test failed with principal mismatch
	•	SCCM could not talk to SQL during the update
	•	The update restarted several times
	•	The staging folder kept rebuilding due to a hash mismatch

Fixing SQL ODBC connectivity resolved the entire issue.


Step-by-Step Fix

1. Created a test ODBC connection

Opened:

ODBC Data Source Administrator (64-bit)

Created a DSN:
	•	Name: SCCM_SQL_TEST
	•	Server: DC01.netfusion.internal,1433
	•	Authentication: Windows Authentication

The initial test failed.


2. Adjusted ODBC encryption settings

Inside the DSN:
	•	Set Connection Encryption = Mandatory
	•	Left the Trust server certificate unchecked
	•	Cleared Hostname in certificate
	•	Left other fields default

The test still failed.


3. Passed the SQL test successfully

I kept Encryption = Mandatory.
This time, the driver allowed encrypted traffic without enforcing certificate validation.

The test passed:

Connection established
Connection was encrypted without server certificate validation
TESTS COMPLETED SUCCESSFULLY

This confirmed SQL connectivity was the root of the problem.


4. Restarted core SCCM services

Restarted:
	•	SMS_EXECUTIVE
	•	SMS_SITE_COMPONENT_MANAGER

This forced SCCM to reload internal components.


5. Let SCCM rebuild the staging folder

After SQL was fixed, CMUpdate.log showed:
	•	Staging folder hash mismatch
	•	SCCM deleted the staging folder
	•	Re-extracted update content
	•	Copied files into the cd.latest
	•	Validated component files
	•	Started hierarchy and replication threads

This was the first time SMS_HIERARCHY_MANAGER and SMS_REPLICATION_CONFIGURATION_MONITOR started properly.


6. Confirmed site version

Under:

Administration → Site Configuration → Sites

I saw:
	•	Status: Active
	•	Version: 5.00.9135.1000
	•	All components running


7. Verified update state

Under:

Administration → Updates and Servicing
	•	Latest update shows Installed
	•	Other updates show Available
	•	No stuck states
	•	No errors


Final State
	•	SCCM update installed
	•	SQL communication is stable
	•	No certificate warnings
	•	No registry-thread errors
	•	No pending updates
	•	Site is fully healthy


What I Learned
	•	Always test ODBC first when SCCM updates fail
	•	Certificate mismatch breaks SQL connectivity
	•	Staging folder mismatch means extraction failure
	•	SMS_EXECUTIVE threads depend on successful SQL access


Files and Logs I Saved
	•	ODBC DSN screenshots
	•	CMUpdate.log
	•	SCCM “Installed update” screenshot
	•	Site version page screenshot


Credits / Tools Used

Tools
	•	Windows Server 2022
	•	SQL Server 2019
	•	MECM / SCCM
	•	CMTrace
	•	Event Viewer
	•	ODBC Data Source Administrator (64-bit)
	•	PowerShell
	•	Services.msc
	•	DNS Manager

Logs Reviewed
	•	CMUpdate.log
	•	hman.log
	•	smsexec.log
	•	ODBC test results

Credits
	•	All troubleshooting done by me in my home lab
	•	Some guidance taken from Microsoft docs
	•	Structure and cleanup done with ChatGPT for clarity

