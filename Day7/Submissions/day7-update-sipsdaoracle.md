# Day 7: Understanding Terraform State Part 2

## Participant Details

- **Name:** Siphokazi Dolo
- **Task Completed:** Understanding Terraform State Part 2
- **Date and Time:** 02-09-2024 at 19:27 pm

  ## T

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

