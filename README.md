servers = {
  dc01 = {
    windows_version        = "2022"
    hostname               = "dc01"
    instance_type          = "t3.large"
    subnet_id              = "subnet-123"
    security_group_ids     = ["sg-123"]
    key_name               = "OSSTEAMKEY"
    additional_user_data   = ""
    default_user_groups    = ["Administrators"]
    additional_user_groups = ["ITAdmins"]
    tags = {
      Environment = "Dev"
    }
  }

  app01 = {
    windows_version        = "2025"
    hostname               = "app01"
    instance_type          = "t3.large"
    subnet_id              = "subnet-456"
    security_group_ids     = ["sg-456"]
    key_name               = "OSSTEAMKEY"
    additional_user_data   = ""
    default_user_groups    = ["Administrators"]
    additional_user_groups = ["AppAdmins"]
    tags = {
      Environment = "Dev"
    }
  }

  file01 = {
    windows_version        = "2019"
    hostname               = "file01"
    instance_type          = "t3.large"
    subnet_id              = "subnet-789"
    security_group_ids     = ["sg-789"]
    key_name               = "OSSTEAMKEY"
    additional_user_data   = ""
    default_user_groups    = ["Administrators"]
    additional_user_groups = ["FileAdmins"]
    tags = {
      Environment = "Prod"
    }
  }
}
