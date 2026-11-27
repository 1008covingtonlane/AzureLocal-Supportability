# TSG | 2509 | Update Service terminated repeatedly by ALM due to high memory usage

Update service can be frequently terminated due to exceeding the maximum configured memory limit. If the service is terminated in this way during preparation, preparation may be unable to complete successfully.

# Symptoms
One symptom observed is that update service is continuously terminated in the middle of every update preparation attempt, while update service is waiting for the health check action plan to complete.

# Issue Validation
To confirm the scenario that you are encountering is the issue documented in this article, confirm you are seeing the following behavior(s)

## PowerShell script

Update service process termination can be detected in the Windows Event logs by examining events in Event Viewer or through PowerShell on the cluster node that is the owner of the `Azure Stack HCI Update Service Cluster Group`. 

Find the current update service owner node:
```powershell
Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group"
```

Sample output:
```
Name                                         OwnerNode State
----                                         --------- -----
Azure Stack HCI Update Service Cluster Group Node01    Online
```

The service that enforces the update service memory limit writes these events to the Application log, with event ID 1 and level Error. On older releases it is the Agent Lifecycle Agent (ALM), and its events use the provider name `AzureStack Agent Lifecycle Agent`. On newer releases, including 2701, the Resource Governor Agent (`AzureStack Resource Governor Agent`) writes them, and its events use the provider name `ResourceGovernorAgent`. The message text is the same for both.

Run the following commands to find events indicating that the update service was terminated due to exceeding the memory limit. The command checks both provider names, so it works on every release:

```powershell
Get-WinEvent -LogName Application | where { $_.ProviderName -in @("AzureStack Agent Lifecycle Agent", "ResourceGovernorAgent") } | where Message -match "Azure Stack HCI Update Service: Memory Error Limit Reached"
```

Sample output from an older release (on newer releases, `ProviderName` is `ResourceGovernorAgent`). Note the `TimeCreated` value to determine that these are recent, relevant events. If the event times are not recent they are likely not relevant:
```
   ProviderName: AzureStack Agent Lifecycle Agent

TimeCreated                      Id LevelDisplayName Message
-----------                      -- ---------------- -------
10/10/2025 6:46:46 PM             1 Error            Azure Stack HCI Update Service: Memory Error Limit Reached, terminating process. Current private memory usage = 4294967384 bytes, memory limit = 4294967296 bytes.
```

The same events can be viewed in Event Viewer, in the Windows Logs > Application log.


# Cause

There is a bug found in 2509 which is a memory leak in a serializer used by the update service cache. This leak can be exacerbated by a large number of health check result objects. These objects are retained even for installed updates and are reported in the corresponding update object returned by `Get-SolutionUpdate`.

# Mitigation Details

## Remove health check result files for installed updates
Health check result files are stored in the `C:\ClusterStorage\Infrastructure_1\Shares\SU1_Infrastructure_1\Updates\HealthCheck` directory on the infrastructure volume. Within that directory, there will be a subdirectory named `System` and additional subdirectories containing the names of installed updates like (for example) `Solution11.2508.1001.51` or `SBE4.1.2508.1`. Directories that correspond to updates that are already installed can be cleaned up.

The following command can be used to clean up all the directories. Save this script on one of the cluster nodes and run.

```powershell
$ErrorActionPreference = "Stop"

Write-Host "Checking for installed updates."
$installedUpdates = Get-SolutionUpdate | where State -eq Installed
$resourceIds = @( $installedUpdates | % { $_.ResourceId.Split('/')[-1] } )
Write-Host "Found $($installedUpdates.Count) installed updates: $($resourceIds -join ", ")"

$removedHC = $false

if ($resourceIds.Count -gt 0)
{
    foreach ($resourceId in $resourceIds)
    {
        $healthCheckPath = Join-Path "C:\ClusterStorage\Infrastructure_1\Shares\SU1_Infrastructure_1\Updates\HealthCheck" $resourceId
        if (Test-Path $healthCheckPath)
        {
            Write-Host "Removing health check directory for $resourceId"
            $removedHC = $true
            Remove-Item -Path $healthCheckPath -Recurse -Force -Verbose
        }
        else
        {
            Write-Host "No health check found for $resourceId at $healthCheckPath"
        }
    }
}

$systemHealthPath = Join-Path "C:\ClusterStorage\Infrastructure_1\Shares\SU1_Infrastructure_1\Updates\HealthCheck" "System"
if (Test-Path $systemHealthPath)
{
    $oldSystemChecks = Get-ChildItem $systemHealthPath | sort LastWriteTime | select -SkipLast 10
    if ($oldSystemChecks)
    {
        Write-Host "Removing $($oldSystemChecks.Count) old system health check results."
        $removedHC = $true
        $oldSystemChecks | Remove-Item -Force -Verbose
    }
    else
    {
        Write-Host "No stale health checks found at $systemHealthPath"
    }
}

if ($removedHC)
{
    Write-Host "Some health checks were removed, so clearing update service cache."
    $clientCert = ls Cert:\LocalMachine\My\ | ? Subject -match "URP" | sort NotAfter | select -last 1
    $updateEndpoint = "https://$(Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group" | % OwnerNode | % Name).$($env:USERDNSDOMAIN):4900"
    $clearCacheResult = Invoke-WebRequest -Certificate $clientCert -UseBasicParsing -Uri "$updateEndpoint/caches/Update" -Method "Delete"
    if ([int]::TryParse($clearCacheResult.Content, [ref]$null))
    {
        Write-Host "Removed $($clearCacheResult.Content) updates from the cache."
    }
    else
    {
        Write-Host "Result from clearing the cache: $($clearCacheResult | Out-String)"
    }
}
```

## Remove failed Update action plan instances
Delete all failed Update action plans except for the last failed one.

```
Import-Module ECEClient -DisableNameChecking
$failedUpdates = Get-ActionPlanInstances | ? { $_.Status -eq "Failed" -and $_.ActionPlanName -match "MAS Update" } | sort LastModifiedDateTime -Descending | select -Skip 1
$instanceIDs = $failedUpdates.InstanceID

$eceClient = Create-ECEClusterServiceClient
$deleteActionPlanInstanceDescription = New-Object Microsoft.AzureStack.Solution.Deploy.EnterpriseCloudEngine.Controllers.Models.DeleteActionPlanInstanceDescription

foreach ($actionPlanInstanceID in $instanceIDs) {
    # remove old instance
    $deleteActionPlanInstanceDescription.ActionPlanInstanceID = $actionPlanInstanceID
    $eceClient.DeleteActionPlanInstance($deleteActionPlanInstanceDescription).Wait()
}
```

## (Last Resort) Increase update service memory limit
If the problem is still occurring, the final thing to do is to temporarily increase the configured memory limit for the update service.

This can be done by changing the configured value for the memory limit in the update service agent manifest on each node and restarting the ALM agent. To do so, set the expected values at the top of this script in MB (existing suggested values are warning: 5.5GB and error: 6GB) and run this script **on every cluster node**.

From the 2701 release there is no ALM agent and no `C:\Agents\AgentManifests` folder; the Resource Governor Agent enforces the limit. The update service limits appear in two files on each node: the update service's registration file, `Azure Stack HCI Update Service.json` in `C:\ProgramData\Microsoft\AzureStack\ResourceGovernance`, and `AgentManifest.json` in the update service package folder under `C:\NugetStore`. On 2701 both files hold the suggested values (warning 5632 MB, error 6144 MB). On these releases the script reads both files on each node and changes nothing: it reports the limits in each file, or stops if a file holds lower values than those at the top of the script. Don't edit either file; if the limits are lower, contact Microsoft Support.

> [!NOTE]
> On releases before 2701, the maximum value allowed by ALM resource governance is **8GB (8192 MB)**. If the configured error limit exceeds this, the next Solution update will fail at patching the update service. If the error limit is configured higher than 8GB temporarily, it must be reset down to 8192 or lower before taking the next Solution update.

```powershell
$nodes = Get-ClusterNode | % Name

Write-Host "Running script block on nodes: $($nodes -join ', ')"
Invoke-Command -ComputerName $nodes -ScriptBlock {
    $ErrorActionPreference = "Stop"

    # Values in MB
    $memoryWarningLimit = 5632
    $memoryErrorLimit = 6144

    if ($memoryErrorLimit -lt 4096)
    {
        throw "[$($env:ComputerName)] Current memory limit is 4096. If this TSG is being used, the suggested memory limit should be higher."
    }

    if ($memoryErrorLimit -lt $memoryWarningLimit)
    {
        throw "[$($env:ComputerName)] Warning limit ($memoryWarningLimit) must be lower than the Error limit $($memoryErrorLimit)"
    }

    if (-not (Test-Path "C:\Agents\AgentManifests"))
    {
        # 2701 and later: no ALM. Report the limits from both files that hold them, and change nothing.
        $limitFiles = @(Join-Path $env:ProgramData "Microsoft\AzureStack\ResourceGovernance\Azure Stack HCI Update Service.json")
        $updateService = Get-CimInstance -ClassName Win32_Service -Filter "Name='Azure Stack HCI Update Service'"
        if ($updateService -and $updateService.PathName -match '^"?([^"]+\.exe)')
        {
            $limitFiles += Join-Path (Split-Path $Matches[1] -Parent) "AgentManifest.json"
        }
        else
        {
            Write-Host "[$($env:ComputerName)] The Azure Stack HCI Update Service is not installed on this node, so its AgentManifest.json is not checked"
        }
        $foundFiles = @($limitFiles | Where-Object { Test-Path -LiteralPath $_ })
        if ($foundFiles.Count -eq 0)
        {
            throw "[$($env:ComputerName)] Unable to find the update service memory limits at $($limitFiles -join ' or ')"
        }
        $lowerLimits = $false
        foreach ($limitFile in $limitFiles)
        {
            if ($foundFiles -notcontains $limitFile)
            {
                Write-Host "[$($env:ComputerName)] Not found: $limitFile"
                continue
            }
            $limits = @{}
            Select-String -LiteralPath $limitFile -Pattern '"(WarningMemoryLimitMB|ErrorMemoryLimitMB)"\s*:\s*(\d+)' -AllMatches |
                ForEach-Object { $_.Matches } | ForEach-Object { $limits[$_.Groups[1].Value] = [int]$_.Groups[2].Value }
            Write-Host "[$($env:ComputerName)] $limitFile sets Warning limit $($limits["WarningMemoryLimitMB"])MB and Error limit $($limits["ErrorMemoryLimitMB"])MB"
            if (-not ($limits["WarningMemoryLimitMB"] -ge $memoryWarningLimit -and $limits["ErrorMemoryLimitMB"] -ge $memoryErrorLimit))
            {
                $lowerLimits = $true
            }
        }
        if ($lowerLimits)
        {
            throw "[$($env:ComputerName)] The update service memory limits are lower than the suggested values. Don't edit these files; contact Microsoft Support."
        }
        Write-Host "[$($env:ComputerName)] No changes needed. The limits are at least the suggested values."
        return
    }

    $updateAgentManifestFileName = ls "C:\Agents\AgentManifests\*Update Service*"
    if ($updateAgentManifestFileName.Count -gt 1)
    {
        throw "[$($env:ComputerName)] Unexpectedly found more than one update service agent manifest in C:\Agents"
    }

    if (-not $updateAgentManifestFileName)
    {
        throw "[$($env:ComputerName)] Unable to find the update agent manifest at C:\Agents\AgentManifests"
    }

    Write-Host "[$($env:ComputerName)] Found update agent manifest at $($updateAgentManifestFileName.FullName)"
    $updateAgentManifest = Get-Content $updateAgentManifestFileName.FullName -Raw | ConvertFrom-Json

    if ($updateAgentManifest.WindowsServiceInstallationParameters.ResourceGovernanceConfiguration.WarningMemoryLimitMB -ne $memoryWarningLimit -or `
        $updateAgentManifest.WindowsServiceInstallationParameters.ResourceGovernanceConfiguration.ErrorMemoryLimitMB -ne $memoryErrorLimit)
    {
        Write-Host "[$($env:ComputerName)] Setting Warning limit to $($memoryWarningLimit)MB and Error limit to $($memoryErrorLimit)MB"
        $updateAgentManifest.WindowsServiceInstallationParameters.ResourceGovernanceConfiguration.WarningMemoryLimitMB = $memoryWarningLimit
        $updateAgentManifest.WindowsServiceInstallationParameters.ResourceGovernanceConfiguration.ErrorMemoryLimitMB = $memoryErrorLimit

        Write-Host "[$($env:ComputerName)] Saving edited agent manifest back to $($updateAgentManifestFileName.FullName)"
        $updateAgentManifest | ConvertTo-Json -Depth 10 | Set-Content -Path $updateAgentManifestFileName.FullName

        Write-Host "[$($env:ComputerName)] Restarting ALM agent to pick up new config."
        Get-Service "AzureStack Agent Lifecycle Agent" | Restart-Service

        $updateService = Get-Service "Azure Stack HCI Update Service"
        if ($updateService.Status -eq "Running")
        {
            Write-Host "[$($env:ComputerName)] Restarting update service to be monitored with new memory limit values."
            $updateService | Restart-Service
        }
        else
        {
            Write-Host "[$($env:ComputerName)] Update service is not primary on this node. Current status: $($updateService.Status)"
        }
    }
    else
    {
        Write-Host "[$($env:ComputerName)] No changes needed. Existing limits are already set. Warning limit $($memoryWarningLimit)MB and Error limit $($memoryErrorLimit)MB"
    }
}
```


