## Fixing SCCM Update Failure

Server: DC01
Site: HQ1
Version: 5.00.9135.x



Table of Contents

	•	Overview
	•	Symptoms
	•	Root Cause
	•	Step-by-Step Fix
	•	Final State
	•	What I Learned
	•	Files and Logs I Saved
	•	Credits and Tools Used



## Overview

My SCCM updates got stuck.
The console showed them as “Available to download” with no progress.
CMUpdate.log showed SQL issues, certificate errors, and thread failures.

This document explains how I fixed the issue in my lab.



Symptoms

I saw the following:

	•	SQL connection failed
	•	“Server not found or not accessible”
	•	Certificate principal mismatch
	•	Staging folder hash mismatch
	•	SMS_HIERARCHY_MANAGER not starting
	•	Updates not installing
	•	Yellow security bar inside the console
	

Example log entries:

A network-related or instance-specific error occurred
The target principal name is incorrect
Failed to open registry key SMS_EXECUTIVE\Threads\SMS_HIERARCHY_MANAGER
Hash mismatch in CMUStaging


## Root Cause

The issue came from SQL connectivity:

	•	ODBC could not validate the SQL certificate
	•	SQL responded through IPv6
	•	ODBC test failed with principal mismatch
	•	SCCM could not talk to SQL during update
	•	The update restarted many times
	•	Staging folder rebuilt because of a hash mismatch

## Fixing the SQL ODBC connectivity resolved the entire issue.



## Step-by-Step Fix

## 1. Created a test ODBC connection

I opened:
ODBC Data Source Administrator (64-bit)

Created a DSN:

	•	Name: SCCM_SQL_TEST
	•	Server: DC01.netfusion.internal,1433
	•	Authentication: Windows Authentication


The test failed.



## 2. Adjusted ODBC encryption settings

Inside the DSN:

	•	Set Connection Encryption = Mandatory
	•	Left the Trust server certificate unchecked
	•	Cleared the “Hostname in certificate” field
	•	Left everything else default

The test still failed.



## 3. Passed the SQL test successfully

I kept Encryption = Mandatory.
This time, ODBC allowed encrypted traffic without forcing certificate validation.


## Result:

Connection established
Connection was encrypted without server certificate validation
TESTS COMPLETED SUCCESSFULLY

This confirmed SQL was the main blocker.



## 4. Restarted SCCM core services

Restarted:

	•	SMS_EXECUTIVE
	•	SMS_SITE_COMPONENT_MANAGER

This allowed SCCM to reload its threads.



## 5. Let SCCM rebuild the staging folder

After the SQL fix, CMUpdate.log showed:

	•	Staging folder hash mismatch
	•	SCCM deleted the folder
	•	SCCM extracted the update again
	•	Files copied into the cd.latest
	•	Components checked
	•	SMS_DATABASE_NOTIFICATION_MONITOR started
	•	SMS_HIERARCHY_MANAGER started
	•	SMS_REPLICATION_CONFIGURATION_MONITOR started

This was the first time all threads started correctly.



## 6. Confirmed site version update

In:

Administration → Site Configuration → Sites

I saw:

	•	Status: Active
	•	Version: 5.00.9135.1000
	•	All components working



## 7. Verified update state

Under:

Administration → Updates and Servicing

	•	Latest update showed Installed
	•	No stuck updates
	•	No warnings
	•	No errors



## Final State

	•	Update installed
	•	SQL communication is stable
	•	No certificate errors
	•	No thread failures
	•	No pending updates
	•	Site healthy



What I Learned

	•	Always test SQL ODBC first
	
	•	Name mismatch in certificates breaks SCCM updates
	
	•	Hash mismatch often means staging extraction failure
	
	•	SCCM internal threads rely on SQL



Files and Logs I Saved

	•	ODBC DSN screenshots
	•	CMUpdate.log
	•	Update status screenshot
	•	Final site version screenshot



Credits and Tools Used

Tools I used

	•	Windows Server 2022
	•	SQL Server 2019
	•	MECM
	•	CMTrace
	•	Event Viewer
	•	ODBC Data Source Administrator
	•	PowerShell
	•	Services.msc
	•	DNS Manager

Logs reviewed

	•	CMUpdate.log
	•	hman.log
	•	smsexec.log
	•	SQL ODBC test output

Credits

	•	Troubleshooting and testing done by me in my lab
	•	Research supported by Microsoft docs and community posts

	

	
## Author

**Emmanuel Anyanwu**  
[GitHub Profile](https://github.com/eanyanwu006)
	

