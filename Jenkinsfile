provider "aws" {
  region = "us-east-1"
}

# --- VPC ---
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "main-vpc" }
}

# --- Subnet ---
resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags = { Name = "main-subnet" }
}

# --- Security Group ---
resource "aws_security_group" "allow_ssh_http" {
  name        = "allow_ssh_http"
  description = "Allow SSH and HTTP"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
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

# --- Latest Amazon Linux 2 AMI ---
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# --- Frontend EC2 ---
resource "aws_instance" "frontend" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  key_name      = "ansible"
  subnet_id     = aws_subnet.main.id
  security_groups = [aws_security_group.allow_ssh_http.name]

  tags = { Name = "frontend" }
}

# --- Backend EC2 ---
resource "aws_instance" "backend" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  key_name      = "ansible"
  subnet_id     = aws_subnet.main.id
  security_groups = [aws_security_group.allow_ssh_http.name]

  tags = { Name = "backend" }
}

# --- Ansible inventory ---
resource "local_file" "ansible_inventory" {
  content = <<EOT
[frontend]
frontend ansible_host=${aws_instance.frontend.public_ip} ansible_user=ec2-user

[backend]
backend ansible_host=${aws_instance.backend.public_ip} ansible_user=ec2-user
EOT
  filename = "${path.module}/ansible_inventory.ini"
}
