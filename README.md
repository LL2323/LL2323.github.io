resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  acl    = var.acl

  versioning {
    enabled = var.versioning
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = var.sse_algorithm
      }
    }
  }

  tags = var.tags

  # Optional: ensure bucket is destroyed if module is destroyed
  force_destroy = var.force_destroy
}
