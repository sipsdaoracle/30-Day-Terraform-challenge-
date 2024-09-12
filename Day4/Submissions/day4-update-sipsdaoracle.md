# Day 4: Mastering Basic Infrastructure

## Participant Details
- **Name:** Siphokazi Dolo
- **Task Completed:** Mastering Basic Infrastructure with Terraform
- **Date and Time:** 2024-08-21 22:00pm

### Configurable & Clustered Web Server:
This Terraform configuration is designed to deploy a web server on AWS, with the option to choose between a single configurable web server or a clustered setup with an Auto Scaling group and a load balancer.

### main.tf
```hcl
# Provider configuration
provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "server_port" {
  default = 80
}

variable "server_text" {
  default = "Hello, World!"
}

variable "enable_clustering" {
  default = true
}

# Data sources
data "aws_availability_zones" "available" {}
data "aws_ssm_parameter" "amazon_linux_2_ami" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

# VPC and Networking Resources
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet ${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "Public Route Table"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Security Groups
resource "aws_security_group" "web" {
  name        = "web-server-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
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

# Configurable Web Server
resource "aws_instance" "single_web_server" {
  count                  = var.enable_clustering ? 0 : 1
  ami                    = data.aws_ssm_parameter.amazon_linux_2_ami.value
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public[0].id
  user_data              = file("user_data.sh")

  tags = {
    Name = "Single Web Server"
  }
}

# Clustered Web Server
resource "aws_launch_template" "web_cluster" {
  name_prefix   = "web-cluster-"
  image_id      = data.aws_ssm_parameter.amazon_linux_2_ami.value
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = base64encode(file("user_data.sh"))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "Clustered Web Server"
    }
  }
}

resource "aws_autoscaling_group" "web_cluster" {
  count               = var.enable_clustering ? 1 : 0
  vpc_zone_identifier = aws_subnet.public[*].id
  desired_capacity    = 2
  max_size            = 4
  min_size            = 2

  launch_template {
    id      = aws_launch_template.web_cluster.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "Clustered Web Server"
    propagate_at_launch = true
  }
}

# Load Balancer (for clustered setup)
resource "aws_lb" "web" {
  count              = var.enable_clustering ? 1 : 0
  name               = "web-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_listener" "web" {
  count             = var.enable_clustering ? 1 : 0
  load_balancer_arn = aws_lb.web[0].arn
  port              = var.server_port
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web[0].arn
  }
}

resource "aws_lb_target_group" "web" {
  count    = var.enable_clustering ? 1 : 0
  name     = "web-tg"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

resource "aws_autoscaling_attachment" "web" {
  count                  = var.enable_clustering ? 1 : 0
  autoscaling_group_name = aws_autoscaling_group.web_cluster[0].name
  lb_target_group_arn    = aws_lb_target_group.web[0].arn
}

# Outputs
output "web_server_address" {
  value       = var.enable_clustering ? aws_lb.web[0].dns_name : aws_instance.single_web_server[0].public_ip
  description = "Address to access the web server"
}

output "server_text" {
  value       = var.server_text
  description = "Text displayed by the web server"
}

output "deployment_type" {
  value       = var.enable_clustering ? "Clustered" : "Configurable"
  description = "Type of web server deployment"
}

output "load_balancer_dns_name" {
  value       = var.enable_clustering ? aws_lb.web[0].dns_name : null
  description = "DNS name of the load balancer (for clustered setup)"
}
```

