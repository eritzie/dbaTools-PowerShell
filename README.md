# dbaTools-PowerShell

A personal reference of common [dbaTools](https://dbatools.io/) commands for SQL Server administration tasks. Covers installation, configuration, and day-to-day DBA operations.

> **Requirements:** PowerShell 5.1+ or PowerShell 7+, and sufficient permissions on target SQL Server instances.

---

## Table of Contents

- [Setup](#setup)
  - [Install dbaTools](#install-dbatools)
  - [Update dbaTools](#update-dbatools)
  - [Install Format-Markdown](#install-format-markdown)
  - [Set Execution Policy](#set-execution-policy)
  - [Trust Self-Signed Server Certificates](#trust-self-signed-server-certificates)
- [Server Build Checks](#server-build-checks)
  - [Update Build Reference](#update-build-reference)
  - [Check SQL Server Patch Level](#check-sql-server-patch-level)
  - [TempDB Configuration](#tempdb-configuration)
  - [Disk Alignment](#disk-alignment)
  - [Disk Allocation](#disk-allocation)
  - [Disk Speed](#disk-speed)
  - [Network Latency](#network-latency)
  - [Max DOP](#max-dop)
  - [Power Plan](#power-plan)
  - [Max Memory](#max-memory)
  - [SPNs](#spns)
- [Install Maintenance Tools](#install-maintenance-tools)
  - [sp_WhoIsActive](#sp_whoisactive)
  - [Ola Hallengren Maintenance Solution](#ola-hallengren-maintenance-solution)
  - [First Responder Kit](#first-responder-kit)
  - [SQL Agent Admin Alerts](#sql-agent-admin-alerts)
  - [Erik Darling's Toolset](#erik-darlings-toolset)
  - [SqlPackage](#sqlpackage)
- [Instance Information](#instance-information)
  - [Disk Space](#disk-space)
  - [Installed SQL Features](#installed-sql-features)
  - [Database Mail Configuration](#database-mail-configuration)
  - [TempDB Usage](#tempdb-usage)
  - [SQL Server Services](#sql-server-services)
  - [SQL Agent Job History](#sql-agent-job-history)
  - [Trace Flags](#trace-flags)
  - [Database Permissions](#database-permissions)
  - [Logins](#logins)
- [High Availability & Replication](#high-availability--replication)
  - [Replication Latency](#replication-latency)
  - [Export Replication Settings](#export-replication-settings)
  - [Log Shipping Status](#log-shipping-status)
  - [Log Shipping Errors](#log-shipping-errors)
  - [Database Mirroring Status](#database-mirroring-status)
- [Database Inventory](#database-inventory)
  - [Database List with Size](#database-list-with-size)
  - [Database Last Access Times](#database-last-access-times)
- [Database Maintenance](#database-maintenance)
  - [Index Fragmentation](#index-fragmentation)
  - [Backup Database](#backup-database)
  - [Restore Database](#restore-database)
  - [Repair Orphaned Users](#repair-orphaned-users)
- [Migration](#migration)
  - [Full Instance Migration](#full-instance-migration)
  - [Copy Login](#copy-login)
  - [Copy Database](#copy-database)
  - [Copy SQL Agent Job](#copy-sql-agent-job)
  - [Copy Linked Server](#copy-linked-server)

  ---

## Setup

### Install dbaTools
```powershell
Install-Module dbatools
```

### Update dbaTools

dbatools releases frequently — older versions can have bugs or missing features. Run this periodically to stay current:
```powershell
Update-Module dbatools
```

### Install Format-Markdown

Third-party module for formatting PowerShell objects as proper markdown tables. Required for the Database Inventory commands below:
```powershell
Install-Module Format-Markdown
```

### Set Execution Policy

PowerShell blocks unsigned scripts by default. `RemoteSigned` allows local scripts while requiring remote scripts to be signed:
```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Trust Self-Signed Server Certificates

SQL Server 2022+ (and some patched older versions) enforce encrypted connections by default. Dev and internal servers typically use self-signed certificates, which the client rejects unless you explicitly trust them. This setting persists across sessions:
```powershell
Set-DbatoolsConfig -FullName sql.connection.trustcert -Value $true -PassThru | Register-DbatoolsConfig
```

## Server Build Checks

A set of independent checks for validating a server's configuration and performance baseline.
These can be run individually or as part of a new server setup review.

Note: Some commands use `-ComputerName` (OS-level, no SQL required) vs. `-SqlInstance` (requires SQL connectivity).

### Update Build Reference

Updates the local dbatools build reference cache from the internet, then returns the build information for an instance. Run this before `Test-DbaBuild` if the instance is showing as unknown:
```powershell
Get-DbaBuildReference -SqlInstance <ServerName> -Update
```

### Check SQL Server Patch Level

Checks whether a SQL Server instance is on the latest known build. Useful for patch compliance audits:
```powershell
Test-DbaBuild -SqlInstance <ServerName> -Latest |
    Select-Object SqlInstance, NameLevel, SPLevel, CULevel, KBLevel, Build, BuildTarget, Compliant |
    Format-Table -AutoSize
```

### TempDB Configuration

Checks TempDB configuration against best practices, including file count, sizing, and autogrowth settings:
```powershell
Test-DbaTempDbConfig -SqlInstance <ServerName> | Format-Table -AutoSize
```

### Disk Alignment

Verifies partitions are aligned to 64K boundaries. Misalignment causes unnecessary I/O overhead:
```powershell
Test-DbaDiskAlignment -ComputerName <ServerName> | Format-Table -AutoSize
```

### Disk Allocation

Checks that NTFS allocation unit size is set to 64K on data volumes, per SQL Server best practice:
```powershell
Test-DbaDiskAllocation -ComputerName <ServerName> | Format-Table -AutoSize
```

### Disk Speed

Measures read/write throughput on volumes hosting SQL Server files:
```powershell
Test-DbaDiskSpeed -SqlInstance <ServerName> | Format-Table -AutoSize
```

### Network Latency

Tests round-trip latency between the client and the SQL Server instance:
```powershell
Test-DbaNetworkLatency -SqlInstance <ServerName> | Format-Table -AutoSize
```

### Max DOP

Tests whether MAXDOP is configured per best practice, and optionally applies the recommended value:
```powershell
Test-DbaMaxDop -SqlInstance <ServerName> | Set-DbaMaxDop | Format-Table -AutoSize
```

### Power Plan

Verifies the server is running the High Performance power plan — anything else throttles CPU:
```powershell
Test-DbaPowerPlan -ComputerName <ServerName> | Format-Table -AutoSize
```

### Max Memory

Tests whether max server memory is configured appropriately for the host's physical RAM.
The filter limits `Set-DbaMaxMemory` to only instances where the configured max exceeds total RAM:
```powershell
Test-DbaMaxMemory -SqlInstance <ServerName> |
    Where-Object { $_.MaxValue -gt $_.Total } |
    Set-DbaMaxMemory
```

### SPNs

Validates that SQL Server SPNs are correctly registered in Active Directory. Accepts a comma-separated list for checking multiple instances at once:
```powershell
Test-DbaSpn -ComputerName <ServerName>| Format-Table -AutoSize
```

## Install Maintenance Tools

Standard third-party tools deployed to SQL Server instances for monitoring and maintenance.
Update the `-Database` target per your environment conventions.

### sp_WhoIsActive

Active session monitoring stored procedure by Adam Machanic:
```powershell
Install-DbaWhoIsActive -SqlInstance <ServerName> -Database master
```

### Ola Hallengren Maintenance Solution

Index maintenance, backups, and integrity checks. `-CleanupTime` sets job history retention in hours:
```powershell
Install-DbaMaintenanceSolution -SqlInstance <ServerName> -Database master -CleanupTime 72 -InstallJobs -Verbose
```

### First Responder Kit

Brent Ozar's diagnostic toolkit (sp_Blitz, sp_BlitzFirst, sp_BlitzIndex, etc.):
```powershell
Install-DbaFirstResponderKit -SqlInstance <ServerName> -Database DBA
```

### SQL Agent Admin Alerts

Creates standard SQL Server Agent alerts for severity 17–25 errors and critical I/O errors (823, 824, 825):
```powershell
Install-DbaAgentAdminAlert -SqlInstance <ServerName>
```

### Erik Darling's Toolset

Diagnostic stored procedures including sp_PressureDetector, sp_HumanEvents, and sp_QuickieStore:
```powershell
Install-DbaDarlingData -SqlInstance <ServerName> -Database <DatabaseName>
```

### SqlPackage

Installs Microsoft's SqlPackage CLI, used for DACPAC/BACPAC export and import operations:
```powershell
Install-DbaSqlPackage
```

## Instance Information

Read-only commands for gathering state across one or more instances. Useful for audits, onboarding a new server, or spot-checking configuration.

### Disk Space

Returns volume-level disk space for a server, sorted by computer and volume name:
```powershell
Get-DbaDiskSpace -ComputerName <ServerName> |
    Sort-Object ComputerName, Name |
    Format-Table -AutoSize
```

### Installed SQL Features

Lists all SQL Server features installed on the host:
```powershell
Get-DbaFeature -ComputerName <ServerName> -Verbose | Format-Table -AutoSize
```

### Database Mail Configuration

Returns the Database Mail configuration for one or more instances. Accepts a comma-separated instance list:
```powershell
Get-DbaDbMail -SqlInstance <ServerName> | Format-Table -AutoSize

Get-DbaDbMailAccount -SqlInstance <ServerName> -Account 'Default SMTP Account' | Format-Table -AutoSize

Get-DbaDbMailProfile -SqlInstance <ServerName> | Format-Table -AutoSize
```

### TempDB Usage

Returns current tempdb session usage. `Out-GridView` is useful here for sorting and filtering interactively:
```powershell
Get-DbaTempdbUsage -SqlInstance <ServerName> | Out-GridView
```

### SQL Server Services

Returns SQL Server services on a host with state and start mode:
```powershell
Get-DbaService -ComputerName <ServerName> |
    Select-Object ComputerName, ServiceName, State, StartMode |
    Format-Table -AutoSize
```

### SQL Agent Job History

Returns job execution history filtered by outcome. Useful for quickly surfacing recent failures:
```powershell
Get-DbaAgentJobHistory -SqlInstance <ServerName> -Job '<JobName>' -OutcomeType Failed | Out-GridView
```

### Trace Flags

Returns enabled trace flags on an instance:
```powershell
Get-DbaTraceFlag -SqlInstance <ServerName>
```

### Database Permissions

Returns permissions for a database, optionally filtered by grantee. `-IncludeServerLevel` includes instance-level permissions alongside database permissions:
```powershell
Get-DbaPermission -SqlInstance <ServerName> -Database <DatabaseName> `
    -ExcludeSystemObjects -IncludeServerLevel |
    Where-Object Grantee -eq '<RoleOrUserName>' |
    Sort-Object Securable |
    Select-Object PermState, PermissionName, SecurableType, Securable |
    Out-GridView
```

### Logins

Returns all logins on an instance. `-PassThru` allows selecting logins from the grid view and piping them to further commands:
```powershell
Get-DbaLogin -SqlInstance <ServerName> | Out-GridView -PassThru
```

## High Availability & Replication

Status checks for replication and log shipping. These are read-only and safe to run against production.

### Replication Latency

Tests replication latency for a publisher instance. `-DisplayTokenHistory` shows the full tracer token history rather than just the latest:
```powershell
Test-DbaRepLatency -SqlInstance <ServerName> -DisplayTokenHistory
```

### Export Replication Settings

Exports the replication server configuration to a SQL script. Useful for documenting or recreating a publisher's setup:
```powershell
Get-DbaRepServer -SqlInstance <ServerName> |
    Export-DbaRepServerSettings -Path C:\temp\replication.sql
```

### Log Shipping Status

Returns log shipping status for an instance. `-Simple` condenses the output to the most relevant columns:
```powershell
Test-DbaDbLogShipStatus -SqlInstance <ServerName> -Simple
```

### Log Shipping Errors

Returns log shipping error history for an instance from a given date. `-Secondary` limits results to the secondary server role:
```powershell
Get-DbaDbLogShipError -SqlInstance <ServerName> -DateTimeFrom "<MM/DD/YYYY>" -Secondary
```

### Database Mirroring Status

Returns mirroring configuration and status for all mirrored databases on an instance:
```powershell
Get-DbaDbMirror -SqlInstance <ServerName>
```

## Database Inventory

Commands for generating database-level inventory reports. The custom format hashtables below produce human-readable size output and placeholder columns (Usage, Notes, SME) suitable for a handoff document or wiki table.

Requires the `Format-Markdown` module — see [Install Maintenance Tools](#install-maintenance-tools).

### Database List with Size

Returns all user databases on an instance with compatibility level, owner, and formatted size. Excludes system databases:
```powershell
$fmtName  = @{ label = "DBName"; Expression = { $_.Name } }
$fmtSize  = @{ label = "Size";   Expression = {
    if    ($_.SizeMB -ge 1048567)                          { '{0:N2} TB' -f [math]::Round($_.SizeMB * 1MB / 1TB, 2) }
    elseif($_.SizeMB -gt 1024 -and $_.SizeMB -lt 1048567) { '{0:N2} GB' -f [math]::Round($_.SizeMB * 1MB / 1GB, 2) }
    else                                                   { '{0:N2} MB' -f [math]::Round($_.SizeMB, 2) }
}}
$fmtUsage = @{ label = "Usage"; Expression = { " " } }
$fmtNotes = @{ label = "Notes"; Expression = { " " } }
$fmtSME   = @{ label = "SME";   Expression = { " " } }

Get-DbaDatabase -SqlInstance <ServerName> -ExcludeSystem |
    Select-Object SqlInstance, $fmtName, Compatibility, Owner, $fmtSize, $fmtUsage, $fmtNotes, $fmtSME |
    Sort-Object SqlInstance, DBName |
    Format-Markdown
```

### Database Last Access Times

Returns the last index read and write times per database. Useful for identifying inactive or orphaned databases. Accepts a comma-separated instance list:
```powershell
Get-DbaDatabase -SqlInstance <ServerName> -ExcludeSystem -IncludeLastUsed |
    Select-Object ComputerName, Name, LastIndexRead, LastIndexWrite |
    Sort-Object ComputerName, Name |
    Format-Table -AutoSize
```

## Database Maintenance

### Index Fragmentation

Returns fragmentation levels per index for a database. Pipe to `Out-File` to save as a report:
```powershell
Get-DbaHelpIndex -SqlInstance <ServerName> -Database <DatabaseName> -IncludeFragmentation |
    Select-Object SqlInstance, Database, Object, Index, IndexType, KeyColumns, IndexFragInPercent |
    Format-Table -AutoSize |
    Out-File -FilePath <OutputPath> -Append
```

To scan all user databases on an instance, use `-ExcludeDatabase` to skip system and unwanted databases. `-IncludeStats` adds index statistics alongside fragmentation data:
```powershell
Get-DbaHelpIndex -SqlInstance <ServerName> `
    -ExcludeDatabase master, msdb, model, tempdb, <DatabaseName1>, <DatabaseName2> `
    -IncludeFragmentation -IncludeStats |
    Out-GridView
```

### Backup Database

Copy-only full backup — safe for ad hoc backups as it does not break the existing backup chain:
```powershell
Backup-DbaDatabase -SqlInstance <ServerName> -Database <DatabaseName> -Type Full -CopyOnly -Path <BackupPath>
```

### Restore Database

Simple restore, replacing the existing database if it exists:
```powershell
Restore-DbaDatabase -SqlInstance <ServerName> -DatabaseName <DatabaseName> -Path <BackupFilePath> -WithReplace
```

Restore with explicit file mapping when the target server uses different drive paths than the source. `-FileMapping` takes a hashtable of logical name to physical path:
```powershell
$FileStructure = @{
    '<LogicalDataName>' = '<DataFilePath>'
    '<LogicalLogName>'  = '<LogFilePath>'
}

Restore-DbaDatabase -SqlInstance <ServerName> -DatabaseName <DatabaseName> -Path <BackupFilePath> `
    -FileMapping $FileStructure -WithReplace `
    -DestinationDataDirectory <DataDirectory> -DestinationLogDirectory <LogDirectory>
```

### Repair Orphaned Users

Remaps or drops orphaned database users that no longer have a matching server login:
```powershell
Repair-DbaDbOrphanUser -SqlInstance <ServerName> -Database <DatabaseName>
```

## Migration

Commands for copying objects between SQL Server instances.

### Full Instance Migration

Migrates all supported objects between instances in a single operation. Use `-Exclude` to skip object types that are handled separately or not needed on the destination:
```powershell
Start-DbaMigration -Verbose -Source <SourceServer> -Destination <DestinationServer> `
    -Exclude Databases, SpConfigure, CentralManagementServer, BackupDevices, Audits, `
             ExtendedEvents, PolicyManagement, ResourceGovernor, ServerAuditSpecifications, `
             DataCollector, Logins `
    -DisableJobsOnDestination
```

### Copy Login

Copies a SQL Server login from one instance to another, including SID and password hash:
```powershell
Copy-DbaLogin -Source <SourceServer> -Destination <DestinationServer> -Login <LoginName>
```

### Copy Database

Copies a database between instances using backup/restore. `-SharedPath` must be accessible by both instances. `-Force` overwrites the destination if it exists:
```powershell
Copy-DbaDatabase -Source <SourceServer> -Destination <DestinationServer> -Database <DatabaseName> `
    -BackupRestore -SharedPath <UNCPath> -Force
```

### Copy SQL Agent Job

Copies a SQL Agent job to another instance. `-DisableOnDestination` prevents the job from running immediately after copy:
```powershell
Copy-DbaAgentJob -Source <SourceServer> -Destination <DestinationServer> -Job '<JobName>' -DisableOnDestination
```

### Copy Linked Server

Copies a linked server definition including credentials from one instance to another:
```powershell
Copy-DbaLinkedServer -Source <SourceServer> -Destination <DestinationServer>
```