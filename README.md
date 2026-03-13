variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "acl" {
  description = "Canned ACL for the bucket"
  type        = string
  default     = "private"
}

variable "versioning" {
  description = "Enable versioning on the bucket"
  type        = bool
  default     = true
}

variable "sse_algorithm" {
  description = "Server-side encryption algorithm"
  type        = string
  default     = "AES256"
}

variable "tags" {
  description = "Tags to apply to the bucket"
  type        = map(string)
  default     = {}
}

variable "force_destroy" {
  description = "Destroy bucket even if it contains objects"
  type        = bool
  default     = false
}
