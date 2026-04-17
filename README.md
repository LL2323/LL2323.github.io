  security_profile {
    security_type = "TrustedLaunch"
    uefi_settings {
      secure_boot_enabled = true
      vtpm_enabled        = true
    }
  }
