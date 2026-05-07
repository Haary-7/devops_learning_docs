# Providers, Resources, and Data Sources

## Providers

Providers are plugins that interact with APIs to manage resources.

### Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }
}
```

### Provider Version Constraints

| Constraint | Meaning | Example |
|------------|---------|---------|
| `= 5.0.0` | Exactly | Only 5.0.0 |
| `!= 5.0.0` | Not equal | Any except 5.0.0 |
| `> 5.0.0` | Greater | 5.0.1, 5.1.0, etc. |
| `>= 5.0.0` | At least | 5.0.0, 5.1.0, etc. |
| `< 5.0.0` | Less than | 4.x, 3.x, etc. |
| `<= 5.0.0` | At most | 5.0.0, 4.x, etc. |
| `~> 5.0` | Pessimistic | >= 5.0, < 6.0 |
| `~> 5.0.0` | Pessimistic | >= 5.0.0, < 5.1.0 |

### Multiple Providers

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
  }
}
```

### Provider Authentication

#### AWS

```hcl
# 1. Environment variables (auto-detected)
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN

# 2. Shared credentials file (~/.aws/credentials)
provider "aws" {
  region  = "us-east-1"
  profile = "myprofile"
}

# 3. IAM Role (EC2, ECS, EKS)
# Automatic when running on AWS

# 4. Assume role
provider "aws" {
  assume_role {
    role_arn = "arn:aws:iam::123456789:role/TerraformRole"
  }
}
```

#### Azure

```hcl
provider "azurerm" {
  features {}

  # Authentication methods:
  # 1. Azure CLI (az login)
  # 2. Service Principal (env vars)
  # 3. Managed Identity
  # 4. OIDC (GitHub Actions, etc.)
}
```

#### GCP

```hcl
provider "google" {
  project = "my-project"
  region  = "us-central1"

  # credentials = file("~/.gcp/credentials.json")
  # Or use Application Default Credentials
}
```

## Resources

### Resource Syntax

```hcl
resource "<TYPE>" "<NAME>" {
  # Configuration arguments
  # = values, references, functions
}
```

### AWS Resource Examples

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
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
    Name = "web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
  EOF

  tags = {
    Name = "web-server"
  }
}

# S3 Bucket
resource "aws_s3_bucket" "app" {
  bucket = "my-app-bucket-unique-name"

  tags = {
    Name = "app-bucket"
  }
}

resource "aws_s3_bucket_versioning" "app" {
  bucket = aws_s3_bucket.app.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "app" {
  bucket = aws_s3_bucket.app.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Azure Resource Examples

```hcl
resource "azurerm_resource_group" "main" {
  name     = "my-rg"
  location = "East US"

  tags = {
    Environment = "production"
  }
}

resource "azurerm_virtual_network" "main" {
  name                = "main-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  admin_username      = "adminuser"
  network_interface_ids = [azurerm_network_interface.main.id]

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  tags = {
    Environment = "production"
  }
}
```

### GCP Resource Examples

```hcl
resource "google_compute_network" "main" {
  name                    = "main-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "main" {
  name          = "main-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
  network       = google_compute_network.main.id
}

resource "google_compute_instance" "web" {
  name         = "web-instance"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.main.id
  }
}
```

## Data Sources

Data sources read information about existing infrastructure.

### Why Use Data Sources?

- Look up AMI IDs dynamically
- Reference resources created outside Terraform
- Get VPC/subnet IDs from existing infrastructure
- Fetch current AWS account/region info

### Common Data Sources

#### AWS

```hcl
# Latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Latest Amazon Linux AMI
data "aws_ami" "amazon-linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# VPC
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Subnets
data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

# Availability Zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Current caller identity
data "aws_caller_identity" "current" {}

# Region
data "aws_region" "current" {}

# IAM role
data "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"
}

# SSM Parameter
data "aws_ssm_parameter" "db_password" {
  name = "/myapp/db/password"
}
```

#### Azure

```hcl
data "azurerm_resource_group" "main" {
  name = "existing-rg"
}

data "azurerm_virtual_network" "main" {
  name                = "existing-vnet"
  resource_group_name = data.azurerm_resource_group.main.name
}

data "azurerm_client_config" "current" {}
```

#### GCP

```hcl
data "google_compute_image" "debian" {
  family  = "debian-11"
  project = "debian-cloud"
}

data "google_client_config" "current" {}
```

### Using Data Sources

```hcl
# Use the AMI ID
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}

# Use VPC ID
resource "aws_subnet" "public" {
  vpc_id     = data.aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

## Resource Import

Bring existing infrastructure under Terraform management.

```bash
# Import existing resource
terraform import aws_instance.web i-0abc123def456

# Import S3 bucket
terraform import aws_s3_bucket.my_bucket my-bucket-name

# Import module resource
terraform import 'module.vpc.aws_vpc.main' vpc-123456
```

**Workflow:**
1. Write the resource block in Terraform config
2. Run `terraform import <address> <id>`
3. Run `terraform plan` to see differences
4. Update config to match existing resource
5. Run `terraform apply` (should show no changes)

## Interview Questions and Answers

### Q: What is a Terraform provider?

A provider is a plugin that translates Terraform configuration into API calls for a specific platform (AWS, Azure, GCP, etc.). Providers define resource types and data sources.

### Q: How do you manage provider versions?

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Use pessimistic constraints (`~>`) to allow minor/patch updates while preventing major version changes.

### Q: What is the difference between a resource and a data source?

- **Resource** - Manages infrastructure (creates, updates, deletes)
- **Data Source** - Reads existing infrastructure (read-only)

### Q: How to look up the latest AMI dynamically?

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

### Q: How to import existing resources into Terraform?

```bash
terraform import aws_instance.web i-0abc123def456
```

Then write the matching resource configuration and run `terraform plan` to verify.

### Q: What is the difference between `count` and `for_each`?

- `count` - Uses numeric index, good for identical resources
- `for_each` - Uses map/set keys, better when resources have unique attributes

`for_each` is preferred because refactoring is safer (resources identified by key, not position).
