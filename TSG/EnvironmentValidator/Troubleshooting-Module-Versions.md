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
| 2026-11-26 | Step 1 now uses Windows Restart Manager to list the exact processes that hold the external copy's files, and Step 2 stops only the PowerShell sessions in that list, after you review it. The cleanup and leftover-folder scripts now print a one-line result, the restart for files that stay in use is its own step, and a solution update that failed during installation can be resumed with one command after the fix is verified. |

## At a glance

| | |
| --- | --- |
| What happens | A deployment, update, upgrade, add node, or repair node operation stops because an extra copy of the Environment Checker PowerShell module could not be removed. |
| Applies to | Azure Local, all releases. The error text depends on the build; see [Symptoms](#symptoms). |
| Severity | High. The operation stays blocked until the extra copy is removed. Running virtual machines (VMs) are not affected. |
| Is it a network, hardware, or OEM problem? | No. The older error mentions `ValidateConnectivity`, but no network, firmware, or driver change is needed. |
| Who should act | An administrator with local administrator rights on every machine. |
| Time to fix | About 10 minutes, plus the time to retry the operation. |
| Impact | The fix removes only the extra copy and, after you review the list, stops only the PowerShell sessions that hold its files. Running VMs are not affected. No restart or maintenance window is needed, except in the rare case described in [Step 5](#step-5-restart-the-machine-if-the-copy-is-still-listed). On a single-machine instance, that restart stops all VMs. |
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
| Result line | The first line that a script in this article prints, in capital letters, for example `NO_LOCKING_SESSIONS`. Each step says what to do for each result line. |

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
- Run [Issue Validation](#issue-validation) first. Continue only if it lists an external copy or a leftover version-number folder.
- If the inbox module is missing from the results, stop and [contact Microsoft support](#when-to-contact-microsoft-support). Do not install a replacement.

## Issue Validation

Run this in an elevated PowerShell session on each machine. It changes nothing; it lists external copies, meaning module copies whose path contains their version number.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
Get-Module -Name $moduleName -ListAvailable | Where-Object {$_.Path -like "*$($_.Version)*"}
```

- **Affected:** one or more modules are listed, each with a version-number folder in its path.
- **Not affected:** no output.

To check every machine at once, run this from one machine. On a deployed cluster, `Get-ClusterNode` lists the machines. Before the cluster exists (during deployment), set `$nodes` to the machine names instead. If PowerShell remoting to a machine fails, run the single-machine command above on that machine instead.

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

| Result | Meaning | Next step |
| --- | --- | --- |
| `ExternalCopies` lists a path | An external copy is present. | Follow Steps 1 to 6 in [Mitigation Details](#mitigation-details). |
| `ExternalCopies` is empty but `VersionFolders` lists a folder | A leftover folder from a partly removed copy. It no longer loads. | [Step 6](#step-6-delete-leftover-folders). If Step 6 already told you to leave it until the next planned restart, no further action is needed now. |
| Both are empty and `AllAvailableModules` shows only the inbox path | This machine is not affected. | None. |
| `AllAvailableModules` does not show the inbox path | The inbox module is missing. | Stop and [contact Microsoft support](#when-to-contact-microsoft-support). |

## Mitigation Details

Run these steps on each affected machine. None of them affects running VMs or needs a maintenance window, except the restart in [Step 5](#step-5-restart-the-machine-if-the-copy-is-still-listed) on a single-machine instance.

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
            NextStep    = if ($id -eq $PID) { 'This PowerShell window holds the files. Close it, open a new window, and start again at Step 1.' } elseif ($isSession) { 'Step 2 stops it.' } else { 'Step 2 does not stop it. Continue with Step 4.' }
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
| `LOCKING_SESSIONS_FOUND` | The listed processes hold files from the external copy. `NextStep` says which ones Step 2 stops. | Read the list, then go to Step 2. |
| `NO_LOCKING_SESSIONS` | No process holds the files. | Go to Step 4. |
| `LOCK_CHECK_UNAVAILABLE` | Windows did not allow the Restart Manager check to run in this PowerShell session, so the script lists every PowerShell process instead. Step 2 cannot stop sessions in this case. | Close the PowerShell windows and remote sessions that you or your team opened to run the Environment Checker, then go to Step 4. |

If a listed session belongs to an Azure Local operation that is still running, stop here and wait for that operation to finish, as described in [Before you start](#before-you-start).

### Step 2: Stop the PowerShell sessions that hold the files

Run this only when Step 1 printed `LOCKING_SESSIONS_FOUND`, and only after you have read its list. The script runs the same check as Step 1 and stops only the processes that the check finds at that moment and that Step 1 marks `Step 2 stops it.` Paste or run the whole block at once.

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
            NextStep    = if ($id -eq $PID) { 'This PowerShell window holds the files. Close it, open a new window, and start again at Step 1.' } elseif ($isSession) { 'Step 2 stops it.' } else { 'Step 2 does not stop it. Continue with Step 4.' }
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
| `LOCK_CHECK_UNAVAILABLE` | The check could not run, so nothing was stopped. | Go to Step 4. |

### Step 3: Confirm that the files are released

Run the Step 1 script again. This changes nothing.

- **`NO_LOCKING_SESSIONS`:** the files are released. Go to Step 4.
- **`LOCKING_SESSIONS_FOUND`:** a process still holds files. If it is a PowerShell session marked `Step 2 stops it.`, it started after Step 2 ran; run Step 2 again. If every listed process is marked `Step 2 does not stop it.`, go to Step 4. The cleanup removes everything it can, and Step 5 covers the rare case where that is not enough.

### Step 4: Run the cleanup script

**Changes:** unloads, uninstalls, and deletes only copies in version-number folders. The inbox module and running VMs are not affected. Run this script only after Issue Validation listed an external copy.

```powershell
$moduleName = "AzStackHci.EnvironmentChecker"
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
```

The result line appears just before the final module list:

| Result | Meaning | Next step |
| --- | --- | --- |
| `CLEANUP_EXTERNAL_REMOVED` | Only the inbox module is listed, at `C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\AzStackHci.EnvironmentChecker.psd1`, with no version-number folder in its path. If the script reported files that are still in use, the copy no longer loads because its module manifest is gone. | Go to Step 6. |
| `CLEANUP_EXTERNAL_REMAINS` | The external copy is still listed, so the operation stays blocked. This is rare: a process that Step 2 does not stop still holds the copy's module manifest. | Go to Step 5. |
| `INBOX_MODULE_MISSING` | The inbox module is not listed. | Stop and [contact Microsoft support](#when-to-contact-microsoft-support). Do not install a replacement. |

To run the cleanup on every machine from one session, save the script above as `Remove-ExternalEnvChecker.ps1` and run:

```powershell
$nodes = (Get-ClusterNode).Name   # Before the cluster exists, set $nodes to the machine names.
Invoke-Command -ComputerName $nodes -FilePath .\Remove-ExternalEnvChecker.ps1
```

### Step 5: Restart the machine if the copy is still listed

Do this step only when Step 4 printed `CLEANUP_EXTERNAL_REMAINS`. A restart releases every file, including files held by processes that Step 2 does not stop.

**Automation boundary.** A person must plan and start this restart. The scripts in this article never restart a machine.

**Changes:** restarts this machine. On a deployed cluster with more than one machine, drain it first so its VMs move to other machines:

- On a deployed cluster with more than one machine, follow the pause and drain procedure in [Suspend and resume Azure Local machines for planned maintenance](https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance), so workloads move to other machines before the restart, and resume the machine afterward.
- On a single-machine instance, there is no other machine to drain to, so the pause and drain procedure does not apply. Plan a maintenance window, and shut down or save the VMs before the restart.
- Before the cluster exists (during deployment), restart the machine normally.

**Plan for:** a maintenance window on a single-machine instance.

> [!WARNING]
> On a single-machine instance, this restart stops all running VMs.

After the restart, run Step 4 again, then Step 6.

### Step 6: Delete leftover folders

**Changes:** deletes only subfolders whose name is a version number. The inbox files are not in such a subfolder and are not touched. Run this after Step 4 printed `CLEANUP_EXTERNAL_REMOVED`.

```powershell
$root = Join-Path $env:ProgramFiles "WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker"
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
```

| Result | Meaning | Next step |
| --- | --- | --- |
| `LEFTOVER_FOLDERS_REMOVED` or `NO_LEFTOVER_FOLDERS` | No version-number folder is left. | Go to [Verify the fix](#verify-the-fix). |
| `LEFTOVER_FOLDER_IN_USE` | A process that Step 2 does not stop still holds files in the folder. The folder's module manifest is already gone, so it does not load and does not block the operation. | Leave the folder for now and go to [Verify the fix](#verify-the-fix). Run this step again after the machine's next planned restart, rather than restarting the machine for this alone. |

## Verify the fix

Run the all-machines check in [Issue Validation](#issue-validation) again. On every machine, `ExternalCopies` should be empty, and `AllAvailableModules` should show only `C:\Program Files\WindowsPowerShell\Modules\AzStackHci.EnvironmentChecker\AzStackHci.EnvironmentChecker.psd1`. `VersionFolders` should also be empty, except for a folder that you deliberately left in place under [Step 6](#step-6-delete-leftover-folders) until the next planned restart.

Do not use the system health check to confirm this fix. It can keep showing a previous result, and it runs the cleanup only on the machine where it runs.

When every machine is clean, retry the operation as described in [Retry the failed operation](#retry-the-failed-operation).

## Retry the failed operation

Retry only after [Verify the fix](#verify-the-fix) shows every machine clean.

### Resume a solution update

Run this on one machine of the deployed system. It resumes a solution update that failed during installation, which is the PowerShell equivalent of **Try again** in the Azure portal.

**Changes:** resumes the update only when exactly one update is in the `InstallationFailed` state. The update continues from the step that failed. In every other case the script changes nothing and only lists the updates.

**Plan for:** the rest of the update installation, including any machine restarts that the update performs, as when you first started it.

```powershell
$updates = $null
$updateError = $null
try { $updates = @(Get-SolutionUpdate -ErrorAction Stop) }
catch { $updateError = $_.Exception.Message }
if ($null -eq $updates) {
    Write-Output "UPDATE_SERVICE_NOT_AVAILABLE"
    Write-Output "Could not read solution updates on this machine: $updateError"
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
        Write-Output "More than one update is in the InstallationFailed state. Resume the one you started by selecting Try again on it in the Azure portal."
    } else {
        Write-Output "NO_UPDATE_TO_RESUME"
    }
    Get-SolutionUpdate -ErrorAction SilentlyContinue | Format-Table DisplayName, Version, State -AutoSize
}
```

| Result | Meaning | Next step |
| --- | --- | --- |
| `SOLUTION_UPDATE_RESUMED` | The update is installing again. Its `State` in the list is `Installing`. | Follow its progress in the Azure portal, under **Updates**. It should continue past the step that failed. |
| `NO_UPDATE_TO_RESUME` | No solution update failed during installation. | If an update in the list shows `HealthCheckFailed`, it failed in its readiness check: retry it the same way you started it, with **Try again** in the Azure portal or with the same `Start-SolutionUpdate` command (including `-PrepareOnly` if you used it). If an upgrade, add node, or repair node operation failed, retry it the same way you started it. |
| `UPDATE_SERVICE_NOT_AVAILABLE` | The solution update service did not answer. This is expected while the system is still being deployed. | If a deployment failed, go to [Resume a deployment](#resume-a-deployment). On a deployed system, see [Troubleshoot solution updates for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-troubleshooting-23h2). |
| `SOLUTION_UPDATE_NOT_RESUMED` | The update did not start again. The script prints the reason. | Select **Try again** on the failed update in the Azure portal. If it still does not start, [contact Microsoft support](#when-to-contact-microsoft-support). |

### Resume a deployment

Do this only when the operation that failed was a deployment.

**Automation boundary.** A person must resume a deployment in the Azure portal. It cannot be resumed from a script on the machine.

**Changes:** resumes the deployment from the step that failed.

In the Azure portal, open your Azure Local instance, go to **Deployments**, and select **Resume deployment**. The deployment should continue past the step that failed.

## Rollback

Normally there is nothing to roll back. A copy installed from the PowerShell Gallery was not part of Azure Local, and a copy left behind by deployment is removed by Azure Local itself before each operation. If you need the standalone Environment Checker from the PowerShell Gallery again, reinstall it with `Install-Module -Name AzStackHci.EnvironmentChecker` after the operation completes, and remove it again before the next one. If you paused and drained a machine, resume it as described in the pause and drain procedure. Sessions that Step 2 stopped cannot be restored; open new PowerShell sessions if you still need them.

## When to contact Microsoft support

Open a support request if any of these is true:

- The inbox module is missing from `AllAvailableModules`.
- Step 4 still prints `CLEANUP_EXTERNAL_REMAINS` after you restarted the machine and ran Step 4 again.
- A leftover version-number folder still cannot be deleted after the machine has restarted.
- The same error returns after the retry while Issue Validation shows no external copy.
- The solution update does not start again after **Try again** in the Azure portal.

Include this information:

- The output of the all-machines Issue Validation command, before and after the steps you ran.
- The full output of Step 1 and of the cleanup script from every machine.
- The exact error text and the time it occurred, from the failed deployment or update step in the Azure portal.
- The deployment log from `C:\CloudDeployment\Logs` that contains `Externally installed module(s) found for AzStackHci.EnvironmentChecker`.
- The Azure Local version and the version of the inbox Environment Checker module.

## Prevention

Azure Local already includes the Environment Checker. If you install it from the PowerShell Gallery to run readiness checks, close that PowerShell window when you finish, then remove the gallery copy from every machine before you start a deployment or update, as described in [Assess your environment with the Environment Checker](https://learn.microsoft.com/azure/azure-local/manage/use-environment-checker).

**Changes:** unloads the module from the current session and uninstalls only copies in version-number folders. The inbox module is not affected.

```powershell
Remove-Module AzStackHci.EnvironmentChecker -Force
Get-Module AzStackHci.EnvironmentChecker -ListAvailable | Where-Object {$_.Path -like "*$($_.Version)*"} | Uninstall-Module -Force
```

## Related documentation

- [Assess your environment with the Environment Checker](https://learn.microsoft.com/azure/azure-local/manage/use-environment-checker)
- [Troubleshoot solution updates for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-troubleshooting-23h2)
- [Understand update phases for Azure Local](https://learn.microsoft.com/azure/azure-local/update/update-phases-23h2)
- [Deploy Azure Local using the Azure portal](https://learn.microsoft.com/azure/azure-local/deploy/deploy-via-portal)
- [Suspend and resume Azure Local machines for planned maintenance](https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance)
