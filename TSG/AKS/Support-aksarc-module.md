<!-- tsg-metadata
{
  "schema": "azure-local-supportability/tsg-metadata/v2",
  "document_type": "troubleshoot",
  "products": ["Azure Local", "Azure Local VMs", "AKS enabled by Azure Arc"],
  "detector": {"type": "command", "signal": "Test-SupportAksArcKnownIssues"}
}
-->

# Support.AksArc Module Troubleshooting Guide

The Support.AksArc PowerShell module provides diagnostic and remediation capabilities for Azure Kubernetes Service on Azure Arc environments.

It checks the infrastructure that Azure Local VMs and AKS enabled by Azure Arc run on, mostly Microsoft On-Premises Cloud (MOC). MOC is the Azure Local service layer that creates and manages those VMs and their disks, networks, and images. The module can fix many of the known issues it finds.

## Table of contents

- [Symptoms](#symptoms)
- [Where these problems appear](#where-these-problems-appear)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Commands](#commands)
- [Support](#support)

## Symptoms

Use this guide on Azure Local 2311.2 or later when you see any of these problems:

- Azure Local VM or AKS Arc operations fail, for example creating, changing, or deleting a VM.
- A solution update fails in the MOC binaries stage or in the Azure Arc resource bridge stage.
- The MOC service doesn't stay online.
- Azure Arc resource bridge, the appliance that lets Azure manage your VMs, is offline.

**Severity:** depends on the failing test. A failed test can block Azure Local VM and AKS Arc operations or a solution update. The module checks the MOC infrastructure only; it doesn't check hardware, storage, or network faults.

**Who runs it:** an Azure Local administrator. If the module doesn't fix the problem, contact Microsoft Support.

## Where these problems appear

| Admin surface | Status | What you see |
| --- | --- | --- |
| PowerShell on an Azure Local machine | Shown | `Test-SupportAksArcKnownIssues` lists each test with its status and a message. This is the only place the module reports its results. |
| Azure portal | Not evident | The module's results don't appear in the portal, and the system health check on the **Updates** page doesn't run these tests, so a passing health check doesn't rule out these problems. You may only see their effects, such as a failed solution update or an Azure Arc resource bridge whose status is **Offline**. |
| Windows Failover Cluster Manager | Shown | For one test only: `Validate MOC Cloud Agent Running` fails when the `MOC Cloud Agent Service` resource in the MOC cloud agent role isn't **Online**, which Failover Cluster Manager shows under **Roles**. Other test results don't appear there. |
| Windows event logs | Not evident | The module doesn't write its results to any Windows event log. |
| Cluster logs (Get-ClusterLog) | Not evident | The module doesn't write its results to the cluster log. |
| Windows Admin Center (standalone host) | Not evident | Windows Admin Center doesn't run these tests or show their results. The problems don't raise a Health Service fault, so its alerts don't list them. |
| Windows Admin Center in the Azure portal | Not evident | Windows Admin Center in the Azure portal doesn't run these tests or show their results. The problems don't raise a Health Service fault, so its alerts don't list them. |
| Component / tool log files (on disk) | Not evident | The module doesn't write a log file. To keep a record, start a transcript as shown in [Support](#support). |

## Prerequisites

- Azure Local 2311.2 or later, with the Moc PowerShell module that Azure Local installs on every machine.
- Sign in directly to one machine of your Azure Local instance, at the console or with Remote Desktop, using your deployment user account or another account that is an administrator on every machine. Open Windows PowerShell as Administrator.
- Run every command in this guide in Windows PowerShell 5.1 (`powershell.exe`), not in PowerShell 7 (`pwsh.exe`), which newer Azure Local machines also have. The module doesn't work correctly in PowerShell 7: the test reports a Failover Cluster service failure that isn't real. To check which one you're in, run `$PSVersionTable.PSEdition`. It must return `Desktop`; `Core` means PowerShell 7.
- Don't run the module in a remote PowerShell session, such as `Enter-PSSession` or `Invoke-Command`. The test uses PowerShell remoting from the machine you're signed in to, to reach every machine in the instance that is Up. In a remote session, that machine usually can't pass your sign-in on to the other machines, so the test can't reach them and doesn't run.
- PowerShell remoting (WinRM) must work from that machine to every machine in the instance that is Up. `Get-ClusterNode` shows the state of each machine.
- Access to the PowerShell Gallery, directly or through your proxy. You need it to install the module, and each time the module loads it checks the gallery for a newer version.

## Installation

**Changes:** installs the Support.AksArc module from the PowerShell Gallery and loads it into the current PowerShell session. It doesn't change Azure Local, MOC, or your workloads.

```powershell
Install-Module -Name Support.AksArc

Import-Module Support.AksArc

```

`Install-Module` asks you to confirm installing from the PSGallery repository, which isn't trusted by default, and to accept the module license. Answer Yes to both. When the module loads, it shows a disclaimer that starts with `THE SOFTWARE IS PROVIDED "AS IS"` and then prints `Checking if you are on the latest version`. It can also show a warning that some commands from the module `Common` use unapproved verbs. These are expected.

If the module was already installed, update it before you use it, so that you run the current tests:

**Changes:** replaces the installed Support.AksArc module with the newest version from the PowerShell Gallery and reloads it.

```powershell
Remove-Module -Name Support.AksArc -ErrorAction SilentlyContinue
Update-Module -Name Support.AksArc
Import-Module -Name Support.AksArc
```

## Commands 

### Test for known issues

Run this command to check if your cluster is running into any of the known issues.

```powershell
Test-SupportAksArcKnownIssues

```

The test takes about a minute on a three-machine instance. Version 1.3.164 of the module runs 23 tests. They check, on every machine that is Up:

- the Failover Cluster and Hyper-V Virtual Machine Management services;
- the MOC cloud agent, node agent, and host agent services, and whether MOC is on the latest patch of its version;
- MOC certificates and tokens, MOC PowerShell settings, and the cloud agent placement rule;
- whether the MOC machines match the cluster machines and are active;
- VMs and gallery images that failed to delete, and the user storage containers;
- the Windows Event Log service.

It only reports what it finds. It doesn't fix anything. Most tests skip machines that are Down or Paused. The Failover Cluster and Hyper-V Virtual Machine Management service tests check every machine in the cluster, so a machine that is Down makes the Failover Cluster test fail and stops the run with the `Critical Test` error. Bring that machine back Up, or find out why it's Down, before you act on the results. While it runs, it also prints many `VERBOSE:` progress lines. They're expected.

The output ends with a summary line, followed by a table that lists each test with its `TestName`, its `Status`, and a `Message`. For example output, see [Use the Support tool to troubleshoot and fix Azure Local infrastructure problems](https://learn.microsoft.com/azure/azure-local/manage/remediate-support-tool-infrastructure). Use this table to decide what to do next:

| What you see | What it means | What to do |
| --- | --- | --- |
| The summary `All tests passed successfully.`, and every test in the table shows `Passed` | The module found no known issue. | Don't run the remediation command. If you still have a problem, see [Support](#support). |
| The summary `Some tests failed. Run Invoke-SupportAksArcRemediation to remediate issues.`, and one or more tests in the table show `Failed` or `Error` | The module found a known issue, or a test couldn't finish. | Run the remediation command in the next section. |
| No table, and the error `Critical Test Failover Cluster service not responsive or not running failed: Failover Cluster service is not running on node: <machine>. Exception: A parameter cannot be found that matches parameter name 'ComputerName'.` | You ran the test in PowerShell 7. The result isn't valid. | Don't run the remediation command. Open Windows PowerShell 5.1 as described in [Prerequisites](#prerequisites), and run the test again. |
| No table, and any other error `Critical Test <test name> failed: <message>` | A critical check failed. The module stops, because the later results can't be trusted. | Run the remediation command in the next section. |
| The error `Please ensure you are running this script directly on the host and not over remote powershell` | The module couldn't reach every machine that is Up over PowerShell remoting. It ran no tests. | Don't run the remediation command. Sign in directly to a machine as described in [Prerequisites](#prerequisites). If you already did, fix PowerShell remoting to the machine named in the error `Remote PowerShell is not available on node: <machine>`, and then run the test again. |
| The summary `Tests completed with warnings. Review warning details before remediation.`, and the tests in the table show only `Passed` or `Warning` | The module found a condition to review, such as no isolated user storage container. | Read the `Message` of each `Warning` test and follow it. The remediation command doesn't change anything for a warning. |

### Test for issues with remediation

Run this command only when the test above reported a `Failed` or `Error` test, or the `Critical Test` error. It reruns every test and fixes only the tests that don't pass. On a three-machine instance with one failed test, it took about 75 seconds; the time depends on the fixes it applies.

**Changes:** depending on the failing test, the fix can:

- update MOC to the latest patch of its current version;
- stop and start the MOC cloud agent role and the MOC node agent and host agent services on every machine, and renew MOC identities and certificates;
- delete stale MOC working folders, and reset MOC PowerShell settings to the standard Azure Local values;
- re-create the MOC PowerShell registry setting or the cloud agent cluster affinity rule `Cloud Agent-CNO`;
- start the Windows Event Log service;
- remove gallery images that failed to delete, and stop and remove VMs that failed to delete because their OS disk was in use, along with any other VM whose name contains one of their names.

It doesn't restart the Failover Cluster service or the Hyper-V Virtual Machine Management service.

**Plan for:** Azure Local VM and AKS Arc management operations, such as creating, changing, or deleting a VM, can fail while MOC agents restart or MOC updates. Apart from the VM removal described in the warning below, the fixes work on the MOC infrastructure, not on running VMs. Machines that are Down or Paused aren't fixed. Run the command in a maintenance window.

> [!WARNING]
> **[HIGH RISK]** The module doesn't undo its fixes, and there is no rollback. An MOC update stays in place. If the test `Validate Virtual Machine Stuck In Deleting Due To OS Disk Lock` failed, the fix stops and removes every VM that its message lists. It finds those VMs by matching part of the name, so it also stops and removes any other VM whose cluster role or Hyper-V name contains a listed VM's name. A removed VM can't be recovered. Run the check below first, and don't run the command if it shows a VM you don't intend to remove.

First, check whether the fix would remove VMs. This check runs only that test and changes nothing:

```powershell
$vmTest = @(Test-SupportAksArcKnownIssues -TestId MOC-0017)
if ($vmTest.Count -eq 0) { 'TEST-DID-NOT-RUN' }
elseif (@($vmTest | Where-Object { $_.Status -in 'Failed', 'Error' }).Count -gt 0) { 'VM-REMOVAL-PENDING'; $vmTest | Format-List TestName, Status, Message }
else { 'NO-VM-REMOVAL' }
```

- `NO-VM-REMOVAL`: run the command below.
- `VM-REMOVAL-PENDING`: the command below would stop and remove the VMs that the message lists, and any other VM whose name contains one of those names. Run the next check to see every VM it would remove.
- `TEST-DID-NOT-RUN`: the test ran no checks. Use the results table above to find out why.

If the check printed `VM-REMOVAL-PENDING`, run this in the same PowerShell window. It lists every cluster role and Hyper-V VM that the fix would stop and remove, matched the same way the fix matches them, and changes nothing. Its last line is `VM-LIST-COMPLETE` only if it checked every machine that is `Up`.

**Stop here and confirm.** Before anyone runs the command below, the owner of the VMs reviews the list this check prints and decides whether every VM on it can be removed. Done looks like: the last line is `VM-LIST-COMPLETE`, and the owner has confirmed each listed VM by name.

```powershell
$listComplete = $true
try {
    $names = @($vmTest | ForEach-Object { $_.AffectedVMs } | ForEach-Object { $_.VMName })
    if ($names.Count -eq 0) { $listComplete = $false; 'VM-LIST-ERROR: the test result lists no VM names. Run the check above again in this window.' }
    foreach ($name in $names) {
        $groups = @(Get-ClusterGroup -ErrorAction Stop | Where-Object { $_.Name.Contains($name) })
        if ($groups.Count -gt 0) { $groups | ForEach-Object { "MOC VM $name -> cluster role $($_.Name) on $($_.OwnerNode.Name)" } }
        else {
            $vms = @(); $notChecked = @()
            foreach ($node in @(Get-ClusterNode -ErrorAction Stop | Where-Object { $_.State -eq 'Up' } | ForEach-Object { $_.Name })) {
                try { $vms += @(Get-VM -ComputerName $node -ErrorAction Stop | Where-Object { $_.Name -like "*$name*" } | ForEach-Object { "MOC VM $name -> Hyper-V VM $($_.Name) on $node" }) }
                catch { $notChecked += "MOC VM $name -> NOT CHECKED on ${node}: $($_.Exception.Message)" }
            }
            $vms; $notChecked
            if ($notChecked.Count -gt 0) { $listComplete = $false }
            elseif ($vms.Count -eq 0) { "MOC VM $name -> no matching cluster role or Hyper-V VM found" }
        }
    }
} catch { $listComplete = $false; "VM-LIST-ERROR: $($_.Exception.Message)" }
if ($listComplete) { 'VM-LIST-COMPLETE' } else { 'VM-LIST-INCOMPLETE' }
```

- If the last line is `VM-LIST-COMPLETE` and the owner confirmed that every VM it lists can be removed, you can run the command below.
- If the last line is `VM-LIST-INCOMPLETE`, the list is missing at least one machine: a `NOT CHECKED` line names the machine it couldn't query, or a `VM-LIST-ERROR` line shows why the check stopped. Don't run the command below, and don't approve removing any VM, until every machine is checked. Fix the reason shown, for example Hyper-V access to that machine, and run this check again.
- If any VM it lists isn't one you intend to remove, or you aren't sure, stop. Don't run the command below. Contact Microsoft Support before you remove any VM. You can still run the block that skips that test, after the command below, to fix the other failed tests.

```powershell
if ($PSVersionTable.PSEdition -ne 'Desktop') { throw 'Run this command in Windows PowerShell 5.1 (powershell.exe), not PowerShell 7.' }
Invoke-SupportAksArcRemediation

```

For each test, the command prints either `No remediation required for <test>.`, or `Remediation for <test> is required.` and then `Remediation for <test> succeeded.` or the warning `Remediation for <test> failed.` Two tests have no automatic fix; see [If a test has no automatic fix](#if-a-test-has-no-automatic-fix).

If the check printed `VM-REMOVAL-PENDING` and you didn't run the command above, run this block instead. It fixes every failed test except `Validate Virtual Machine Stuck In Deleting Due To OS Disk Lock`, which removes VMs. It runs the test and reads each test's ID from its `Running test: <test name> with ID: <ID>` line. Then it runs the remediation for each test that failed, had an error, or stopped the run as a critical test, except `MOC-0017`. It prints:

- `REMEDIATE <ID>` before each fix;
- `SKIPPED MOC-0017` if it left that test alone;
- `NO-FAILED-TESTS` if no test needed a fix;
- `TEST-DID-NOT-RUN` if the test ran no checks. Use the results table above to find out why.

If a critical test stopped the run, the tests after it didn't run, so this block can't fix them yet. The verify step shows whether any of them failed.

**Changes:** applies the fixes listed above for the failed tests only, one test at a time. It never runs the fix that removes VMs, but it can still update MOC and restart MOC agents.

> [!WARNING]
> **[HIGH RISK]** This block can't be undone either. An MOC update, renewed MOC identities and certificates, and deleted MOC working folders stay as they are; there is no rollback. If you aren't sure the failed tests are the problem you're troubleshooting, contact Microsoft Support before you run it.

```powershell
if ($PSVersionTable.PSEdition -ne 'Desktop') { throw 'Run this command in Windows PowerShell 5.1 (powershell.exe), not PowerShell 7.' }
$info = $null
$results = @(Test-SupportAksArcKnownIssues -InformationVariable info)
$ids = @($info | ForEach-Object { [regex]::Match([string]$_.MessageData, '^Running test: .+ with ID: ((MOC|FC|VMMS)-\d{4})$') } | Where-Object { $_.Success } | ForEach-Object { $_.Groups[1].Value })
$failed = @(for ($i = 0; $i -lt $ids.Count; $i++) { if ($i -ge $results.Count -or $results[$i].Status -in 'Failed', 'Error') { $ids[$i] } })
if ($ids.Count -eq 0) { 'TEST-DID-NOT-RUN' }
elseif ($failed.Count -eq 0) { 'NO-FAILED-TESTS' }
foreach ($id in $failed) {
    if ($id -eq 'MOC-0017') { 'SKIPPED MOC-0017'; continue }
    "REMEDIATE $id"
    Invoke-SupportAksArcRemediation -TestId $id
}
```

### If a test has no automatic fix

The tests `Failover Cluster service not responsive or not running` (`FC-0011`) and `Virtual Machine Management service not responsive or not running` (`VMMS-0012`) have no automatic fix. For them, the remediation command prints `Auto Remediation not available. Please Restart the Node manually, which has the issue` and still reports success, but it changes nothing.

After any remediation, run this check. It runs only those two tests and changes nothing:

```powershell
if ($PSVersionTable.PSEdition -ne 'Desktop') { throw 'Run this command in Windows PowerShell 5.1 (powershell.exe), not PowerShell 7.' }
$fcErrors = $null; $vmmsErrors = $null
$fcRows = @(Test-SupportAksArcKnownIssues -TestId FC-0011 -ErrorVariable fcErrors)
$vmmsRows = @(Test-SupportAksArcKnownIssues -TestId VMMS-0012 -ErrorVariable vmmsErrors)
$critical = @(@($fcErrors) + @($vmmsErrors) | Where-Object { "$_" -match '^Critical Test ' })
$failed = @(@($fcRows) + @($vmmsRows) | Where-Object { $_.Status -in 'Failed', 'Error' })
if ($critical.Count -gt 0 -or $failed.Count -gt 0) { 'MANUAL-RESTART-NEEDED'; $critical | ForEach-Object { "$_" }; $failed | Format-List TestName, Status, Message }
elseif ($fcRows.Count -eq 0 -or $vmmsRows.Count -eq 0) { 'TEST-DID-NOT-RUN' }
else { 'NO-MANUAL-RESTART' }
```

- `NO-MANUAL-RESTART`: continue with [Verify the fix](#verify-the-fix).
- `MANUAL-RESTART-NEEDED`: the message names the machine. Run `Get-ClusterNode` to see its state:
  - If the machine is `Up` or `Paused`, follow [If the machine is Up or Paused](#if-the-machine-is-up-or-paused).
  - If the machine is `Down`, follow [If the machine is Down](#if-the-machine-is-down).

  Then run this check again. If it still prints `MANUAL-RESTART-NEEDED`, contact Microsoft Support.
- `TEST-DID-NOT-RUN`: at least one of the two tests ran no checks, so the check can't say that no restart is needed. Use the results table above to find out why, and then run this check again.

#### If the machine is Up or Paused

**Operator action.** An administrator restarts the machine as described in [Suspend and resume Azure Local machines for maintenance](https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance): suspend and drain it, restart it, and resume it. Done looks like: `Get-ClusterNode` shows the machine `Up`, and the check above prints `NO-MANUAL-RESTART`.

**Changes:** the restart moves the machine's workloads to other machines, restarts it, and returns it to the instance. On a single-machine instance there is no other machine, so its VMs stop until the restart finishes.

**Plan for:** the instance runs with one machine fewer until the restart finishes. Restart in a maintenance window.

#### If the machine is Down

`Down` means the instance can't reach the machine's Cluster service. The machine can still be powered on and running VMs, for example when only its Cluster service stopped or it lost network communication with the other machines. Find out which before you restart anything.

1. Check whether the machine is reachable and what runs on it. Replace `MACHINE-NAME` with the machine's name, and run this on any machine that is `Up` (on a single-machine instance, on that machine). It changes nothing:

   ```powershell
   $machine = 'MACHINE-NAME'
   try {
       $null = Get-ClusterNode -Name $machine -ErrorAction Stop
       $state = $null
       try {
           $state = Invoke-Command -ComputerName $machine -ErrorAction Stop -ScriptBlock {
               $ErrorActionPreference = 'Stop'
               [pscustomobject]@{ ClusSvc = [string](Get-Service -Name ClusSvc).Status; VMs = @(Get-VM | Where-Object { $_.State -ne 'Off' } | ForEach-Object { "$($_.Name) ($($_.State))" }) }
           }
       } catch { "MACHINE-NOT-REACHABLE: $($_.Exception.Message)" }
       if ($state) {
           "MACHINE-REACHABLE ClusSvc=$($state.ClusSvc) VMsNotOff=$(@($state.VMs).Count)"
           @($state.VMs) | ForEach-Object { "  VM: $_" }
       }
   } catch { "CHECK-ERROR: $($_.Exception.Message)" }
   ```

   - `MACHINE-REACHABLE` with `ClusSvc` other than `Running`: the machine is on, and only its Cluster service stopped. Don't restart the machine. Start the service instead, which doesn't stop its VMs: `Invoke-Command -ComputerName $machine -ScriptBlock { Start-Service -Name ClusSvc }`. Then go to step 4.
   - `MACHINE-REACHABLE` with `ClusSvc=Running`: the machine is on, but the instance can't communicate with it. Don't restart it; a restart would stop the VMs it lists. Contact Microsoft Support.
   - `MACHINE-NOT-REACHABLE`: go to step 2.
   - `CHECK-ERROR`: the machine name isn't a machine in this instance, or the instance couldn't be queried. Fix the name and run the check again.

2. Check the machine from its baseboard management controller (BMC) or at the machine. Restart it only if it's powered off or doesn't respond at all, from its console or on the network. If it's powered on and responds, don't restart it; contact Microsoft Support.

   On an instance with more than one machine, run this in the same PowerShell window as step 1, on a machine that is `Up`, to confirm that its clustered VMs moved to other machines. It should print nothing:

   ```powershell
   Get-ClusterGroup | Where-Object { $_.OwnerNode.Name -eq $machine } | Format-Table Name, State
   ```

   If it lists any VM role, that VM didn't move. Contact Microsoft Support before you restart the machine.

3. **Operator action.** An administrator turns on or restarts the machine from its BMC or at the machine, only after step 2 shows that it's off or unresponsive and, on an instance with more than one machine, that its VMs moved. Done looks like: step 4 shows the machine `Up` or `Paused`.

4. Run this on any machine that is `Up` (on a single-machine instance, on that machine), until the machine shows `Up` or `Paused`:

   ```powershell
   Get-ClusterNode | Format-Table Name, State
   ```

5. If the machine shows `Paused`, resume it with `Resume-ClusterNode -Name` followed by the machine name.
6. Run the check in [If a test has no automatic fix](#if-a-test-has-no-automatic-fix) again.

**Changes:** starting the Cluster service returns the machine to the instance without stopping its VMs. Turning on or restarting the machine from its BMC stops anything still running on it, which is why you check first that it's off or unresponsive. On a single-machine instance, that restart stops every VM on the machine. Clustered VMs normally start again when the instance brings its roles back online; check that the VMs you need are running.

### Verify the fix

Run the test again, and list any test that failed:

```powershell
$results = @(Test-SupportAksArcKnownIssues)
$failed = @($results | Where-Object { $_.Status -in 'Failed', 'Error' })
$failed | Format-List TestName, Status, Message
"Tests run: $($results.Count). Failed or error: $($failed.Count)."
```

The fix worked when no test is listed, no error is shown, and the last line shows `Failed or error: 0` with a test count above 0, for example `Tests run: 23. Failed or error: 0.` `Tests run: 0` means an error stopped the run. A `Warning` test isn't listed; handle it as described in the results table. If any test is listed, or an error remains, go back to [Test for issues with remediation](#test-for-issues-with-remediation) once, starting with the check for VM removal. If a test is still listed after that, see [Support](#support).


## Support
Review Azure Local  documentation or contact Microsoft Support for unresolved issues.

When you contact Microsoft Support, include the Support.AksArc module version, your Azure Local version, and the output of a new test run. This block saves all three to a file on your Desktop. If you ran the remediation command, also copy its console output into the case.

**Changes:** writes a transcript file to your Desktop. It doesn't change Azure Local.

```powershell
Start-Transcript -Path "$env:USERPROFILE\Desktop\Support-AksArc-$(Get-Date -Format yyyyMMdd-HHmmss).txt"
Import-Module -Name Support.AksArc
Get-Module -Name Support.AksArc | Select-Object Name, Version | Out-Host
Get-SolutionUpdateEnvironment | Select-Object CurrentVersion | Out-Host
Test-SupportAksArcKnownIssues | Format-List TestName, Status, Message | Out-Host
Stop-Transcript
```
