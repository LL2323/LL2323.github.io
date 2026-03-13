buckets = {
  logs = {
    bucket_name = "company-logs-001"

    kms_key_arn = "arn:aws:kms:us-east-2:123456789012:key/abcd1234-abcd-1234-abcd-123456789abc"

    versioning = true

    tags = {
      Environment = "Prod"
      Owner       = "DevOps"
    }
  }
}
