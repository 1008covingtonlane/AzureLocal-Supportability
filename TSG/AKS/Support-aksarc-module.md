# Support.AksArc Module Troubleshooting Guide

The Support.AksArc PowerShell module provides diagnostic and remediation capabilities for Azure Kubernetes Service on Azure Arc environments.

## Prerequisites

- PowerShell 5.1+, Administrator privileges, Moc module, Azure Stack HCI

## Installation

**Changes:** installs the Support.AksArc module from the PowerShell Gallery and loads it into the current PowerShell session.

```powershell
Install-Module -Name Support.AksArc

Import-Module Support.AksArc

```

## Commands 

### Test for known issues

Run this command to check if your cluster is running into any of the known issues.

```powershell
Test-SupportAksArcKnownIssues

```

The output lists each test with its `Status`, `Passed` or `Failed`, and a message. For example output, see [Use the Support tool to troubleshoot and fix Azure Local infrastructure problems](https://learn.microsoft.com/azure/azure-local/manage/remediate-support-tool-infrastructure).

- If every test shows `Passed` and the command reports no error, the module found no known issue. Do not run the remediation command. If you still have a problem, see [Support](#support).
- If any test shows `Failed`, or the command reports an error, run the remediation command in the next section.

### Test for issues with remediation

Run this command only when the test above reported a failed test or an error. It fixes the known issues that have a remediation.

**Changes:** applies the module's fixes for the known issues that it finds. These fixes can update infrastructure components, such as Microsoft On-Premises Cloud (MOC).

```powershell
Invoke-SupportAksArcRemediation

```

Then run `Test-SupportAksArcKnownIssues` again and confirm that every test shows `Passed` and that the command reports no error. If any test still shows `Failed`, or the error remains, see [Support](#support).


## Support
Review Azure Local  documentation or contact Microsoft Support for unresolved issues.
