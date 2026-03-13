variable "create_server" {
  description = "Whether to create Windows server(s)"
  type        = bool
  default     = false
}

variable "create_s3_bucket" {
  description = "Whether to create S3 bucket(s)"
  type        = bool
  default     = false
}
