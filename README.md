$log = "C:\packer-debug-log.txt"

function Write-Log {
	param(
		[string]$Message,
		[string]$Level = "INFO"
	)

	$entry = "[{0}] [{1}] {2}" -f $Level, (Get-Date -Format "yyyy-MM-dd HH:mm:ss"), $Message
	Write-Host $entry
	Add-Content -Path $log -Value $entry
}

function Wait-ForWsusPolicy {
	param(
		[int]$TimeoutSeconds = 300
	)

	$policyPath = "HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate"
	$auPolicyPath = Join-Path $policyPath "AU"
	$deadline = (Get-Date).AddSeconds($TimeoutSeconds)

	do {
		$policyExists = Test-Path $policyPath
		$auPolicyExists = Test-Path $auPolicyPath

		if ($policyExists -and $auPolicyExists) {
			$policy = Get-ItemProperty -Path $policyPath -ErrorAction SilentlyContinue
			$auPolicy = Get-ItemProperty -Path $auPolicyPath -ErrorAction SilentlyContinue

			if (-not [string]::IsNullOrWhiteSpace($policy.WUServer) -and
				-not [string]::IsNullOrWhiteSpace($policy.WUStatusServer) -and
				$auPolicy.UseWUServer -eq 1) {
				Write-Log "WSUS policy detected. WUServer=$($policy.WUServer)"
				return $true
			}
		}

		Write-Log "Waiting for WSUS policy to be applied..."
		Start-Sleep -Seconds 15
	} while ((Get-Date) -lt $deadline)

	return $false
}

function Invoke-WsusClientRefresh {
	Write-Log "Refreshing group policy and Windows Update services"
	gpupdate.exe /target:computer /force | Out-Null

	foreach ($serviceName in @("wuauserv", "bits", "UsoSvc")) {
		$service = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
		if ($null -ne $service) {
			if ($service.StartType -eq "Disabled") {
				Set-Service -Name $serviceName -StartupType Manual
			}
			if ($service.Status -ne "Running") {
				Start-Service -Name $serviceName
			}
		}
	}

	Write-Log "Triggering WSUS report and scan cycle"
	& "$env:SystemRoot\System32\wuauclt.exe" /resetauthorization /detectnow
	& "$env:SystemRoot\System32\wuauclt.exe" /reportnow

	$usoClient = Join-Path $env:SystemRoot "System32\UsoClient.exe"
	if (Test-Path $usoClient) {
		& $usoClient RefreshSettings
		& $usoClient StartScan
	}
}

Write-Log "WSUS update script started"

if (-not (Get-Module -ListAvailable -Name PSWindowsUpdate)) {
	Write-Log "PSWindowsUpdate module was not found." "ERROR"
	exit 1
}

Import-Module PSWindowsUpdate -Force
Write-Log "PSWindowsUpdate imported"

if (-not (Wait-ForWsusPolicy)) {
	Write-Log "WSUS policy was not applied within the expected time window." "ERROR"
	exit 1
}

Invoke-WsusClientRefresh

Write-Log "Waiting 120 seconds so the new VM can register in WSUS before download/install begins"
Start-Sleep -Seconds 120

Write-Log "Requesting available updates from the configured update service"
$availableUpdates = Get-WindowsUpdate -AcceptAll -IgnoreReboot -ErrorAction Stop

if (-not $availableUpdates) {
	Write-Log "No applicable updates were offered by WSUS"
	exit 0
}

Write-Log ("{0} update(s) found. Starting download and install." -f $availableUpdates.Count)
Install-WindowsUpdate -AcceptAll -IgnoreReboot -ErrorAction Stop | Out-String | ForEach-Object {
	if (-not [string]::IsNullOrWhiteSpace($_)) {
		Add-Content -Path $log -Value $_.TrimEnd()
	}
}

Write-Log "Triggering a final report back to WSUS"
& "$env:SystemRoot\System32\wuauclt.exe" /reportnow

Write-Log "WSUS update script completed"
exit 0
