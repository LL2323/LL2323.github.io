#-----------------------------------------------------------------------------
# Deploy S3 Buckets
#-----------------------------------------------------------------------------
module "s3_bucket" {
  for_each = var.buckets

  source = "./modules/s3_bucket"

  bucket_name  = each.value.bucket_name
  versioning   = each.value.versioning
  kms_key_arn  = var.kms_key_id
  tags         = each.value.tags
}
