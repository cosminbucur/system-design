# Terraform Guide: Concepts & AWS Step-by-Step Example

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool created by HashiCorp. It allows engineers to define, provision, and manage cloud and on-premises resources using human-readable configuration files written in **HashiCorp Configuration Language (HCL)**.

Instead of manually clicking through cloud consoles or writing ad-hoc custom scripts, Terraform lets you treat infrastructure like application code—versioning it in Git, reviewing changes before applying them, and tearing down or recreating environments consistently.

---

## How Terraform Works (Core Workflow)

1. **Write:** Declare the desired end-state of infrastructure (e.g., virtual machines, load balancers, databases) in `.tf` files.
2. **Init (`terraform init`):** Initializes the working directory and downloads necessary provider plugins.
3. **Plan (`terraform plan`):** Compares the desired configuration against actual real-world state and outputs an execution plan.
4. **Apply (`terraform apply`):** Executes the plan to reach the targeted state.

---

## Key Topics & Concepts for Technical Discussions

Whether preparing for a design review, technical interview, or architectural planning, these are critical topics to cover:

### 1. Declarative vs. Imperative Infrastructure
* **Declarative Approach:** Terraform uses a declarative model where you specify **what** the final infrastructure should look like, not **how** to create it step-by-step. Terraform automatically computes dependencies and executes actions in parallel where possible.
* **Discussion Angle:** Comparing Terraform with imperative tools (like custom Python/Bash scripts or Ansible for configuration management).

### 2. Providers & Data Sources
* **Providers:** Plugins that translate HCL into specific cloud API calls (e.g., AWS, Azure, GCP, Kubernetes, Cloudflare).
* **Data Sources:** Allow Terraform to fetch and reference existing infrastructure configured outside the current root module (e.g., looking up an existing VPC ID or AMI).

### 3. State File Management (`.tfstate`)
* **Role of State:** Terraform keeps track of the mappings between real-world infrastructure and local code in a state file.
* **Remote Backends:** In production, state files must be stored in a secure remote backend (e.g., AWS S3 with DynamoDB locking, HashiCorp Cloud) to allow team collaboration and prevent concurrent runs.
* **Discussion Angle:** State security/encryption (state files contain plain-text secrets), state locking mechanisms, and recovering from corrupted state or drift.

### 4. Modules & Dry Code Structure
* **Modules:** Reusable containers for multiple resources that group related infrastructure together (e.g., a standardized VPC module or Kubernetes cluster module).
* **Discussion Angle:** Structuring monolithic vs. granular modules, module versioning strategies, and private registries.

### 5. Managing Drift & Immutability
* **Drift Detection:** Happens when manual changes are made to resources outside of Terraform. Running `terraform plan` detects discrepancies between real resources and the state file.
* **Immutable Infrastructure:** Emphasizes tearing down and replacing instances/resources with new configurations rather than patching running servers in-place.

### 6. Workspaces vs. Multi-Directory Architecture
* **Workspaces:** Allow managing multiple environments (Dev, Staging, Prod) within the same configuration directory using isolated state files.
* **Multi-Directory Approach:** Keeping separate folders (e.g., `environments/dev`, `environments/prod`) using tools like Terragrunt or native Terraform backend configurations.
* **Discussion Angle:** Pros and cons of using workspaces versus explicit directory separations for environment isolation.

### 7. Licensing & Ecosystem Shifts (OpenTofu)
* **License Shift:** HashiCorp transitioned Terraform from an open-source license (MPL-2.0) to a Business Source License (BSL).
* **OpenTofu:** A community-driven, Linux Foundation-backed fork of Terraform created in response to the license change.
* **Discussion Angle:** Evaluating vendor lock-in, open-source compliance, and choosing between HashiCorp Terraform and OpenTofu.

---

## Step-by-Step AWS Infrastructure Example

Below is a complete example deploying a basic **AWS EC2 instance** inside a dedicated **VPC (Virtual Private Cloud)** and **Security Group**.

### Prerequisites
1. **Terraform CLI** installed locally.
2. **AWS CLI** installed and configured (`aws configure` with an Access Key and Secret Key).

### Step 1: Create the Configuration File (`main.tf`)

```hcl
# 1. Specify required providers and Terraform version
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 2. Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

# 3. Create a Virtual Private Cloud (VPC)
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "demo-vpc"
  }
}

# 4. Create a Public Subnet inside the VPC
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.main_vpc.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "demo-public-subnet"
  }
}

# 5. Create an Internet Gateway for public access
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "demo-igw"
  }
}

# 6. Create a Route Table routing internet traffic to the Gateway
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "demo-public-rt"
  }
}

# Associate Route Table with the Subnet
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# 7. Create a Security Group (Firewall) allowing SSH & HTTP
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Allow SSH and HTTP traffic"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # For demo purposes; restrict to your IP in production
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 8. Look up the latest Amazon Linux 2023 AMI dynamically
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

# 9. Provision the EC2 Instance running Nginx
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public_subnet.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              dnf update -y
              dnf install -y nginx
              systemctl start nginx
              systemctl enable nginx
              echo "<h1>Hello from Terraform!</h1>" > /usr/share/nginx/html/index.html
              EOF

  tags = {
    Name = "demo-web-server"
  }
}

# 10. Output the Public IP of the Instance
output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web_server.public_ip
}
```

### Step 2: Provisioning Commands

1. **Initialize Terraform:**
   ```bash
   terraform init
   ```
2. **Preview Changes:**
   ```bash
   terraform plan
   ```
3. **Apply Configuration:**
   ```bash
   terraform apply
   ```
4. **Clean Up Resources:**
   ```bash
   terraform destroy
   ```