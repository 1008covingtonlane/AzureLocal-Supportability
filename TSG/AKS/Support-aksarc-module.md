<!-- tsg-metadata
{
  "schema": "azure-local-supportability/tsg-metadata/v1",
  "document_type": "troubleshoot",
  "products": ["Azure Local", "Azure Local VMs", "AKS enabled by Azure Arc"],
  "detector": {"type": "command", "signal": "Test-SupportAksArcKnownIssues"},
  "validation": {
    "fidelity_level": "L4",
    "technical_grade": "A",
    "reproduction_substrate": "vm",
    "automation_status": "proven",
    "last_validated": "2026-11-25",
    "spec_ref": "AzLocal_AKSArc_SupportModule_KnownIssues"
  }
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
- Run every command in this guide in Windows PowerShell 5.1 (`powershell.exe`), not in PowerShell 7 (`pwsh.exe`), which Azure Local machines also have. The module doesn't work correctly in PowerShell 7: the test reports a Failover Cluster service failure that isn't real. To check which one you're in, run `$PSVersionTable.PSEdition`. It must return `Desktop`; `Core` means PowerShell 7.
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

It only reports what it finds. It doesn't fix anything. Machines that are Down or Paused aren't checked. While it runs, it also prints many `VERBOSE:` progress lines. They're expected.

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
- remove gallery images that failed to delete, and stop and remove VMs that failed to delete because their OS disk was in use.

It doesn't restart the Failover Cluster service or the Hyper-V Virtual Machine Management service.

**Plan for:** Azure Local VM and AKS Arc management operations, such as creating, changing, or deleting a VM, can fail while MOC agents restart or MOC updates. Apart from removing VMs that failed to delete, the fixes work on the MOC infrastructure, not on running VMs. Machines that are Down or Paused aren't fixed. Run the command in a maintenance window.

> [!WARNING]
> The module doesn't undo its fixes. An MOC update stays in place. If the test `Validate Virtual Machine Stuck In Deleting Due To OS Disk Lock` failed, the fix stops and removes every VM that its message lists. Before you run the command, read that message and confirm that you meant to delete those VMs. If you didn't, fix only the other failing tests, one at a time, as described after this command.

```powershell
if ($PSVersionTable.PSEdition -ne 'Desktop') { throw 'Run this command in Windows PowerShell 5.1 (powershell.exe), not PowerShell 7.' }
Invoke-SupportAksArcRemediation

```

For each test, the command prints either `No remediation required for <test>.`, or `Remediation for <test> is required.` and then `Remediation for <test> succeeded.` or the warning `Remediation for <test> failed.` Some tests have no automatic fix. For those, the command prints `Auto Remediation not available. Please Restart the Node manually, which has the issue` and still reports success, but it changed nothing. Restart the machine named in the test message as described in [Suspend and resume Azure Local machines for maintenance](https://learn.microsoft.com/azure/azure-local/manage/suspend-resume-cluster-maintenance), or contact Microsoft Support.

To fix one test only, find its ID in the `Running test: <test name> with ID: <ID>` line of the test output, for example `MOC-0018`, and run the remediation for that ID.

**Changes:** applies only that test's fix, from the list above.

```powershell
if ($PSVersionTable.PSEdition -ne 'Desktop') { throw 'Run this command in Windows PowerShell 5.1 (powershell.exe), not PowerShell 7.' }
Invoke-SupportAksArcRemediation -TestId <ID>
```

### Verify the fix

Run the test again, and list any test that failed:

```powershell
$results = @(Test-SupportAksArcKnownIssues)
$failed = @($results | Where-Object { $_.Status -in 'Failed', 'Error' })
$failed | Format-Table TestName, Status, Message -AutoSize
"Tests run: $($results.Count). Failed or error: $($failed.Count)."
```

The fix worked when no test is listed, no error is shown, and the last line shows `Failed or error: 0` with a test count above 0, for example `Tests run: 23. Failed or error: 0.` `Tests run: 0` means an error stopped the run. A `Warning` test isn't listed; handle it as described in the results table. If any test is listed, or an error remains, see [Support](#support).


## Support
Review Azure Local  documentation or contact Microsoft Support for unresolved issues.

When you contact Microsoft Support, include the Support.AksArc module version, your Azure Local version, and the output of a new test run. This block saves all three to a file on your Desktop. If you ran the remediation command, also copy its console output into the case.

**Changes:** writes a transcript file to your Desktop. It doesn't change Azure Local.

```powershell
Start-Transcript -Path "$env:USERPROFILE\Desktop\Support-AksArc-$(Get-Date -Format yyyyMMdd-HHmmss).txt"
Import-Module -Name Support.AksArc
Get-Module -Name Support.AksArc | Select-Object Name, Version
Get-SolutionUpdateEnvironment | Select-Object CurrentVersion
Test-SupportAksArcKnownIssues
Stop-Transcript
```
