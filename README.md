#-----------------------------------------------------------------------------
# Root Variables.tf for HCP .tfvars
#-----------------------------------------------------------------------------

variable "aws_region" {
  type = string
}

variable "ami_owner" {
  type = string
}

variable "ami_name" {
  type = map(string)
}

variable "workspace" {
  type = string
}

variable "key_name" {
  type = string
}
#-----------------------------------------------------------------------------
variable "servers" {
  description = "Map of servers to deploy"
  type = map(object({
    windows_version        = string
    hostname               = string
    instance_type          = string
    subnet_id              = string
    security_group_ids     = list(string)
    tags                   = map(string)

    # Root drive
    root_volume_size       = number
    root_volume_type       = string

    # Optional additional drives
    additional_volumes = optional(list(object({
      device_name = string
      size        = number
      type        = string
    })), [])
  }))
}
#-----------------------------------------------------------------------------
variable "kms_key_id" {
  description = "KMS key ARN or ID used to encrypt all S3 buckets"
  type        = string
}

variable "buckets" {
  description = "Map of S3 buckets to create"

  type = map(object({
    bucket_name = string
    versioning  = bool
    tags        = map(string)
  }))

  default = {}
}
