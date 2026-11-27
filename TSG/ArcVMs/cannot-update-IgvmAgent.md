# Symptoms    
When updating your Azure Local instance to the 2505 release, you encounter one of the following errors:

```
Could not update IgvmAgent on physical node Exception: Unable to create a new version folder with in 10 tries.
```
```
Could not update IgvmAgent on physical node Exception: Cannot remove item C:\CloudContent\IgvmAgent\_v2\AszIgvmAgent.exe: Access to the path ‘C:\CloudContent\IgvmAgent_v2\AszIgvmAgent.exe’ is denied.
```
Note, in the sample error message, _v2 refers to a version 2, which in practice can be v*, * range between 1 and 9.

# Issue Validation

Run the following PowerShell in an elevated session on one machine in your Azure Local instance. It only reads: it walks the steps of your solution update runs and lists every step whose error matches one of the errors above. If the update runs can't be read (for example, the update service is not reachable), it searches the update summary logs on that machine instead; those logs are on the machine that ran the update.

```powershell
& {
    $pattern = 'Could not update IgvmAgent on physical node'
    $found = @()
    $fromLogs = $false
    try {
        $runs = @(Get-SolutionUpdate -ErrorAction Stop | Get-SolutionUpdateRun -ErrorAction Stop)
        foreach ($run in $runs) {
            $pending = [System.Collections.Generic.Queue[object]]::new()
            if ($null -ne $run.Progress) { $pending.Enqueue($run.Progress) }
            while ($pending.Count -gt 0) {
                $step = $pending.Dequeue()
                if ("$($step.ErrorMessage) $($step.AggregatedErrorMessages)" -match $pattern) {
                    $found += [pscustomobject]@{ UpdateRun = $run.ResourceId; Step = $step.Name; Status = $step.Status; Error = $step.ErrorMessage }
                }
                foreach ($child in @($step.Steps)) { if ($null -ne $child) { $pending.Enqueue($child) } }
            }
        }
    }
    catch {
        Write-Host "Could not read the update runs ($($_.Exception.Message)). Searching the update summary logs on this machine instead."
        $fromLogs = $true
        $found = @(Select-String -Path 'C:\CloudDeployment\Logs\UpdateSummary*.xml' -Pattern $pattern -SimpleMatch -ErrorAction SilentlyContinue |
            Select-Object @{ n = 'File'; e = { $_.Path } }, @{ n = 'Line'; e = { $_.LineNumber } }, @{ n = 'Error'; e = { $_.Line.Trim() } })
    }
    if ($found.Count -gt 0) {
        'The IgvmAgent update error is present. Continue with Mitigation Details.'
        $found | Format-List
    }
    elseif ($fromLogs) {
        'No update summary log on this machine shows the IgvmAgent update error. Run this on the machine that ran the update.'
    }
    else {
        'No update run shows the IgvmAgent update error, so this article does not apply.'
    }
}
```

If the error is present, the output lists the update run, the step (for example `Update IgvmAgent`), its status and the error text.

# Cause
This occurs because there is an underlying issue with removing certain folders. We have fixed this in 2506 release.

# Mitigation Details

For customers updating their Azure Local instance to 2505 release, if you encounter any of the above errors, perform the following steps to resolve the issue:

1. [Required] Restart Azure Local instance (all the machines in your Azure Local instance).
2. [Optional] Run IGVM Agent update plan.
   - Log in into one of the nodes instances and open a Powershell admin console.
   - Create an ECE Cluster service by running: ```$ececlient = Create-EceClusterServiceClient```
   - Run update action plan for IGVM Agent by running: ```$guid = Invoke-ActionPlanInstance -RolePath IgvmAgentDeployment -ActionType DeployIgvmAgent -EceClient:$ececlient```
   - Monitor the action plan until it succeeds by running: ```Start-MonitoringActionplanInstanceToComplete $guid```
3. [Required] Resume Azure Local update (repeat steps to update your Azure Local instance).

Alternately, you can skip 2505 and directly update your Azure Local instance to 2506 release or above to resolve the issue.