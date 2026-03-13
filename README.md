# Windows server module
module "windows_server" {
  source = "./modules/windows_server"
  count  = var.create_server ? 1 : 0

  # Pass server variables
  servers = var.servers
  key_name = var.key_name
}

# S3 bucket module
module "app_logs_bucket" {
  source = "./modules/s3_bucket"
  count  = var.create_s3_bucket ? 1 : 0

  # Pass S3 variables
  bucket_name    = var.bucket_name
  acl            = var.acl
  versioning     = var.versioning
  sse_algorithm  = var.sse_algorithm
  tags           = var.tags
}
