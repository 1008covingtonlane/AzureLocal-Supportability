<!-- tsg-metadata
{
  "schema": "azure-local-supportability/tsg-metadata/v1",
  "document_type": "troubleshoot",
  "products": ["Azure Local"],
  "detector": {
    "type": "command",
    "signal": "Get-Module -Name AzStackHci.EnvironmentChecker -ListAvailable | Where-Object {$_.Path -like \"*$($_.Version)*\"}"
  },
  "validation": {
    "fidelity_level": "L4",
    "technical_grade": "A",
    "reproduction_substrate": "hardware",
    "automation_status": "proven",
    "last_validated": "2026-11-20",
    "spec_ref": "AzStackHci_EnvironmentChecker_ModuleShadowing"
  }
}
-->

# Troubleshoot Environment Checker module versions

## Table of contents

- [Revision history](#revision-history)
- [At a glance](#at-a-glance)
- [Overview](#overview)
- [Symptoms](#symptoms)
- [Cause](#cause)
- [Terms used in this article](#terms-used-in-this-article)
- [Where you can see this problem](#where-you-can-see-this-problem)
- [Before you start](#before-you-start)
- [Issue Validation](#issue-validation)
- [Mitigation Details](#mitigation-details)
- [Verify the fix](#verify-the-fix)
- [Retry the failed operation](#retry-the-failed-operation)
- [Rollback](#rollback)
- [When to contact Microsoft support](#when-to-contact-microsoft-support)
- [Prevention](#prevention)
- [Related documentation](#related-documentation)

## Revision history

| Date | Summary |
| --- | --- |
| 2026-11-20 | Corrected the expected inbox module path and the Issue Validation command. Added the error that newer builds show, steps for module files that are still in use, commands for all machines, and guidance to verify, escalate, and prevent the problem. Added the metadata marker. Step 1 now lists every PowerShell session that could hold the external copy's files, including remote sessions, and the impact labels are replaced with plain statements of what each step changes. |
| 2026-11-26 | Corrected the statement that Windows cannot report which process holds the module's files: Step 1 now uses Windows Restart Manager to list the exact processes, and Step 2 stops only the PowerShell sessions in that list, after you review it. Every script now prints a result line. Step 5 checks whether a restart is needed and prints how to restart the machine safely. Verify the fix checks every machine, and a solution update that failed during installation can be resumed with one script once every machine is clean. |

## At a glance

| | |
| --- | --- |
| What happens | A deployment, update, upgrade, add node, or repair node operation stops because an extra copy of the Environment Checker PowerShell module could not be removed. |
| Applies to | Azure Local, all releases. The error text depends on the build; see [Symptoms](#symptoms). |
| Severity | High. The operation stays blocked until the extra copy is removed. Running virtual machines (VMs) are not affected. |
| Is it a network, hardware, or OEM problem? | No. The older error mentions `ValidateConnectivity`, but no network, firmware, or driver change is needed. |
| Who should act | An administrator with local administrator rights on every machine. |
| Time to fix | About 10 minutes, plus the time to retry the operation. |
| Impact | The fix removes only the extra copy and, after you review the list, stops only the PowerShell sessions that hold its files. Running VMs are not affected. No restart or maintenance window is needed, except in the rare case described in [Step 5](#step-5-check-whether-a-restart-is-needed). On a single-machine instance, that restart stops all VMs. |
| First step | Run the check in [Issue Validation](#issue-validation). |

## Overview

Azure Local ships its own copy of the Environment Checker PowerShell module, called the inbox module. The same module is also published on the PowerShell Gallery so that you can run readiness checks before deployment. The inbox version must be the one that loads, so before a deployment, update, upgrade, add node, or repair node operation, Azure Local tries to remove every other copy.

If you installed the module from the PowerShell Gallery, remove it after use (see [Prevention](#prevention)). When you troubleshoot a related issue, removing the extra copy first also makes the results easier to read.

## Symptoms

On rare occasions Azure Local cannot remove the extra copy, and the operation fails. The error depends on the build.

**Older builds** (example from Environment Checker 10.2504.0.2040): the operation stopped when `Uninstall-Module` could not find the copy.

```text
Type 'ValidateConnectivity' of Role 'EnvironmentValidator' raised an exception:
{
    "ExceptionType":  "text",
    "ErrorMessage":  "No match was found for the specified search criteria and module names \u0027AzStackHci.EnvironmentChecker\u0027.",
    "ExceptionStackTrace":  "at Uninstall-Module\u003cProcess\u003e, C:\\Program Files\\WindowsPowerShell\\Modules\\PowerShellGet\\2.2.5\\PSModule.psm1: line 12733\r\nat CleanEnvironmentValidator, C:\\NugetStore\\AzStackHci.EnvironmentChecker.Deploy.10.2504.0.2040\\content\\Classes\\EnvironmentValidator\\EnvironmentValidator.psm1: line 1230\r\nat EnvironmentValidatorImport, C:\\NugetStore\\AzStackHci.EnvironmentChecker.Deploy.10.2504.0.2040\\content\\Classes\\EnvironmentValidator\\EnvironmentValidator.psm1: line 784\r\nat RunSingleValidator, C:\\NugetStore\\AzStackHci.EnvironmentChecker.Deploy.10.2504.0.2040\\content\\Classes\\EnvironmentValidator\\EnvironmentValidator.psm1: line 806\r\nat ValidateConnectivity, C:\\NugetStore\\AzStackHci.EnvironmentChecker.Deploy.10.2504.0.2040\\content\\Classes\\EnvironmentValidator\\EnvironmentValidator.psm1: line 265\r\nat \u003cScriptBlock\u003e, C:\\CloudDeployment\\ECEngine\\InvokeInterfaceInternal.psm1: line 147\r\nat Invoke-EceInterfaceInternal, C:\\CloudDeployment\\ECEngine\\InvokeInterfaceInternal.psm1: line 142\r\nat \u003cScriptBlock\u003e, \u003cNo file\u003e: line 36"
}
```

**Newer builds** (confirmed on Environment Checker 10.2510.0.2100): Azure Local removes the extra copy by itself and fails only when it cannot delete the files, usually because a PowerShell session still has the module loaded. The error links to this article. For example (the version folder and file name vary):

```text
Unable to remove module files at C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\1.0.0. Error: Access to the path 'Microsoft.AzureStack.Observability.Standalone.dll' is denied.. Please follow https://aka.ms/envchkrversiontsg to remove the module manually before retrying lifecycle operation.
```

The system health check (`Invoke-SolutionUpdatePrecheck -SystemHealth`) runs the same cleanup on the machine where it runs. If the cleanup fails, the check can keep showing its previous result, so a passing health check does not prove that no extra copy is present. Use [Issue Validation](#issue-validation) instead.

## Cause

- **Older builds:** `Uninstall-Module` removes only modules that PowerShellGet installed and recorded. A copy that was copied in by hand, or whose install record is missing, still loads, but `Uninstall-Module` reports `No match was found`, and the operation stopped on that error.
- **Newer builds:** Azure Local ignores that error and deletes the version-number folder itself. It fails only when Windows does not allow the delete because a process still has files from the copy loaded.
- **The usual reason files are in use:** a PowerShell window or remote session that was used to run the Environment Checker from the PowerShell Gallery, for example for readiness checks before deployment, and was left open.

## Terms used in this article

| Term | Meaning |
| --- | --- |
| Environment Checker | The `AzStackHci.EnvironmentChecker` PowerShell module that runs the Azure Local readiness checks. |
| Inbox module | The copy that ships with Azure Local, in `C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker`. Its files sit directly in that folder, with no version-number subfolder. Never delete it. |
| External copy | Any other copy, for example one installed with `Install-Module` from the PowerShell Gallery. It lives in a version-number subfolder such as `...\AzStackHci.EnvironmentChecker\10.2411.0.12`. Deployment can also leave copies in version-number folders on some machines; they are removed the same way. A listed copy without the error in [Symptoms](#symptoms) is not by itself the cause of a failure. |
| PowerShell Gallery | The public online repository that `Install-Module` downloads PowerShell modules from. |
| In use | A process, usually a PowerShell window or remote session, still has files from the external copy loaded, so Windows does not allow them to be deleted. |
| Restart Manager | A built-in Windows component that reports which processes hold a file. [Step 1](#step-1-find-the-processes-that-hold-the-external-copys-files) uses it to name the exact processes that keep the external copy's files in use. |
| Result line | A line in capital letters, for example `NO_LOCKING_SESSIONS`, that a script in this article prints to report its result. Each step says what each result line means and what to do next. |

## Where you can see this problem

- **PowerShell on each machine:** the check in [Issue Validation](#issue-validation) lists the external copy.
- **Azure portal:** the failed step shows the error from [Symptoms](#symptoms), under **Deployments** for a deployment, or in the failure details under **Updates** for a solution update. This follows the documented behavior for failed steps; it was not reproduced during validation of this article.
- **Log files on the machine:** the deployment log for the failed step in `C:\CloudDeployment\Logs`, for example `EnvironmentValidatorFull.<date>.log` during deployment, contains `Externally installed module(s) found for AzStackHci.EnvironmentChecker` and `Attempting to uninstall` (older builds log `Uninstalling`), followed by the result or the error.
- **Windows event logs:** the error does not appear in the Windows event logs.
- **Cluster log:** the error does not appear in the cluster log (`Get-ClusterLog`), because no cluster resource changes state.
- **Failover Cluster Manager:** the error does not appear in Failover Cluster Manager.
- **Windows Admin Center:** the error does not appear in Windows Admin Center on a standalone host or in Windows Admin Center in the Azure portal.

## Before you start

- Open PowerShell with **Run as administrator** on each machine.
- Make sure no update or deployment step is running right now. If an update shows `Installing`, wait for it to stop before you change anything.
- Run [Issue Validation](#issue-validation) first. Continue only if it prints `EXTERNAL_COPY_FOUND` or `LEFTOVER_FOLDER_ONLY`.
- If it prints `INBOX_MODULE_MISSING`, stop and [contact Microsoft support](#when-to-contact-microsoft-support). Do not install a replacement.

## Issue Validation

Run this in an elevated PowerShell session on each machine. It changes nothing. It lists external copies, meaning module copies whose path contains their version number, and leftover version-number folders, and its first line is the result.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
$root = Join-Path $env:ProgramFiles "WindowsPowerShell\Modules\$moduleName"
$available = @(Get-Module -Name $moduleName -ListAvailable)
$external = @($available | Where-Object {$_.Path -like "*$($_.Version)*"})
$versionFolders = @(Get-ChildItem -Path $root -Directory -ErrorAction SilentlyContinue | Where-Object { $_.Name -as [version] })
if (@($available | Where-Object {$_.Path -notlike "*$($_.Version)*"}).Count -eq 0) {
    Write-Output "INBOX_MODULE_MISSING"
} elseif ($external.Count -gt 0) {
    Write-Output "EXTERNAL_COPY_FOUND"
} elseif ($versionFolders.Count -gt 0) {
    Write-Output "LEFTOVER_FOLDER_ONLY"
} else {
    Write-Output "NOT_AFFECTED"
}
$external | Format-Table Name, Version, Path -AutoSize
$versionFolders | ForEach-Object { "Version-number folder: $($_.FullName)" }
```

To check every machine at once, run this from one machine. On a deployed cluster, `Get-ClusterNode` lists the machines. Before the cluster exists (during deployment), set `$nodes` to the machine names instead. If PowerShell remoting to a machine fails, run the single-machine check above on that machine instead.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
$nodes = (Get-ClusterNode).Name
Invoke-Command -ComputerName $nodes -ScriptBlock {
    $name = $using:moduleName
    $root = Join-Path $env:ProgramFiles "WindowsPowerShell\Modules\$name"
    [pscustomobject]@{
        ExternalCopies      = @(Get-Module -Name $name -ListAvailable | Where-Object { $_.Path -like "*$($_.Version)*" } | ForEach-Object { $_.Path })
        VersionFolders      = @(Get-ChildItem -Path $root -Directory -ErrorAction SilentlyContinue | Where-Object { $_.Name -as [version] } | ForEach-Object { $_.FullName })
        AllAvailableModules = @(Get-Module -Name $name -ListAvailable | ForEach-Object { $_.Path })
    }
} | Format-List PSComputerName, ExternalCopies, VersionFolders, AllAvailableModules
```

How to read the output for each machine:

| Single-machine result | All-machines output | Meaning | Next step |
| --- | --- | --- | --- |
| `EXTERNAL_COPY_FOUND` | `ExternalCopies` lists a path | An external copy is present. | Follow Steps 1 to 6 in [Mitigation Details](#mitigation-details). |
| `LEFTOVER_FOLDER_ONLY` | `ExternalCopies` is empty but `VersionFolders` lists a folder | A leftover folder from a partly removed copy. It no longer loads. | Follow Steps 1 to 6. Step 4 has nothing to uninstall, and Step 6 deletes the folder. If Step 6 already told you to leave it until the next planned restart, no further action is needed now. |
| `NOT_AFFECTED` | Both are empty and `AllAvailableModules` shows only the inbox path | This machine is not affected. | None. |
| `INBOX_MODULE_MISSING` | `AllAvailableModules` does not show the inbox path | The inbox module is missing. | Stop and [contact Microsoft support](#when-to-contact-microsoft-support). Do not install a replacement. The cleanup scripts in Steps 4 and 6 remove nothing on a machine without the inbox module, and the retry script retries nothing. |

## Mitigation Details

Run these steps in order on each affected machine. Each script checks or changes only the machine it runs on, is safe to run when it has nothing to do, and prints a result line. None of the steps affects running VMs or needs a maintenance window, except the restart that [Step 5](#step-5-check-whether-a-restart-is-needed) can ask for on a single-machine instance.

### Step 1: Find the processes that hold the external copy's files

This step changes nothing. It asks Windows Restart Manager which processes hold files in the external copy's version-number folders, and lists each one with its owner, start time, and command line. Restart Manager reports these processes even though the module list that `Get-Process` shows does not include them, because PowerShell loads the module's .NET files in a way that is not listed there. Remote PowerShell sessions appear as `wsmprovhost`. Paste or run the whole block at once.

```powershell
& {
    $moduleRoot = Join-Path $env:ProgramFiles 'WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker'
    $sessionNames = 'powershell', 'pwsh', 'wsmprovhost'
    $files = @(Get-ChildItem -Path $moduleRoot -Directory -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -as [version] } |
        ForEach-Object { Get-ChildItem -LiteralPath $_.FullName -Recurse -File -Force -ErrorAction SilentlyContinue } |
        ForEach-Object { $_.FullName })
    $holderIds = @()
    $checkError = $null
    if ($files.Count -gt 0) {
        try {
            if (-not ('EnvCheckerFileHolders' -as [type])) {
                Add-Type -ErrorAction Stop -TypeDefinition @'
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Runtime.InteropServices;
using System.Text;
public static class EnvCheckerFileHolders {
    [StructLayout(LayoutKind.Sequential)]
    struct RM_UNIQUE_PROCESS { public int ProcessId; public System.Runtime.InteropServices.ComTypes.FILETIME StartTime; }
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct RM_PROCESS_INFO {
        public RM_UNIQUE_PROCESS Process;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 256)] public string AppName;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 64)] public string ServiceShortName;
        public int ApplicationType; public uint AppStatus; public uint SessionId;
        [MarshalAs(UnmanagedType.Bool)] public bool Restartable;
    }
    [DllImport("rstrtmgr.dll", CharSet = CharSet.Unicode)]
    static extern int RmStartSession(out uint session, int flags, StringBuilder key);
    [DllImport("rstrtmgr.dll")]
    static extern int RmEndSession(uint session);
    [DllImport("rstrtmgr.dll", CharSet = CharSet.Unicode)]
    static extern int RmRegisterResources(uint session, uint fileCount, string[] files, uint appCount, RM_UNIQUE_PROCESS[] apps, uint serviceCount, string[] services);
    [DllImport("rstrtmgr.dll")]
    static extern int RmGetList(uint session, out uint needed, ref uint count, [In, Out] RM_PROCESS_INFO[] info, ref uint reasons);
    public static int[] GetProcessIds(string[] files) {
        uint session;
        int rc = RmStartSession(out session, 0, new StringBuilder(64));
        if (rc != 0) { throw new Win32Exception(rc); }
        try {
            rc = RmRegisterResources(session, (uint)files.Length, files, 0, null, 0, null);
            if (rc != 0) { throw new Win32Exception(rc); }
            for (int attempt = 0; attempt < 5; attempt++) {
                uint needed = 0, count = 0, reasons = 0;
                rc = RmGetList(session, out needed, ref count, null, ref reasons);
                if (rc == 0) { return new int[0]; }
                if (rc != 234) { throw new Win32Exception(rc); }
                RM_PROCESS_INFO[] info = new RM_PROCESS_INFO[needed];
                count = needed;
                rc = RmGetList(session, out needed, ref count, info, ref reasons);
                if (rc == 0) {
                    List<int> ids = new List<int>();
                    for (int i = 0; i < count; i++) { ids.Add(info[i].Process.ProcessId); }
                    return ids.ToArray();
                }
                if (rc != 234) { throw new Win32Exception(rc); }
            }
            throw new Win32Exception(234);
        }
        finally { RmEndSession(session); }
    }
}
'@
            }
            $holderIds = @([EnvCheckerFileHolders]::GetProcessIds([string[]]$files))
        }
        catch { $checkError = $_.Exception.Message }
    }
    $serviceIds = @(Get-CimInstance -ClassName Win32_Service -Filter 'ProcessId > 0' -ErrorAction SilentlyContinue | ForEach-Object { [int]$_.ProcessId })
    $rows = @(foreach ($id in @($holderIds | Sort-Object -Unique)) {
        $process = Get-CimInstance -ClassName Win32_Process -Filter "ProcessId = $id" -ErrorAction SilentlyContinue
        if (-not $process) { continue }
        $owner = Invoke-CimMethod -InputObject $process -MethodName GetOwner -ErrorAction SilentlyContinue
        $isSession = ($sessionNames -contains ($process.Name -replace '\.exe$', '')) -and ($serviceIds -notcontains $id)
        [pscustomobject]@{
            ProcessId   = $id
            Name        = $process.Name
            Owner       = if ($owner.User) { "$($owner.Domain)\$($owner.User)" } else { 'unknown' }
            StartTime   = $process.CreationDate
            CanStop     = $isSession -and ($id -ne $PID)
            NextStep    = if ($id -eq $PID) { 'This PowerShell window holds the files. Close it, open a new window, and start again at Step 1.' } elseif ($isSession) { 'Step 2 stops it.' } else { 'Step 2 does not stop it.' }
            CommandLine = $process.CommandLine
        }
    })
    if ($checkError) {
        Write-Output 'LOCK_CHECK_UNAVAILABLE'
        Write-Output "Restart Manager could not be used in this PowerShell session: $checkError"
        Write-Output 'These PowerShell processes are running. Any of them could hold the files:'
        Get-CimInstance -ClassName Win32_Process -ErrorAction SilentlyContinue |
            Where-Object { $sessionNames -contains ($_.Name -replace '\.exe$', '') } |
            Format-Table ProcessId, Name, CreationDate, CommandLine -AutoSize -Wrap
    }
    elseif ($rows.Count -gt 0) {
        Write-Output 'LOCKING_SESSIONS_FOUND'
        $rows | Format-List ProcessId, Name, Owner, StartTime, NextStep, CommandLine
    }
    else {
        Write-Output 'NO_LOCKING_SESSIONS'
        Write-Output "No process holds files in the external copy's version-number folders ($($files.Count) files checked)."
    }
}
```

The first line of the output is the result:

| Result | Meaning | Next step |
| --- | --- | --- |
| `LOCKING_SESSIONS_FOUND` | The listed processes hold files from the external copy. `NextStep` says which ones Step 2 stops. Check each `Owner` and `CommandLine` before you continue. | Go to Step 2. |
| `NO_LOCKING_SESSIONS` | No process holds the files. | Go to Step 2. It has nothing to stop. |
| `LOCK_CHECK_UNAVAILABLE` | Windows did not allow the Restart Manager check to run in this PowerShell session, so the script lists every PowerShell process instead, and Step 2 stops nothing. | Close the PowerShell windows and remote sessions that you or your team opened to run the Environment Checker, then go to Step 2. |

### Step 2: Stop the PowerShell sessions that hold the files

Run this after you have read the Step 1 list. The script runs the same check as Step 1 and stops only the processes that the check finds at that moment and that Step 1 marks `Step 2 stops it.` When no PowerShell session holds the files, it stops nothing. Paste or run the whole block at once.

**Changes:** stops each process that holds the external copy's files, is named `powershell`, `pwsh`, or `wsmprovhost`, and does not run a Windows service. Work that is not saved in those sessions is lost, and stopping a `wsmprovhost` process ends the remote PowerShell sessions that it hosts. It does not stop the PowerShell window that you run it from, any service, or any other process. Running VMs are not affected.

```powershell
& {
    $moduleRoot = Join-Path $env:ProgramFiles 'WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker'
    $sessionNames = 'powershell', 'pwsh', 'wsmprovhost'
    $files = @(Get-ChildItem -Path $moduleRoot -Directory -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -as [version] } |
        ForEach-Object { Get-ChildItem -LiteralPath $_.FullName -Recurse -File -Force -ErrorAction SilentlyContinue } |
        ForEach-Object { $_.FullName })
    $holderIds = @()
    $checkError = $null
    if ($files.Count -gt 0) {
        try {
            if (-not ('EnvCheckerFileHolders' -as [type])) {
                Add-Type -ErrorAction Stop -TypeDefinition @'
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Runtime.InteropServices;
using System.Text;
public static class EnvCheckerFileHolders {
    [StructLayout(LayoutKind.Sequential)]
    struct RM_UNIQUE_PROCESS { public int ProcessId; public System.Runtime.InteropServices.ComTypes.FILETIME StartTime; }
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct RM_PROCESS_INFO {
        public RM_UNIQUE_PROCESS Process;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 256)] public string AppName;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 64)] public string ServiceShortName;
        public int ApplicationType; public uint AppStatus; public uint SessionId;
        [MarshalAs(UnmanagedType.Bool)] public bool Restartable;
    }
    [DllImport("rstrtmgr.dll", CharSet = CharSet.Unicode)]
    static extern int RmStartSession(out uint session, int flags, StringBuilder key);
    [DllImport("rstrtmgr.dll")]
    static extern int RmEndSession(uint session);
    [DllImport("rstrtmgr.dll", CharSet = CharSet.Unicode)]
    static extern int RmRegisterResources(uint session, uint fileCount, string[] files, uint appCount, RM_UNIQUE_PROCESS[] apps, uint serviceCount, string[] services);
    [DllImport("rstrtmgr.dll")]
    static extern int RmGetList(uint session, out uint needed, ref uint count, [In, Out] RM_PROCESS_INFO[] info, ref uint reasons);
    public static int[] GetProcessIds(string[] files) {
        uint session;
        int rc = RmStartSession(out session, 0, new StringBuilder(64));
        if (rc != 0) { throw new Win32Exception(rc); }
        try {
            rc = RmRegisterResources(session, (uint)files.Length, files, 0, null, 0, null);
            if (rc != 0) { throw new Win32Exception(rc); }
            for (int attempt = 0; attempt < 5; attempt++) {
                uint needed = 0, count = 0, reasons = 0;
                rc = RmGetList(session, out needed, ref count, null, ref reasons);
                if (rc == 0) { return new int[0]; }
                if (rc != 234) { throw new Win32Exception(rc); }
                RM_PROCESS_INFO[] info = new RM_PROCESS_INFO[needed];
                count = needed;
                rc = RmGetList(session, out needed, ref count, info, ref reasons);
                if (rc == 0) {
                    List<int> ids = new List<int>();
                    for (int i = 0; i < count; i++) { ids.Add(info[i].Process.ProcessId); }
                    return ids.ToArray();
                }
                if (rc != 234) { throw new Win32Exception(rc); }
            }
            throw new Win32Exception(234);
        }
        finally { RmEndSession(session); }
    }
}
'@
            }
            $holderIds = @([EnvCheckerFileHolders]::GetProcessIds([string[]]$files))
        }
        catch { $checkError = $_.Exception.Message }
    }
    $serviceIds = @(Get-CimInstance -ClassName Win32_Service -Filter 'ProcessId > 0' -ErrorAction SilentlyContinue | ForEach-Object { [int]$_.ProcessId })
    $rows = @(foreach ($id in @($holderIds | Sort-Object -Unique)) {
        $process = Get-CimInstance -ClassName Win32_Process -Filter "ProcessId = $id" -ErrorAction SilentlyContinue
        if (-not $process) { continue }
        $owner = Invoke-CimMethod -InputObject $process -MethodName GetOwner -ErrorAction SilentlyContinue
        $isSession = ($sessionNames -contains ($process.Name -replace '\.exe$', '')) -and ($serviceIds -notcontains $id)
        [pscustomobject]@{
            ProcessId   = $id
            Name        = $process.Name
            Owner       = if ($owner.User) { "$($owner.Domain)\$($owner.User)" } else { 'unknown' }
            StartTime   = $process.CreationDate
            CanStop     = $isSession -and ($id -ne $PID)
            NextStep    = if ($id -eq $PID) { 'This PowerShell window holds the files. Close it, open a new window, and start again at Step 1.' } elseif ($isSession) { 'Step 2 stops it.' } else { 'Step 2 does not stop it.' }
            CommandLine = $process.CommandLine
        }
    })
    if ($checkError) {
        Write-Output 'LOCK_CHECK_UNAVAILABLE'
        Write-Output "Nothing was stopped. Restart Manager could not be used in this PowerShell session: $checkError"
    }
    else {
        $results = @(foreach ($row in @($rows | Where-Object { $_.CanStop })) {
            try {
                Stop-Process -Id $row.ProcessId -Force -ErrorAction Stop
                Wait-Process -Id $row.ProcessId -Timeout 30 -ErrorAction SilentlyContinue
                "Stopped $($row.Name), process ID $($row.ProcessId), owner $($row.Owner), started $($row.StartTime)."
            }
            catch { "Could not stop $($row.Name), process ID $($row.ProcessId): $($_.Exception.Message)" }
        })
        if (@($results | Where-Object { $_ -like 'Stopped *' }).Count -gt 0) { Write-Output 'SESSIONS_STOPPED' } else { Write-Output 'NO_SESSIONS_STOPPED' }
        $results
        foreach ($row in @($rows | Where-Object { -not $_.CanStop })) { "Left running: $($row.Name), process ID $($row.ProcessId). $($row.NextStep)" }
    }
}
```

| Result | Meaning | Next step |
| --- | --- | --- |
| `SESSIONS_STOPPED` | The script stopped the PowerShell sessions it lists. | Go to Step 3. |
| `NO_SESSIONS_STOPPED` | No PowerShell session held the files when the script ran, or the ones it found had already ended. | Go to Step 3. |
| `LOCK_CHECK_UNAVAILABLE` | The check could not run, so nothing was stopped. | Go to Step 3. |

### Step 3: Confirm that the files are released

Run the Step 1 script again. This changes nothing. Then go to Step 4.

- **`NO_LOCKING_SESSIONS`:** the files are released.
- **`LOCKING_SESSIONS_FOUND`:** a process still holds files. Either it is not a PowerShell session, or it is a session that started after Step 2 ran. The cleanup in Step 4 removes everything it can, and Step 5 tells you whether a restart is needed.
- **`LOCK_CHECK_UNAVAILABLE`:** the check still cannot run in this PowerShell session.

### Step 4: Run the cleanup script

**Changes:** unloads, uninstalls, and deletes only copies in version-number folders. The inbox module and running VMs are not affected. When the inbox module is missing, the script removes nothing.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
if (-not (Get-Module -Name $moduleName -ListAvailable | Where-Object {$_.Path -notlike "*$($_.Version)*"})) {
    Write-Output "INBOX_MODULE_MISSING"
    Write-Output "The inbox module is missing, so nothing was removed."
} else {
    # Unload any previously loaded modules
    $loadedModule = Get-Module -Name $moduleName | Where-Object {$_.Path -like "*$($_.Version)*"}
    if ($loadedModule)
    {
        Write-Host "Loaded module(s) found for $($loadedModule.Name):"
        Write-Host ($loadedModule | Format-Table Name, Version, Path -AutoSize | Out-String)
        Write-Host "Unloading"
        $loadedModule | Remove-Module -Force
    }

    # Uninstall any modules installed from external
    $installedModule = Get-Module -Name $moduleName -ListAvailable | Where-Object {$_.Path -like "*$($_.Version)*"}
    if ($installedModule)
    {
        Write-Host "Externally installed module(s) found for $($installedModule.Name):"
        Write-Host ($installedModule | Format-Table Name, Version, Path -AutoSize | Out-String)
        Write-Host "Uninstalling module(s)"
        $installedModule | Uninstall-Module -Force -ErrorAction SilentlyContinue
        foreach ($im in $installedModule)
        {
            Write-Host "Checking module still exists on disk..."
            if (Test-Path $im.ModuleBase -ErrorAction SilentlyContinue) {
                Write-Host "Removing old module files $($im.ModuleBase)"
                Remove-Item -Path $im.ModuleBase -Recurse -Force -ErrorAction SilentlyContinue -ErrorVariable removeErrors
                if ($removeErrors) {
                    Write-Host "Some files in $($im.ModuleBase) are still in use and were not removed: $($removeErrors[0].Exception.Message)"
                }
            }else {
                Write-Host "Old module files not found at $($im.ModuleBase)"
            }
        }
    } else {
        Write-Host "Nothing to uninstall"
    }

    # Check final module is our one.
    $FinalModule = Get-Module -Name $moduleName -ListAvailable
    if (-not ($FinalModule | Where-Object {$_.Path -notlike "*$($_.Version)*"})) {
        Write-Output "INBOX_MODULE_MISSING"
    } elseif ($FinalModule | Where-Object {$_.Path -like "*$($_.Version)*"}) {
        Write-Output "CLEANUP_EXTERNAL_REMAINS"
    } else {
        Write-Output "CLEANUP_EXTERNAL_REMOVED"
    }
    Write-Host "Final installed module:"
    Write-Host ($FinalModule | Format-Table Name, Version, Path -AutoSize | Out-String)
}
```

The result line appears just before the final module list:

| Result | Meaning | Next step |
| --- | --- | --- |
| `CLEANUP_EXTERNAL_REMOVED` | Only the inbox module is listed, at `C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\AzStackHci.EnvironmentChecker.psd1`, with no version-number folder in its path. If the script reported files that are still in use, the copy no longer loads because its module manifest is gone. | Go to Step 5. |
| `CLEANUP_EXTERNAL_REMAINS` | The external copy is still listed, so the operation stays blocked. This is rare: a process that Step 2 does not stop still holds the copy's module manifest. | Go to Step 5. |
| `INBOX_MODULE_MISSING` | The inbox module is not listed, so the script removed nothing. | [Contact Microsoft support](#when-to-contact-microsoft-support). Do not install a replacement. The later steps change nothing on this machine, and the retry script retries nothing. |

To run the cleanup on every machine from one session, save the script above as `Remove-ExternalEnvChecker.ps1` and run:

```powershell
$nodes = (Get-ClusterNode).Name   # Before the cluster exists, set $nodes to the machine names.
Invoke-Command -ComputerName $nodes -FilePath .\Remove-ExternalEnvChecker.ps1
```

### Step 5: Check whether a restart is needed

This step changes nothing. A restart is needed only in the rare case where Step 4 printed `CLEANUP_EXTERNAL_REMAINS`, because a process that Step 2 does not stop still holds the copy's module manifest. The script checks for that case again and, when a restart is needed, prints how to restart this machine safely.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
$remaining = @(Get-Module -Name $moduleName -ListAvailable | Where-Object {$_.Path -like "*$($_.Version)*"})
if ($remaining.Count -eq 0) {
    Write-Output "NO_RESTART_NEEDED"
} else {
    Write-Output "RESTART_REQUIRED"
    $remaining | ForEach-Object { "Still listed: $($_.Path)" }
    $machineCount = @(Get-ClusterNode -ErrorAction SilentlyContinue).Count
    if ($machineCount -gt 1) {
        "This machine is in a cluster of $machineCount machines. Pause and drain it before the restart, and resume it afterward: https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance"
    } elseif ($machineCount -eq 1) {
        "This is a single-machine instance. The restart stops all running VMs, so plan a maintenance window and shut down or save the VMs first."
    } else {
        "The cluster does not exist yet, so restart this machine normally."
    }
    "After the restart, run Steps 4, 5, and 6 again."
}
```

| Result | Meaning | What to do |
| --- | --- | --- |
| `NO_RESTART_NEEDED` | The external copy no longer loads. | Go to Step 6. |
| `RESTART_REQUIRED` | The external copy is still listed, and only a restart releases the files that hold it. The script prints the maintenance procedure that fits this machine's cluster size. On a single-machine instance, a restart stops all running VMs. Until the restart, Step 6 deletes nothing and the retry script retries nothing. | Follow the printed guidance. |

### Step 6: Delete leftover folders

**Changes:** deletes only subfolders whose name is a version number. The inbox files are not in such a subfolder and are not touched. The script deletes nothing when the inbox module is missing or while an external copy still loads.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
$root = Join-Path $env:ProgramFiles "WindowsPowerShell\Modules\$moduleName"
$available = @(Get-Module -Name $moduleName -ListAvailable)
if (@($available | Where-Object {$_.Path -notlike "*$($_.Version)*"}).Count -eq 0) {
    Write-Output "INBOX_MODULE_MISSING"
    Write-Output "The inbox module is missing, so nothing was deleted."
} elseif (@($available | Where-Object {$_.Path -like "*$($_.Version)*"}).Count -gt 0) {
    Write-Output "RESTART_REQUIRED"
    Write-Output "An external copy still loads, so nothing was deleted. Step 5 prints how to restart this machine."
} else {
    $leftover = @(Get-ChildItem -Path $root -Directory -ErrorAction SilentlyContinue | Where-Object { $_.Name -as [version] })
    $inUse = @()
    foreach ($folder in $leftover) {
        Write-Host "Removing leftover folder $($folder.FullName)"
        Remove-Item -LiteralPath $folder.FullName -Recurse -Force -ErrorAction SilentlyContinue
        if (Test-Path -LiteralPath $folder.FullName) { $inUse += $folder.FullName }
    }
    if ($leftover.Count -eq 0) {
        Write-Output "NO_LEFTOVER_FOLDERS"
    } elseif ($inUse.Count -gt 0) {
        Write-Output "LEFTOVER_FOLDER_IN_USE"
        $inUse | ForEach-Object { "Still in use, left in place: $_" }
    } else {
        Write-Output "LEFTOVER_FOLDERS_REMOVED"
    }
}
```

| Result | Meaning | What to do |
| --- | --- | --- |
| `LEFTOVER_FOLDERS_REMOVED` or `NO_LEFTOVER_FOLDERS` | No version-number folder is left. | Go to [Verify the fix](#verify-the-fix). |
| `LEFTOVER_FOLDER_IN_USE` | A process that Step 2 does not stop still holds files in the folder. The folder's module manifest is already gone, so it does not load and does not block the operation. | Leave the folder for now and go to [Verify the fix](#verify-the-fix). Run this step again after the machine's next planned restart, rather than restarting the machine for this alone. |
| `RESTART_REQUIRED` | An external copy still loads, so nothing was deleted. | See Step 5. |
| `INBOX_MODULE_MISSING` | The inbox module is not listed, so nothing was deleted. | [Contact Microsoft support](#when-to-contact-microsoft-support). |

## Verify the fix

Run this on one machine. It changes nothing. It checks every machine of the cluster and prints a result line. Before the cluster exists (during deployment), it checks only the machine it runs on, so run it on each machine. Paste or run the whole block at once.

```powershell
& {
    $moduleName = "AzStackHci.EnvironmentChecker"
    $checkMachine = {
        param($Name)
        $root = Join-Path $env:ProgramFiles "WindowsPowerShell\Modules\$Name"
        $found = @(Get-Module -Name $Name -ListAvailable)
        [pscustomobject]@{
            ExternalCopies = @($found | Where-Object { $_.Path -like "*$($_.Version)*" } | ForEach-Object { $_.Path })
            InboxModule    = @($found | Where-Object { $_.Path -notlike "*$($_.Version)*" } | ForEach-Object { $_.Path })
            VersionFolders = @(Get-ChildItem -Path $root -Directory -ErrorAction SilentlyContinue | Where-Object { $_.Name -as [version] } | ForEach-Object { $_.FullName })
        }
    }
    $clusterExists = $true
    $nodes = @($env:COMPUTERNAME)
    try { $nodes = @((Get-ClusterNode -ErrorAction Stop).Name) } catch { $clusterExists = $false }
    $results = @(foreach ($node in $nodes) {
        try {
            if ($node -eq $env:COMPUTERNAME) { $state = & $checkMachine $moduleName }
            else { $state = Invoke-Command -ComputerName $node -ScriptBlock $checkMachine -ArgumentList $moduleName -ErrorAction Stop }
            [pscustomobject]@{
                Machine        = $node
                Clean          = ($state.ExternalCopies.Count -eq 0) -and ($state.InboxModule.Count -ge 1)
                ExternalCopies = ($state.ExternalCopies -join '; ')
                InboxModule    = ($state.InboxModule -join '; ')
                VersionFolders = ($state.VersionFolders -join '; ')
            }
        } catch {
            [pscustomobject]@{ Machine = $node; Clean = $false; ExternalCopies = "Could not be checked: $($_.Exception.Message)"; InboxModule = ''; VersionFolders = '' }
        }
    })
    if (@($results | Where-Object { -not $_.Clean }).Count -gt 0) { Write-Output "NOT_ALL_MACHINES_CLEAN" }
    elseif ($clusterExists) { Write-Output "ALL_MACHINES_CLEAN" }
    else { Write-Output "THIS_MACHINE_CLEAN" }
    $results | Format-List Machine, Clean, ExternalCopies, InboxModule, VersionFolders
}
```

| Result | Meaning | What to do |
| --- | --- | --- |
| `ALL_MACHINES_CLEAN` | No machine has an external copy, and every machine has the inbox module at `C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\AzStackHci.EnvironmentChecker.psd1`. A `VersionFolders` entry is a folder you deliberately left in place under [Step 6](#step-6-delete-leftover-folders) until the next planned restart; it does not load. | Go to [Retry the failed operation](#retry-the-failed-operation). |
| `THIS_MACHINE_CLEAN` | The cluster does not exist yet, and this machine is clean. | Run this on every other machine. When each one prints `THIS_MACHINE_CLEAN`, go to [Retry the failed operation](#retry-the-failed-operation). |
| `NOT_ALL_MACHINES_CLEAN` | At least one machine still has an external copy, is missing the inbox module, or could not be checked. | Run Steps 1 to 6 on each machine where `Clean` is `False`. If a machine could not be checked, run this on that machine. |

Do not use the system health check to confirm this fix. It can keep showing a previous result, and it runs the cleanup only on the machine where it runs.

## Retry the failed operation

Run this script on one machine. It first runs the same check as [Verify the fix](#verify-the-fix), and it retries nothing unless every machine is clean. It then resumes a solution update that failed during installation. For any other operation, it prints how to retry it. Paste or run the whole block at once.

**Changes:** resumes the update only when every machine is clean and exactly one update is in the `InstallationFailed` state. The update continues from the step that failed. In every other case the script changes nothing.

**Plan for:** the rest of the update installation, including any machine restarts that the update performs, as when you first started it.

```powershell
& {
    $moduleName = "AzStackHci.EnvironmentChecker"
    $checkMachine = {
        param($Name)
        $found = @(Get-Module -Name $Name -ListAvailable)
        [pscustomobject]@{
            ExternalCopies = @($found | Where-Object { $_.Path -like "*$($_.Version)*" }).Count
            InboxModule    = @($found | Where-Object { $_.Path -notlike "*$($_.Version)*" }).Count
        }
    }
    $clusterExists = $true
    $nodes = @($env:COMPUTERNAME)
    try { $nodes = @((Get-ClusterNode -ErrorAction Stop).Name) } catch { $clusterExists = $false }
    $notClean = @(foreach ($node in $nodes) {
        try {
            if ($node -eq $env:COMPUTERNAME) { $state = & $checkMachine $moduleName }
            else { $state = Invoke-Command -ComputerName $node -ScriptBlock $checkMachine -ArgumentList $moduleName -ErrorAction Stop }
            if ($state.ExternalCopies -gt 0 -or $state.InboxModule -eq 0) { "$node still has an external copy, or its inbox module is missing." }
        } catch { "$node could not be checked: $($_.Exception.Message)" }
    })
    if ($notClean.Count -gt 0) {
        Write-Output "RETRY_BLOCKED"
        $notClean | ForEach-Object { "Nothing was retried. $_" }
    } elseif (-not $clusterExists) {
        Write-Output "CLUSTER_NOT_DEPLOYED"
        Write-Output "The cluster does not exist yet, so only this machine was checked. When Verify the fix prints THIS_MACHINE_CLEAN on every machine, resume the deployment: open your Azure Local instance, go to Deployments, and select Resume deployment."
    } else {
        $updates = $null
        $updateError = $null
        try { $updates = @(Get-SolutionUpdate -ErrorAction Stop) }
        catch { $updateError = $_.Exception.Message }
        if ($null -eq $updates) {
            Write-Output "UPDATE_SERVICE_NOT_AVAILABLE"
            Write-Output "Could not read solution updates: $updateError"
            Write-Output "If a deployment failed, resume it: open your Azure Local instance, go to Deployments, and select Resume deployment."
        } else {
            $failed = @($updates | Where-Object { $_.State -eq 'InstallationFailed' })
            if ($failed.Count -eq 1) {
                try {
                    $run = $failed[0] | Start-SolutionUpdate -ErrorAction Stop
                    Write-Output "SOLUTION_UPDATE_RESUMED"
                    Write-Output "Resumed $($failed[0].DisplayName). Update run: $run"
                } catch {
                    $state = (Get-SolutionUpdate -Id $failed[0].ResourceId -ErrorAction SilentlyContinue).State
                    if ($state -eq 'Installing') {
                        Write-Output "SOLUTION_UPDATE_RESUMED"
                        Write-Output "$($failed[0].DisplayName) is already installing again."
                    } else {
                        Write-Output "SOLUTION_UPDATE_NOT_RESUMED"
                        Write-Output "Start-SolutionUpdate did not resume $($failed[0].DisplayName): $($_.Exception.Message)"
                    }
                }
            } elseif ($failed.Count -gt 1) {
                Write-Output "SOLUTION_UPDATE_NOT_RESUMED"
                Write-Output "More than one update is in the InstallationFailed state, so none was resumed."
            } else {
                Write-Output "NO_UPDATE_TO_RESUME"
                Write-Output "No solution update failed during installation. Retry the operation that failed the same way you started it. An update that shows HealthCheckFailed failed in its readiness check: select Try again on it, or run the same Start-SolutionUpdate command again."
            }
            Get-SolutionUpdate -ErrorAction SilentlyContinue | Format-Table DisplayName, Version, State -AutoSize
        }
    }
}
```

| Result | Meaning | What to do |
| --- | --- | --- |
| `SOLUTION_UPDATE_RESUMED` | The update is installing again. Its `State` in the list is `Installing`. | Follow its progress under **Updates** for your Azure Local instance. It should continue past the step that failed. |
| `NO_UPDATE_TO_RESUME` | No solution update failed during installation, so nothing was resumed. The script prints how to retry other operations, including an update that failed in its readiness check (`HealthCheckFailed`). | Retry the operation that failed the same way you started it. If you started a readiness check with `Start-SolutionUpdate -PrepareOnly`, use the same command again. |
| `CLUSTER_NOT_DEPLOYED` | The cluster does not exist yet, so the failed operation is a deployment. Only this machine was checked. | Confirm with [Verify the fix](#verify-the-fix) that every machine prints `THIS_MACHINE_CLEAN`, then resume the deployment as described in [Resume deployment](https://learn.microsoft.com/azure/azure-local/deploy/deploy-via-portal#resume-deployment). |
| `UPDATE_SERVICE_NOT_AVAILABLE` | The cluster exists, but the solution update service did not answer. This can happen when a deployment failed after the cluster was created. | For a deployment, see [Resume deployment](https://learn.microsoft.com/azure/azure-local/deploy/deploy-via-portal#resume-deployment). On a deployed system, see [Troubleshoot solution updates for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-troubleshooting-23h2). |
| `SOLUTION_UPDATE_NOT_RESUMED` | The update did not start again. The script prints the reason. | Retry the update the same way you started it. If it still does not start, [contact Microsoft support](#when-to-contact-microsoft-support). |
| `RETRY_BLOCKED` | A machine still has an external copy, is missing the inbox module, or could not be checked. The script names each one. | Run Steps 1 to 6 on each named machine, then run this script again. If a machine could not be checked, run [Verify the fix](#verify-the-fix) on that machine. |

## Rollback

Normally there is nothing to roll back. A copy installed from the PowerShell Gallery was not part of Azure Local, and a copy left behind by deployment is removed by Azure Local itself before each operation. If you need the standalone Environment Checker from the PowerShell Gallery again, install it only after the operation completes, and remove it again before the next one, as described in [Prevention](#prevention). If you paused and drained a machine, resume it as described in the pause and drain procedure. Sessions that Step 2 stopped cannot be restored; open new PowerShell sessions if you still need them.

## When to contact Microsoft support

Open a support request if any of these is true:

- A check prints `INBOX_MODULE_MISSING`, or the inbox module is missing from `AllAvailableModules`.
- Step 4 still prints `CLEANUP_EXTERNAL_REMAINS` after you restarted the machine and ran Step 4 again.
- A leftover version-number folder still cannot be deleted after the machine has restarted.
- The same error returns after the retry while Issue Validation shows no external copy.
- The solution update does not start again after **Try again** in the Azure portal.

Include this information:

- The output of the all-machines Issue Validation command before the steps you ran, and of [Verify the fix](#verify-the-fix) after them.
- The full output of Step 1 and of the cleanup script from every machine.
- The exact error text and the time it occurred, from the failed deployment or update step in the Azure portal.
- The deployment log from `C:\CloudDeployment\Logs` that contains `Externally installed module(s) found for AzStackHci.EnvironmentChecker`.
- The Azure Local version and the version of the inbox Environment Checker module.

## Prevention

Azure Local already includes the Environment Checker, so you do not need the PowerShell Gallery copy on its machines. If you install it from the PowerShell Gallery to run readiness checks, as described in [Assess your environment with the Environment Checker](https://learn.microsoft.com/azure/azure-local/manage/use-environment-checker), close that PowerShell window when you finish. Then, before you start a deployment, update, upgrade, add node, or repair node operation, remove the gallery copy from every machine with the cleanup script in [Step 4](#step-4-run-the-cleanup-script). Closing the window first matters: the module's .NET files stay in use until the PowerShell process that loaded them ends.

## Related documentation

- [Assess your environment with the Environment Checker](https://learn.microsoft.com/azure/azure-local/manage/use-environment-checker)
- [Troubleshoot solution updates for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-troubleshooting-23h2)
- [Understand update phases for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-phases-23h2)
- [Deploy Azure Local using the Azure portal](https://learn.microsoft.com/azure/azure-local/deploy/deploy-via-portal)
- [Suspend and resume Azure Local machines for planned maintenance](https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance)
