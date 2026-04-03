
# Prompt for temporary AWS credentials before running builds
$awsAccessKeyId = Read-Host "Enter AWS Access Key ID"
$awsSecretAccessKey = Read-Host "Enter AWS Secret Access Key"
$awsSessionToken = Read-Host "Enter AWS Session Token"

if ([string]::IsNullOrWhiteSpace($awsAccessKeyId) -or
	[string]::IsNullOrWhiteSpace($awsSecretAccessKey) -or
	[string]::IsNullOrWhiteSpace($awsSessionToken)) {
	Write-Error "AWS Access Key, Secret Access Key, and Session Token are required."
	exit 1
}

aws configure set aws_access_key_id $awsAccessKeyId
aws configure set aws_secret_access_key $awsSecretAccessKey
aws configure set aws_session_token $awsSessionToken

Write-Host "AWS credentials configured. Starting Packer builds..."

Start-Process powershell.exe -ArgumentList "-Command", "\AWS_Templates\packer.exe init .\SDOSS2K19AMI.pkr.hcl"
Start-Process powershell.exe -ArgumentList "-NoExit", "-Command", "\AWS_Templates\packer.exe build .\SDOSS2K19AMI.pkr.hcl"

Start-Process powershell.exe -ArgumentList "-Command", "\AWS_Templates\packer.exe init .\SDOSS2K22AMI.pkr.hcl"
Start-Process powershell.exe -ArgumentList "-NoExit", "-Command", "\AWS_Templates\packer.exe build .\SDOSS2K22AMI.pkr.hcl"

Start-Process powershell.exe -ArgumentList "-Command", "\AWS_Templates\packer.exe init .\SDOSS2K25AMI.pkr.hcl"
Start-Process powershell.exe -ArgumentList "-NoExit", "-Command", "\AWS_Templates\packer.exe build .\SDOSS2K25AMI.pkr.hcl"
