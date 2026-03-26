---
layout: post
title:  "Using VPC Endpoints to enable connecting to EC2 instances in a private subnet with SSM (Terraform automation)"
---

# Context
Connecting to an EC2 instance in a private subnet using Session Manager comes with a bunch of prerequisites. Most importantly:

1. SSM Agent must be installed and running on the instance,
2. The instance must have network access to SSM endpoints,
3. Outbound HTTPS traffic between the EC2 and the SSM endpoints must be allowed,
4. VPC DNS resolution and Private DNS for endpoints must be enabled,
5. The instance must have appropriate IAM permissions.

# Solution
For the purpose of this lab, requirement 1. is met by using an AMI of Amazon Linux 2023, which comes with SSM Agent preinstalled. Additionally, IAM permissions are handled automatically for eligible instances by enabling DHMC.

For the remaining points, I first enable DNS resolution within the VPC. Then, in order to allow connection to the SSM endpoints within the private subnet, I use VPC Endpoints for SSM, SSMMessages and EC2Messages services with Private DNS enabled. Finally, Security Groups for EC2 and VPC Endpoints are configured to enable connection between them.

I initially created this setup manually in order to test it and managed to connect successfully. A high level overview of the architecture is presented below

![ssm-vpc-diagram](/static/ssm-vpc-diagram.svg)

The next step was to define it as IaC with Terraform.

# No connection after implementing IaC
After translating the steps performed in AWS Console into the Terraform template, I was unable to connect to the EC2 using Session Manager. 

![ec2-connect-status](/static/ec2-connect-status.png)

Based on the prerequisites, I once again verified that:
- VPC DNS resolution is enabled and endpoints have private DNS names - this was set up correctly,
- security groups assigned to VPC Endpoints allow inbound traffic - initially I configured them to allow all traffic.

The last thing I verified was an EC2 Security Group. I initially replicated the settings configured in the console - enabling all inbound connection temporarily. This turned out to be insufficient with Terraform.

When creating a new security group in AWS Console, it automatically adds a rule allowing all outbound traffic. This is not the case in Terraform, where this rule needs to be defined explicitly. After adding the outbound rule I managed to connect to the EC2 instance using SSM.

# Hardening the setup
Based on the minimum requirements for the setup to work, I further restricted the setup.

**EC2**:
- only allow outbound traffic to port 443 of the VPC Endpoints Security Group.

**VPC Endpoints**:
- only allow inbound traffic from EC2 Security Group.

# IaC Implementation
Below is the full Terraform template for the implemented architecture.

```hcl
provider "aws" {
    region = "us-east-1"
}

# Networking
resource "aws_vpc" "vpc-1" {
    cidr_block = "10.0.0.0/24"
    enable_dns_support = true
    enable_dns_hostnames = true

    tags = {
        Name = "my-vpc-1"
    }
}

resource "aws_subnet" "subnet-1a" {
    vpc_id = aws_vpc.vpc-1.id
    cidr_block = "10.0.0.0/25"

    availability_zone = "us-east-1a"

    tags = {
        Name = "my-subnet-1a"
    }
}

# VPC Endpoints
locals {
    interface_endpoints = toset([
        "ssm",
        "ssmmessages",
        "ec2messages"
    ])
}

resource "aws_vpc_endpoint" "interface" {
    for_each = local.interface_endpoints

    vpc_id = aws_vpc.vpc-1.id
    subnet_ids = [aws_subnet.subnet-1a.id]
    vpc_endpoint_type = "Interface"

    service_name = "com.amazonaws.us-east-1.${each.key}"
    private_dns_enabled = true
    security_group_ids = [aws_security_group.endpoint-sg.id]

    tags = {
        Name = "endpoint-${each.key}"
    }
}

# Security Group for VPC endpoints
resource "aws_security_group" "endpoint-sg" {
    name = "endpoint-sg"
    description = "Allow EC2 traffic."
    vpc_id = aws_vpc.vpc-1.id

    tags = {
        Name = "endpoint-sg"
    }
}

resource "aws_vpc_security_group_ingress_rule" "ingress-endpoint-sg" {
    security_group_id = aws_security_group.endpoint-sg.id
    referenced_security_group_id = aws_security_group.sg-ec2.id

    ip_protocol = -1
}

# EC2 Security Group
resource "aws_security_group" "sg-ec2" {
    name = "my-ec2"
    description = "Allows outbound traffic to Endpoint Security Group"
    vpc_id = aws_vpc.vpc-1.id

    tags = {
        Name = "my-ec2-sg"
    }
}

resource "aws_vpc_security_group_egress_rule" "sg-ec2-egress" {
    security_group_id = aws_security_group.sg-ec2.id
    referenced_security_group_id = aws_security_group.endpoint-sg.id
    ip_protocol = "tcp"
    from_port = 443
    to_port = 443
}

# EC2
resource "aws_instance" "my-ec2-1" {
    ami = data.aws_ssm_parameter.al2023.value
    instance_type = "t3.micro"

    subnet_id = aws_subnet.subnet-1a.id
    vpc_security_group_ids = [
        aws_security_group.sg-ec2.id,
    ]
    
    tags = {
        Name = "my-ec2-1"
    }
}

data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}
```