packer {
  required_version = ">= 1.9.0"

  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "region" {
  default = "us-east-1"
}

source "amazon-ebs" "windows" {
  region           = var.region
  instance_type    = "t3.xlarge"
  communicator     = "winrm"
  winrm_username   = "Administrator"
  winrm_use_ssl    = false
  winrm_insecure   = true
  security_group_ids = ["sg-053861bc2ca0d590c","sg-0f496e3055f21c21d"]
  subnet_id = "subnet-0f359b98a86b90e67"
  ssh_keypair_name  = "PACKERKEY"
  ssh_private_key_file = "PACKERKEY.pem"
  iam_instance_profile = "EC2-S3-Installer-ReadOnly-Profile"
  disable_stop_instance = true

  aws_polling {
    delay_seconds = 30
    max_attempts  = 240
  }

  ami_users = ["325484260930", "886315010003", "097608627441", "752438423036", "075523027401", "232763445789", "606410660423", "092491373872",
                "986502462268", "039815767409", "201334828778", "687792729971", "290568021089", "891376958044", "676206932188", "664418982855",
                "703671918405", "264193146292"]
  run_tags = {
    Name = "SDOSS2K19AMI"
  }

# Root volume size during the build (temporary instance)
launch_block_device_mappings {
  device_name           = "/dev/sda1"
  volume_size           = 90
  volume_type           = "gp3"
  delete_on_termination = true
  encrypted             = true
}

# Root volume size recorded in the AMI you produce
ami_block_device_mappings {
  device_name           = "/dev/sda1"
  volume_size           = 90
  volume_type           = "gp3"
  delete_on_termination = true
  encrypted             = true
}

  # Enable WinRM on first boot
  user_data_file = "../scripts/01-enable-winrm.ps1"

  source_ami_filter {
    filters = {
      name = "Windows_Server-2019-English-Full-Base-*"
    }
    owners      = ["amazon"]
    most_recent = true
  }

  ami_name = "SDOSS2K19AMI-D{{isotime \"20060102\"}}"
}

build {
  name    = "SDOSS2K19AMI"
  sources = ["source.amazon-ebs.windows"]

  # Rename Template
  provisioner "powershell" {
  inline = [
    "Rename-Computer -NewName 'SDOSS2K19AMI' -Force -Restart"
  ]
  }

  # Install AWS CLI v2
  provisioner "powershell" {
    scripts = ["../scripts/02-setup-awscli.ps1"]
  }

    # Setup install folder
  provisioner "powershell" {
    inline = [
      "New-Item -Path C:/installers -ItemType Directory -Force"
    ]
  }

  # Download installers from S3
  provisioner "powershell" {
    inline = [
      "aws s3 sync s3://ams3osstmp0001/installers/ C:/installers/"
    ]
  }

  # Setup and necessary install files from C:/installers
  provisioner "powershell" {
    scripts = ["../scripts/03-install-local-software.ps1"]
  }

  # Reboot OS
  provisioner "windows-restart" {
  }

  # WSUS update process
  provisioner "powershell" {
    scripts = ["../scripts/04-wsus-update.ps1"]
  }

  # Reboot OS
  provisioner "windows-restart" {
  }

  # Cleanup before sysprep
  provisioner "powershell" {
    scripts = ["../scripts/05-cleanup.ps1"]
  }

  # Reboot OS
  provisioner "windows-restart" {
  }

  # Run EC2 sysprep using the tooling available on the AMI
  provisioner "powershell" {
    inline = [
      "$ec2LaunchExe = 'C:\\Program Files\\Amazon\\EC2Launch\\EC2Launch.exe'",
      "$ec2ConfigInit = 'C:\\ProgramData\\Amazon\\EC2-Windows\\Launch\\Scripts\\InitializeInstance.ps1'",
      "$ec2ConfigSysprep = 'C:\\ProgramData\\Amazon\\EC2-Windows\\Launch\\Scripts\\SysprepInstance.ps1'",
      "$ec2LaunchMsi = 'C:\\installers\\EC2LaunchV2\\AmazonEC2Launch.msi'",
      "if ((-not (Test-Path $ec2LaunchExe)) -and (Test-Path $ec2LaunchMsi)) { Start-Process msiexec.exe -ArgumentList '/i', $ec2LaunchMsi, '/qn', '/norestart' -Wait }",
      "if (Test-Path $ec2LaunchExe) { & $ec2LaunchExe reset --block; & $ec2LaunchExe sysprep --shutdown --block } elseif ((Test-Path $ec2ConfigInit) -and (Test-Path $ec2ConfigSysprep)) { & $ec2ConfigInit -Schedule; & $ec2ConfigSysprep -NoShutdown; Stop-Computer -Force } else { throw 'No supported EC2 sysprep tooling found on this AMI.' }"
    ]
  }
}
