# Day 5: Scaling Infrastructure

## Participant Details
- **Name:** Siphokazi Dolo
- **Task Completed:** Scaling Infrastructure
- **Date and Time:** 2024-08-26 14:51pm

### Terraform Code

### main.tf

```
# Provider configuration
provider "aws" {
  region = "us-east-1"
}

# Use of variables for better configurability
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "server_port" {
  default = 80
}

# Data sources
data "aws_availability_zones" "available" {}

# VPC and Networking Resources
resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
}

resource "aws_subnet" "public_subnets" {
  count                   = 3
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet ${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "Public Route Table"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public_subnets)
  subnet_id      = aws_subnet.public_subnets[count.index].id
  route_table_id = aws_route_table.public.id
}

# Security Groups
resource "aws_security_group" "alb" {
  name   = "alb-security-group"
  vpc_id = aws_vpc.vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "instance" {
  name   = "instance-security-group"
  vpc_id = aws_vpc.vpc.id

  ingress {
    from_port       = var.server_port
    to_port         = var.server_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Load Balancer and related resources
resource "aws_lb" "alb" {
  name               = "web-alb"
  load_balancer_type = "application"
  subnets            = aws_subnet.public_subnets[*].id
  security_groups    = [aws_security_group.alb.id]
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404: page not found"
      status_code  = 404
    }
  }
}

resource "aws_lb_target_group" "asg" {
  name     = "asg-target-group"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = aws_vpc.vpc.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener_rule" "asg" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg.arn
  }
}

# Launch Template and Auto Scaling Group
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = "ami-0c55b159cbfafe1f0"  # Replace with your desired AMI
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.instance.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup python -m SimpleHTTPServer ${var.server_port} &
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "web-server"
    }
  }
}

resource "aws_autoscaling_group" "web" {
  vpc_zone_identifier = aws_subnet.public_subnets[*].id
  target_group_arns   = [aws_lb_target_group.asg.arn]
  health_check_type   = "ELB"
  min_size            = 2
  max_size            = 5

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "web-server-asg"
    propagate_at_launch = true
  }
}

# S3 backend configuration
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-unique-terraform-state-bucket-name"

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

### Comaprison Table

| **Block Type**     | **Description**                                                               | **Code Example**                                                                                       | **Use Case**                                     |
|--------------------|-------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `provider`         | Defines the provider to interact with cloud platforms or APIs.                | `provider "aws" { region = "us-east-1" }`                                                               | Set AWS as the cloud provider for Terraform      |
| `resource`         | Defines the resource that Terraform manages.                                  | `resource "aws_instance" "web" { ami = "ami-12345" instance_type = "t2.micro" }`                        | Create resources like EC2 instances, S3 buckets  |
| `data`             | Retrieves information from existing resources in AWS or other platforms.      | `data "aws_ami" "ubuntu" { most_recent = true filter { name = "name" values = ["ubuntu/images/*"] }}`    | Look up AMI IDs, fetch existing resources        |
| `variable`         | Defines input variables to make configurations more dynamic and reusable.     | `variable "instance_type" { default = "t2.micro" }`                                                     | Make instance types, regions configurable        |
| `output`           | Displays output values like resource IDs, public IPs, or any computed values. | `output "instance_id" { value = aws_instance.web.id }`                                                  | Retrieve and display the result of operations    |
| `locals`           | Defines local variables to simplify and reuse expressions in your configuration. | `locals { environment = "production" }`                                                                 | Reuse common values in multiple places           |
| `module`           | A reusable block of Terraform code that can be shared across different environments. | `module "vpc" { source = "terraform-aws-modules/vpc/aws" version = "2.78.0"}`                           | Encapsulate and reuse sets of resources          |

