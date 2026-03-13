variable "buckets" {
  description = "Map of S3 buckets to create"

  type = map(object({
    bucket_name = string
    kms_key_arn = string
    versioning  = bool
    tags        = map(string)
  }))

  default = {}
}
