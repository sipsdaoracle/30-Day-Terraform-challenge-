# Day 3: Deploying Basic Infrastructure with Terraform

## Participant Details

- **Name:** Siphokazi Dolo
- **Task Completed:** Deploying Basic Infrastructure with Terraform
- **Date and Time:** 2024-09-03 at 9:00 pm

### TERRAFORM Main.tf
``` Bash
# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"  # AWS region
}

# Create a security group
resource "aws_security_group" "instance" {
  name = "terraform-default-instance"
  
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_8080"
  }
}

# Create a new EC2 instance
resource "aws_instance" "default" {
  ami           = "ami-0182f373e66f89c85"  # Amazon Linux 2 AMI ID in us-east-1
  instance_type = "t2.micro"               # EC2 instance type
  
  vpc_security_group_ids = [aws_security_group.instance.id]
  
  user_data = <<-EOF
              #!/bin/bash
              echo "Hi, I'm Sips and I've just deployed my first server using terraform" > /var/www/html/index.html
              nohup busybox httpd -f -p 8080 &
              EOF
  
  tags = {
    Name = "terraform-default"
  }
}

# Output the EC2 instance ID
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.default.id
}

# Output the EC2 instance public IP
output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.default.public_ip
}

# Output the EC2 instance public DNS name
output "instance_public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.default.public_dns
}

```
