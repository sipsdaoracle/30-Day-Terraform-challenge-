# Day 6: Understanding Terraform State

## Participant Details
- **Name:** Siphokazi Dolo
- **Task Completed:** Understanding Terraform State
- **Date and Time:** 08/31/24 12:34 PM

Terraform state is like a snapshot of your infrastructure which helps Terraform know what resources exist and how to manage them. It's stored in a file, which keeps track of details like resource IDs and dependencies, allowing Terraform to make only the necessary changes when you update your infrastructure. In a team, it's important to store this state remotely (e.g., in AWS S3) so everyone works from the same source and to avoid conflicts. Using remote state with locking ensures that only one person can make changes at a time, preventing issues and making your infrastructure consistent and reliable

### Terraform Code

## main.tf
```
# S3 backend configuration
resource "aws_s3_bucket" "terraform_state" {
  bucket = "sips-terraform-state-bucket"


  lifecycle {
    prevent_destroy = true
  }
}


resource "aws_s3_bucket_versioning" "enabled" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}


resource "aws_s3_bucket_server_side_encryption_configuration" "default" {
  bucket = aws_s3_bucket.terraform_state.id


  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}


resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket = aws_s3_bucket.terraform_state.id


  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}


resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-up-and-running-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"


  attribute {
    name = "LockID"
    type = "S"
  }
}


# Outputs
output "alb_dns_name" {
  value       = aws_lb.alb.dns_name
  description = "The domain name of the load balancer"
}
```
### backend.tf
```
terraform {
  backend "s3" {
    bucket         = "sips-terraform-state-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-up-and-running-locks"
    encrypt        = true
  }
}

```

