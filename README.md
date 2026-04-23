
# Windows Update - pass 1 (main patching)
provisioner "windows-update" {
  pause_before    = "30s"

  search_criteria = "IsInstalled=0 and Type='Software' and IsAssigned=1"

  filters = [
    "exclude:$.Title -like '*Preview*'",
    "exclude:$.Title -like '*Driver*'",
    "exclude:$.Title -like '*Definition Update*'",
    "exclude:$.Title -like '*Defender*'"
  ]

  restart_timeout     = "60m"
  update_max_retries  = 8
}
