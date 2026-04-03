<#
.SYNOPSIS
Creates a detailed HTML overview report for all datastores in a vCenter environment.

.DESCRIPTION
Prompts the user for a vCenter server and credentials, connects using VMware PowerCLI,
and exports a readable HTML report with summary cards, datastore risk highlights,
and per-datastore detail sections.

.REQUIREMENTS
- VMware PowerCLI must be installed
- Network access to the target vCenter
- Account permissions to read datastore, host, VM, alarm, and tag information

.EXAMPLE
.\vcenter_datastore_overview_report.ps1

.EXAMPLE
.\vcenter_datastore_overview_report.ps1 -VCenterServer vcsa01.contoso.com -OutputPath C:\Reports\datastore-report.html
#>

[CmdletBinding()]
param(
    [string]$VCenterServer,
    [string]$OutputPath
)

$ErrorActionPreference = 'Stop'

function Convert-ToHumanSize {
    param([Nullable[Double]]$Bytes)

    if ($null -eq $Bytes) { return 'N/A' }
    if ($Bytes -ge 1TB) { return ('{0:N2} TB' -f ($Bytes / 1TB)) }
    if ($Bytes -ge 1GB) { return ('{0:N2} GB' -f ($Bytes / 1GB)) }
    if ($Bytes -ge 1MB) { return ('{0:N2} MB' -f ($Bytes / 1MB)) }
    return ('{0:N0} B' -f $Bytes)
}

function Join-Text {
    param([object[]]$Values)

    $clean = @($Values | Where-Object { $null -ne $_ -and ([string]$_).Trim() -ne '' } | Sort-Object -Unique)
    if ($clean.Count -eq 0) { return 'N/A' }
    return ($clean -join ', ')
}

function Escape-Html {
    param([object]$Text)

    if ($null -eq $Text) { return '' }
    return [System.Net.WebUtility]::HtmlEncode([string]$Text)
}

$scriptRoot = if ($PSScriptRoot) { $PSScriptRoot } else { (Get-Location).Path }
if (-not $OutputPath) {
    $OutputPath = Join-Path $scriptRoot ("vcenter-datastore-overview-{0}.html" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))
}

$outputFolder = Split-Path -Path $OutputPath -Parent
if ($outputFolder -and -not (Test-Path -Path $outputFolder)) {
    New-Item -ItemType Directory -Path $outputFolder -Force | Out-Null
}

if (-not $VCenterServer) {
    $VCenterServer = Read-Host 'Enter the vCenter Server FQDN or IP'
}

if ([string]::IsNullOrWhiteSpace($VCenterServer)) {
    throw 'A vCenter server name or IP is required.'
}

# Prompt for credentials before any connection attempt.
$Credential = Get-Credential -Message "Enter credentials for vCenter Server [$VCenterServer]"
if ($null -eq $Credential) {
    throw 'Credential prompt was cancelled.'
}

$powerCliModule = Get-Module -ListAvailable -Name VMware.PowerCLI | Sort-Object Version -Descending | Select-Object -First 1
if (-not $powerCliModule) {
    throw "VMware PowerCLI is not installed. Install it first with:`nInstall-Module VMware.PowerCLI -Scope CurrentUser"
}

Import-Module VMware.PowerCLI -ErrorAction Stop | Out-Null

try {
    Set-PowerCLIConfiguration -Scope Session -InvalidCertificateAction Ignore -Confirm:$false | Out-Null
} catch {
    Write-Warning 'Unable to set InvalidCertificateAction to Ignore for this session. Continuing anyway.'
}

$connection = $null

try {
    Write-Host "Connecting to vCenter server '$VCenterServer'..." -ForegroundColor Cyan
    $connection = Connect-VIServer -Server $VCenterServer -Credential $Credential -WarningAction SilentlyContinue -ErrorAction Stop

    Write-Host 'Collecting datastore inventory...' -ForegroundColor Cyan
    $reportTime = Get-Date
    $datastores = @(Get-Datastore | Sort-Object Name)

    if ($datastores.Count -eq 0) {
        throw "No datastores were returned from vCenter server '$VCenterServer'."
    }

    # Pull related inventory once for faster correlation.
    # Use the higher-level cmdlets for cluster/datastore-cluster/alarm metadata because
    # some PowerCLI versions reject StoragePod and Alarm when passed to Get-View -ViewType.
    $vmViews = @(Get-View -ViewType VirtualMachine -Property Name,Config.Template,Datastore -ErrorAction SilentlyContinue)
    $hostViews = @(Get-View -ViewType HostSystem -Property Name,Parent,Datastore -ErrorAction SilentlyContinue)
    $clusters = @(Get-Cluster -ErrorAction SilentlyContinue)
    $datastoreClusters = @(Get-DatastoreCluster -ErrorAction SilentlyContinue)

    $alarmDefinitions = @()
    try {
        $alarmDefinitions = @(Get-AlarmDefinition -ErrorAction SilentlyContinue)
    } catch {
        $alarmDefinitions = @()
    }

    $clusterNameById = @{}
    foreach ($cluster in $clusters) {
        if ($cluster.ExtensionData -and $cluster.ExtensionData.MoRef) {
            $clusterNameById[$cluster.ExtensionData.MoRef.Value] = $cluster.Name
        }
    }

    $storagePodNameById = @{}
    foreach ($datastoreCluster in $datastoreClusters) {
        if ($datastoreCluster.ExtensionData -and $datastoreCluster.ExtensionData.MoRef) {
            $storagePodNameById[$datastoreCluster.ExtensionData.MoRef.Value] = $datastoreCluster.Name
        }
    }

    $alarmNameById = @{}
    foreach ($alarmDefinition in $alarmDefinitions) {
        if ($alarmDefinition.ExtensionData -and $alarmDefinition.ExtensionData.MoRef) {
            $alarmKey = $alarmDefinition.ExtensionData.MoRef.Value
            $alarmValue = if ($alarmDefinition.ExtensionData.Info -and $alarmDefinition.ExtensionData.Info.Name) {
                $alarmDefinition.ExtensionData.Info.Name
            } else {
                $alarmDefinition.Name
            }
            $alarmNameById[$alarmKey] = $alarmValue
        }
    }

    $rows = foreach ($datastore in $datastores) {
        Write-Host ("Processing datastore: {0}" -f $datastore.Name) -ForegroundColor DarkGray

        $view = $datastore.ExtensionData
        $summary = $view.Summary
        $info = $view.Info

        $capacityBytes = [double]$summary.Capacity
        $freeBytes = [double]$summary.FreeSpace
        $usedBytes = $capacityBytes - $freeBytes
        $uncommittedBytes = if ($null -ne $summary.Uncommitted) { [double]$summary.Uncommitted } else { $null }
        $provisionedBytes = if ($null -ne $uncommittedBytes) { $usedBytes + $uncommittedBytes } else { $usedBytes }
        $percentFree = if ($capacityBytes -gt 0) { [Math]::Round(($freeBytes / $capacityBytes) * 100, 2) } else { 0 }

        $dsMoRef = $view.MoRef.Value

        $attachedVmViews = @($vmViews | Where-Object {
            $_.Datastore -and ($_.Datastore.Value -contains $dsMoRef)
        })
        $vmCount = @($attachedVmViews | Where-Object { -not $_.Config.Template }).Count
        $templateCount = @($attachedVmViews | Where-Object { $_.Config.Template }).Count

        $attachedHostViews = @($hostViews | Where-Object {
            $_.Datastore -and ($_.Datastore.Value -contains $dsMoRef)
        })
        $hostNames = @($attachedHostViews | ForEach-Object { $_.Name } | Sort-Object -Unique)
        $clusterNames = @($attachedHostViews | ForEach-Object {
            if ($_.Parent -and $clusterNameById.ContainsKey($_.Parent.Value)) {
                $clusterNameById[$_.Parent.Value]
            }
        } | Where-Object { $_ } | Sort-Object -Unique)

        $datacenterName = 'N/A'
        try {
            $dc = Get-Datacenter -Datastore $datastore -ErrorAction Stop | Select-Object -First 1
            if ($dc) { $datacenterName = $dc.Name }
        } catch {
            $datacenterName = 'N/A'
        }

        $datastoreCluster = if ($view.Parent -and $storagePodNameById.ContainsKey($view.Parent.Value)) {
            $storagePodNameById[$view.Parent.Value]
        } else {
            'Standalone'
        }

        $vmfsVersion = 'N/A'
        $vmfsUuid = 'N/A'
        $ssd = 'N/A'
        $local = 'N/A'
        $blockSizeMb = 'N/A'
        $extentCount = 'N/A'
        $extentDiskNames = 'N/A'
        $remoteHost = 'N/A'
        $remotePath = 'N/A'
        $nasVersion = 'N/A'

        if ($info -and $info.Vmfs) {
            $vmfsVersion = if ($info.Vmfs.Version) { $info.Vmfs.Version } else { 'N/A' }
            $vmfsUuid = if ($info.Vmfs.Uuid) { $info.Vmfs.Uuid } else { 'N/A' }
            $ssd = if ($null -ne $info.Vmfs.Ssd) { [string]$info.Vmfs.Ssd } else { 'N/A' }
            $local = if ($null -ne $info.Vmfs.Local) { [string]$info.Vmfs.Local } else { 'N/A' }
            $blockSizeMb = if ($null -ne $info.Vmfs.BlockSizeMb) { $info.Vmfs.BlockSizeMb } else { 'N/A' }
            $extentCount = @($info.Vmfs.Extent).Count
            if ($extentCount -eq 0) { $extentCount = 'N/A' }
            $extentDiskNames = Join-Text @($info.Vmfs.Extent | ForEach-Object { $_.DiskName })
        }

        if ($info -and $info.Nas) {
            $remoteHost = if ($info.Nas.RemoteHost) { $info.Nas.RemoteHost } else { 'N/A' }
            $remotePath = if ($info.Nas.RemotePath) { $info.Nas.RemotePath } else { 'N/A' }
            $nasVersion = if ($summary.Type) { $summary.Type } else { 'N/A' }
        } elseif ($summary.Url) {
            $remotePath = $summary.Url
        }

        $triggeredAlarmNames = @()
        if ($view.TriggeredAlarmState) {
            $triggeredAlarmNames = @($view.TriggeredAlarmState | ForEach-Object {
                if ($_.Alarm -and $alarmNameById.ContainsKey($_.Alarm.Value)) {
                    $alarmNameById[$_.Alarm.Value]
                } elseif ($_.Alarm) {
                    $_.Alarm.Value
                }
            } | Where-Object { $_ } | Sort-Object -Unique)
        }

        $tagNames = @()
        try {
            $tagNames = @(Get-TagAssignment -Entity $datastore -ErrorAction SilentlyContinue |
                Select-Object -ExpandProperty Tag |
                Select-Object -ExpandProperty Name |
                Sort-Object -Unique)
        } catch {
            $tagNames = @()
        }

        $storageIoControl = 'N/A'
        $congestionThreshold = 'N/A'
        if ($view.IormConfiguration) {
            if ($view.IormConfiguration.PSObject.Properties.Match('Enabled').Count -gt 0 -and $null -ne $view.IormConfiguration.Enabled) {
                $storageIoControl = [string]$view.IormConfiguration.Enabled
            } elseif ($view.IormConfiguration.PSObject.Properties.Match('StorageIORMEnabled').Count -gt 0 -and $null -ne $view.IormConfiguration.StorageIORMEnabled) {
                $storageIoControl = [string]$view.IormConfiguration.StorageIORMEnabled
            }

            if ($view.IormConfiguration.PSObject.Properties.Match('CongestionThreshold').Count -gt 0 -and $null -ne $view.IormConfiguration.CongestionThreshold) {
                $congestionThreshold = [string]$view.IormConfiguration.CongestionThreshold
            } elseif ($view.IormConfiguration.PSObject.Properties.Match('CongestionThresholdMillisecond').Count -gt 0 -and $null -ne $view.IormConfiguration.CongestionThresholdMillisecond) {
                $congestionThreshold = [string]$view.IormConfiguration.CongestionThresholdMillisecond
            }
        }

        [PSCustomObject]@{
            Name                = $datastore.Name
            Datacenter          = $datacenterName
            DatastoreCluster    = $datastoreCluster
            Type                = $summary.Type
            Accessible          = $summary.Accessible
            MaintenanceMode     = $summary.MaintenanceMode
            MultipleHostAccess  = $summary.MultipleHostAccess
            OverallStatus       = $view.OverallStatus
            CapacityGB          = [Math]::Round(($capacityBytes / 1GB), 2)
            FreeGB              = [Math]::Round(($freeBytes / 1GB), 2)
            UsedGB              = [Math]::Round(($usedBytes / 1GB), 2)
            PercentFree         = $percentFree
            UncommittedGB       = if ($null -ne $uncommittedBytes) { [Math]::Round(($uncommittedBytes / 1GB), 2) } else { $null }
            ProvisionedGB       = if ($null -ne $provisionedBytes) { [Math]::Round(($provisionedBytes / 1GB), 2) } else { $null }
            OverProvisioned     = if ($provisionedBytes -gt $capacityBytes) { 'Yes' } else { 'No' }
            VMCount             = $vmCount
            TemplateCount       = $templateCount
            HostCount           = $hostNames.Count
            Hosts               = Join-Text $hostNames
            ClusterCount        = $clusterNames.Count
            Clusters            = Join-Text $clusterNames
            Url                 = $summary.Url
            VMFSVersion         = $vmfsVersion
            VMFSUuid            = $vmfsUuid
            SSD                 = $ssd
            Local               = $local
            BlockSizeMB         = $blockSizeMb
            ExtentCount         = $extentCount
            ExtentDisks         = $extentDiskNames
            RemoteHost          = $remoteHost
            RemotePath          = $remotePath
            NASVersion          = $nasVersion
            StorageIOControl    = $storageIoControl
            CongestionThreshold = $congestionThreshold
            AlarmCount          = $triggeredAlarmNames.Count
            TriggeredAlarms     = if ($triggeredAlarmNames.Count -gt 0) { $triggeredAlarmNames -join ', ' } else { 'None' }
            Tags                = if ($tagNames.Count -gt 0) { $tagNames -join ', ' } else { 'None' }
        }
    }

    $totalCapacityGB = [double](($rows | Measure-Object -Property CapacityGB -Sum).Sum)
    $totalFreeGB = [double](($rows | Measure-Object -Property FreeGB -Sum).Sum)
    $avgFreePercent = if ($rows.Count -gt 0) {
        [Math]::Round((($rows | Measure-Object -Property PercentFree -Average).Average), 2)
    } else {
        0
    }

    $inaccessibleCount = @($rows | Where-Object { $_.Accessible -ne $true }).Count
    $maintenanceCount = @($rows | Where-Object { $_.MaintenanceMode -and $_.MaintenanceMode -ne 'normal' }).Count
    $lowFreeCount = @($rows | Where-Object { $_.PercentFree -lt 20 }).Count

    $typeSummary = $rows |
        Group-Object -Property Type |
        Sort-Object Name |
        ForEach-Object {
            [PSCustomObject]@{
                Type       = $_.Name
                Count      = $_.Count
                CapacityTB = [Math]::Round((($_.Group | Measure-Object -Property CapacityGB -Sum).Sum / 1024), 2)
                FreeTB     = [Math]::Round((($_.Group | Measure-Object -Property FreeGB -Sum).Sum / 1024), 2)
                AvgFreePct = [Math]::Round((($_.Group | Measure-Object -Property PercentFree -Average).Average), 2)
            }
        }

    $riskRows = @($rows | Where-Object {
        $_.PercentFree -lt 20 -or $_.Accessible -ne $true -or ($_.MaintenanceMode -and $_.MaintenanceMode -ne 'normal')
    } | Sort-Object PercentFree, Name)

    $summaryCards = @(
        "<div class='card'><div class='label'>Total Datastores</div><div class='value'>$($rows.Count)</div></div>",
        "<div class='card'><div class='label'>Total Capacity</div><div class='value'>$(Convert-ToHumanSize ($totalCapacityGB * 1GB))</div></div>",
        "<div class='card'><div class='label'>Total Free Space</div><div class='value'>$(Convert-ToHumanSize ($totalFreeGB * 1GB))</div></div>",
        "<div class='card'><div class='label'>Average Free %</div><div class='value'>$avgFreePercent%</div></div>",
        "<div class='card warn'><div class='label'>Below 20% Free</div><div class='value'>$lowFreeCount</div></div>",
        "<div class='card warn'><div class='label'>Inaccessible / Maintenance</div><div class='value'>$($inaccessibleCount + $maintenanceCount)</div></div>"
    ) -join "`n"

    $typeRowsHtml = foreach ($typeRow in $typeSummary) {
        "<tr><td>$(Escape-Html $typeRow.Type)</td><td>$($typeRow.Count)</td><td>$($typeRow.CapacityTB)</td><td>$($typeRow.FreeTB)</td><td>$($typeRow.AvgFreePct)%</td></tr>"
    }
    if (-not $typeRowsHtml) {
        $typeRowsHtml = "<tr><td colspan='5'>No datastore type data available.</td></tr>"
    }

    $riskRowsHtml = foreach ($riskRow in $riskRows) {
        "<tr><td>$(Escape-Html $riskRow.Name)</td><td>$(Escape-Html $riskRow.Type)</td><td>$($riskRow.CapacityGB)</td><td>$($riskRow.FreeGB)</td><td>$($riskRow.PercentFree)%</td><td>Accessible=$($riskRow.Accessible); Maintenance=$($riskRow.MaintenanceMode)</td></tr>"
    }
    if (-not $riskRowsHtml) {
        $riskRowsHtml = "<tr><td colspan='6'>No immediate datastore capacity or accessibility issues detected.</td></tr>"
    }

    $matrixRowsHtml = foreach ($row in ($rows | Sort-Object Name)) {
        $freeClass = if ($row.PercentFree -lt 20) { 'status-warn' } else { 'status-ok' }
        $accessText = if ($row.Accessible) { 'Yes' } else { 'No' }
        @"
<tr>
    <td>$(Escape-Html $row.Name)</td>
    <td>$(Escape-Html $row.Datacenter)</td>
    <td>$(Escape-Html $row.DatastoreCluster)</td>
    <td>$(Escape-Html $row.Type)</td>
    <td>$accessText</td>
    <td>$(Escape-Html $row.MaintenanceMode)</td>
    <td>$($row.CapacityGB)</td>
    <td>$($row.FreeGB)</td>
    <td>$($row.UsedGB)</td>
    <td class='$freeClass'>$($row.PercentFree)%</td>
    <td>$($row.UncommittedGB)</td>
    <td>$($row.ProvisionedGB)</td>
    <td>$(Escape-Html $row.OverProvisioned)</td>
    <td>$($row.VMCount)</td>
    <td>$($row.TemplateCount)</td>
    <td>$($row.HostCount)</td>
    <td>$(Escape-Html $row.OverallStatus)</td>
    <td>$($row.AlarmCount)</td>
</tr>
"@
    }

    $detailsHtml = foreach ($row in ($rows | Sort-Object Name)) {
        $detailPairs = [ordered]@{
            'Datacenter'           = $row.Datacenter
            'Datastore Cluster'    = $row.DatastoreCluster
            'Type'                 = $row.Type
            'Accessible'           = $row.Accessible
            'Maintenance Mode'     = $row.MaintenanceMode
            'Multiple Host Access' = $row.MultipleHostAccess
            'Overall Status'       = $row.OverallStatus
            'Capacity (GB)'        = $row.CapacityGB
            'Free Space (GB)'      = $row.FreeGB
            'Used Space (GB)'      = $row.UsedGB
            'Percent Free'         = ("{0}%" -f $row.PercentFree)
            'Uncommitted (GB)'     = $row.UncommittedGB
            'Provisioned (GB)'     = $row.ProvisionedGB
            'Over-Provisioned'     = $row.OverProvisioned
            'VM Count'             = $row.VMCount
            'Template Count'       = $row.TemplateCount
            'Host Count'           = $row.HostCount
            'Hosts'                = $row.Hosts
            'Cluster Count'        = $row.ClusterCount
            'Clusters'             = $row.Clusters
            'Datastore URL'        = $row.Url
            'VMFS Version'         = $row.VMFSVersion
            'VMFS UUID'            = $row.VMFSUuid
            'SSD'                  = $row.SSD
            'Local'                = $row.Local
            'Block Size (MB)'      = $row.BlockSizeMB
            'Extent Count'         = $row.ExtentCount
            'Extent Disks'         = $row.ExtentDisks
            'Remote Host'          = $row.RemoteHost
            'Remote Path'          = $row.RemotePath
            'NAS Version'          = $row.NASVersion
            'Storage I/O Control'  = $row.StorageIOControl
            'Congestion Threshold' = $row.CongestionThreshold
            'Triggered Alarms'     = $row.TriggeredAlarms
            'Tags'                 = $row.Tags
        }

        $pairRows = foreach ($key in $detailPairs.Keys) {
            "<tr><th>$(Escape-Html $key)</th><td>$(Escape-Html $detailPairs[$key])</td></tr>"
        }

        @"
<details class='detail-card'>
    <summary>$(Escape-Html $row.Name) <span class='muted'>| $(Escape-Html $row.Type) | $($row.CapacityGB) GB total</span></summary>
    <div class='detail-body'>
        <table class='kv-table'>
            $($pairRows -join "`n")
        </table>
    </div>
</details>
"@
    }

    $html = @"
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='utf-8' />
    <meta name='viewport' content='width=device-width, initial-scale=1' />
    <title>vCenter Datastore Overview Report</title>
    <style>
        body {
            font-family: Segoe UI, Arial, sans-serif;
            background: #f4f7fb;
            color: #1f2937;
            margin: 0;
            padding: 0;
        }
        .container {
            max-width: 1600px;
            margin: 0 auto;
            padding: 24px;
        }
        h1, h2 {
            margin-bottom: 10px;
        }
        .subtitle {
            color: #4b5563;
            margin-top: 0;
            margin-bottom: 20px;
        }
        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 14px;
            margin: 20px 0 28px;
        }
        .card {
            background: #ffffff;
            border-radius: 10px;
            box-shadow: 0 1px 4px rgba(0,0,0,0.08);
            padding: 16px;
            border-left: 5px solid #2563eb;
        }
        .card.warn {
            border-left-color: #dc2626;
        }
        .label {
            font-size: 12px;
            text-transform: uppercase;
            color: #6b7280;
            margin-bottom: 8px;
            letter-spacing: .4px;
        }
        .value {
            font-size: 24px;
            font-weight: 700;
        }
        .section {
            background: #ffffff;
            border-radius: 10px;
            box-shadow: 0 1px 4px rgba(0,0,0,0.08);
            padding: 18px;
            margin-bottom: 20px;
        }
        .table-wrap {
            overflow-x: auto;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            font-size: 13px;
        }
        th, td {
            border: 1px solid #e5e7eb;
            padding: 8px 10px;
            text-align: left;
            vertical-align: top;
        }
        th {
            background: #eff6ff;
            position: sticky;
            top: 0;
        }
        tr:nth-child(even) {
            background: #fafafa;
        }
        .status-ok {
            color: #15803d;
            font-weight: 700;
        }
        .status-warn {
            color: #b91c1c;
            font-weight: 700;
        }
        .detail-card {
            background: #ffffff;
            border: 1px solid #dbe3ef;
            border-radius: 10px;
            margin-bottom: 12px;
            overflow: hidden;
        }
        .detail-card summary {
            cursor: pointer;
            padding: 14px 16px;
            font-weight: 600;
            background: #f8fbff;
        }
        .detail-body {
            padding: 10px 16px 16px;
        }
        .kv-table th {
            width: 260px;
            background: #f9fafb;
            position: static;
        }
        .muted {
            color: #6b7280;
            font-weight: 400;
        }
        .footer {
            margin-top: 20px;
            color: #6b7280;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class='container'>
        <h1>vCenter Datastore Overview Report</h1>
        <p class='subtitle'>Server: <strong>$(Escape-Html $VCenterServer)</strong> &nbsp;|&nbsp; Generated: <strong>$(Escape-Html ($reportTime.ToString('yyyy-MM-dd HH:mm:ss')))</strong> &nbsp;|&nbsp; User: <strong>$(Escape-Html $connection.User)</strong></p>

        <div class='grid'>
            $summaryCards
        </div>

        <div class='section'>
            <h2>Datastore Type Summary</h2>
            <div class='table-wrap'>
                <table>
                    <thead>
                        <tr>
                            <th>Type</th>
                            <th>Count</th>
                            <th>Total Capacity (TB)</th>
                            <th>Total Free (TB)</th>
                            <th>Average Free %</th>
                        </tr>
                    </thead>
                    <tbody>
                        $($typeRowsHtml -join "`n")
                    </tbody>
                </table>
            </div>
        </div>

        <div class='section'>
            <h2>Capacity / Accessibility Watch List</h2>
            <div class='table-wrap'>
                <table>
                    <thead>
                        <tr>
                            <th>Name</th>
                            <th>Type</th>
                            <th>Capacity (GB)</th>
                            <th>Free (GB)</th>
                            <th>Free %</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        $($riskRowsHtml -join "`n")
                    </tbody>
                </table>
            </div>
        </div>

        <div class='section'>
            <h2>Full Datastore Matrix</h2>
            <div class='table-wrap'>
                <table>
                    <thead>
                        <tr>
                            <th>Name</th>
                            <th>Datacenter</th>
                            <th>Datastore Cluster</th>
                            <th>Type</th>
                            <th>Accessible</th>
                            <th>Maintenance</th>
                            <th>Capacity GB</th>
                            <th>Free GB</th>
                            <th>Used GB</th>
                            <th>Free %</th>
                            <th>Uncommitted GB</th>
                            <th>Provisioned GB</th>
                            <th>Over-Provisioned</th>
                            <th>VMs</th>
                            <th>Templates</th>
                            <th>Hosts</th>
                            <th>Overall Status</th>
                            <th>Alarm Count</th>
                        </tr>
                    </thead>
                    <tbody>
                        $($matrixRowsHtml -join "`n")
                    </tbody>
                </table>
            </div>
        </div>

        <div class='section'>
            <h2>Detailed Datastore Sections</h2>
            $($detailsHtml -join "`n")
        </div>

        <div class='footer'>
            Report saved to: $(Escape-Html $OutputPath)
        </div>
    </div>
</body>
</html>
"@

    $html | Out-File -FilePath $OutputPath -Encoding utf8

    Write-Host "HTML report created successfully: $OutputPath" -ForegroundColor Green
    Start-Process $OutputPath
}
finally {
    if ($connection) {
        Disconnect-VIServer -Server $connection -Confirm:$false | Out-Null
    }
}
