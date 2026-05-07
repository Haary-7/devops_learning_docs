# Terraform Comprehensive Cheat Sheet

## Commands

### Core Workflow

```bash
# Initialize
terraform init
terraform init -upgrade            # Upgrade providers
terraform init -backend=false      # Skip backend
terraform init -migrate-state      # Migrate backend
terraform init -reconfigure        # Reconfigure backend

# Plan
terraform plan
terraform plan -out=tfplan         # Save plan
terraform plan -target=aws_instance.web
terraform plan -var-file=prod.tfvars
terraform plan -refresh=false      # Skip refresh
terraform plan -detailed-exitcode

# Apply
terraform apply
terraform apply tfplan             # Apply saved plan
terraform apply -auto-approve
terraform apply -target=aws_instance.web
terraform apply -var-file=prod.tfvars
terraform apply -replace=aws_instance.web

# Destroy
terraform destroy
terraform destroy -auto-approve
terraform destroy -target=aws_instance.web
```

### State Management

```bash
# View state
terraform show
terraform show -json

# List/show resources
terraform state list
terraform state show aws_instance.web

# Modify state
terraform state mv old new
terraform state rm aws_instance.web

# Import
terraform import aws_instance.web i-0abc123
terraform import 'module.vpc.aws_vpc.main' vpc-123

# Pull/push
terraform state pull > state.json
terraform state push state.json

# Force unlock
terraform force-unlock <LOCK_ID>
```

### Workspaces

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select prod
terraform workspace show
terraform workspace delete dev
```

### Maintenance

```bash
# Format
terraform fmt
terraform fmt -recursive
terraform fmt -check        # CI mode

# Validate
terraform validate

# Providers
terraform providers
terraform providers lock

# Console
terraform console           # Interactive REPL

# Graph
terraform graph | dot -Tpng > graph.png

# Version
terraform version
```

### Debug

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply
# Log levels: TRACE, DEBUG, INFO, WARN, ERROR
```

## HCL Quick Reference

### Block Types

```hcl
terraform { }       # Terraform settings
provider "aws" { }  # Provider configuration
resource "type" "name" { }   # Infrastructure resource
data "type" "name" { }       # Data source lookup
variable "name" { }          # Input variable
output "name" { }            # Output value
locals { }                   # Local values
module "name" { }            # Child module
```

### Variables

```hcl
variable "name" {
  type        = string
  description = "Description"
  default     = "default"
  sensitive   = true
  nullable    = false

  validation {
    condition     = length(var.name) > 3
    error_message = "Name must be > 3 chars."
  }
}
```

### Outputs

```hcl
output "instance_ip" {
  description = "Public IP"
  value       = aws_instance.web.public_ip
  sensitive   = false
}
```

### Locals

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = merge(var.tags, { ManagedBy = "terraform" })
}
```

### Resource Meta-Arguments

```hcl
resource "aws_instance" "web" {
  count      = 3
  for_each   = var.instances
  depends_on = [aws_vpc.main]

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [ami, tags]
    replace_triggered_by  = [aws_launch_template.web.id]
  }
}
```

### Expressions

```hcl
# Reference
aws_instance.web.id
var.instance_type
local.name_prefix
data.aws_ami.ubuntu.id

# Conditional
var.env == "prod" ? "large" : "small"

# Splat
aws_instance.web[*].id
aws_instance.web.*.public_ip

# For expression
[for x in list : upper(x)]
{ for k, v in map : k => upper(v) }

# Splat with condition
[for i in aws_instance.web : i.id if i.tags.Env == "prod"]
```

### Functions

```hcl
# String
upper(), lower(), title(), join(), split(), replace(), substr(), format(), trim()

# Collection
length(), concat(), distinct(), flatten(), merge(), lookup(), keys(), values()
contains(), toset(), element(), coalesce(), compact()

# Numeric
max(), min(), abs(), floor(), ceil(), log(), pow()

# File
file(), filebase64(), templatefile(), jsonencode(), jsondecode()
yamlencode(), yamldecode(), path.module, path.root

# Crypto
base64encode(), base64decode(), md5(), sha256(), uuid(), bcrypt()

# Type
tostring(), tonumber(), tobool(), tolist(), toset(), tomap()
```

## Backend Configurations

### S3

```hcl
terraform {
  backend "s3" {
    bucket         = "my-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Azure

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "terraformstate"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
  }
}
```

### GCS

```hcl
terraform {
  backend "gcs" {
    bucket = "my-state-bucket"
    prefix = "prod"
  }
}
```

### Terraform Cloud

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "my-app-prod" }
  }
}
```

## Provider Setup

### AWS

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
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }
}
```

### Multiple Providers

```hcl
provider "aws" { region = "us-east-1"; alias = "east" }
provider "aws" { region = "us-west-2"; alias = "west" }

resource "aws_instance" "east" {
  provider = aws.east
  # ...
}
```

### Version Constraints

| Constraint | Meaning |
|------------|---------|
| `= 5.0.0` | Exactly |
| `!= 5.0.0` | Not equal |
| `> 5.0.0` | Greater |
| `>= 5.0.0` | At least |
| `< 5.0.0` | Less |
| `<= 5.0.0` | At most |
| `~> 5.0` | >= 5.0, < 6.0 |
| `~> 5.0.0` | >= 5.0.0, < 5.1.0 |

## Module Usage

### Local

```hcl
module "vpc" {
  source = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}
```

### Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
  name    = "my-vpc"
  cidr    = "10.0.0.0/16"
}
```

### Git

```hcl
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"
}
```

### With for_each

```hcl
module "env" {
  for_each    = var.environments
  source      = "./modules/environment"
  environment = each.key
}
```

## Common Patterns

### Conditional Resource

```hcl
resource "aws_instance" "optional" {
  count = var.create ? 1 : 0
  # ...
}
```

### Dynamic Block

```hcl
resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Multiple Environments

```hcl
locals {
  environment = terraform.workspace
  config = {
    dev  = { type = "t2.micro", count = 1 }
    prod = { type = "t2.large", count = 3 }
  }
}

resource "aws_instance" "web" {
  count         = local.config[local.environment].count
  instance_type = local.config[local.environment].type
}
```

### Remote State Reference

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "state-bucket"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_id
}
```

## JSONPath / State Queries

```bash
terraform state list
terraform state show aws_instance.web
terraform state pull | jq '.resources[].type'
terraform output -json
terraform show -json | jq '.values.root_module.resources[].type'
```

## Variable Passing

```bash
# Flag
terraform apply -var="name=value"

# File
terraform apply -var-file=prod.tfvars

# Environment
export TF_VAR_name="value"

# Auto-loaded (in order)
terraform.tfvars
terraform.tfvars.json
*.auto.tfvars (alphabetical)
```

## .gitignore

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
.terraform.lock.hcl
```

## Security Quick Reference

```hcl
# Sensitive variable
variable "password" { type = string; sensitive = true }

# Sensitive output
output "db_password" { value = var.password; sensitive = true }

# Encryption
resource "aws_ebs_volume" "data" { encrypted = true }

# SSM Parameter Store
data "aws_ssm_parameter" "password" { name = "/app/password" }

# Secrets Manager
data "aws_secretsmanager_secret_version" "creds" { secret_id = "app/creds" }
```

## CI/CD Pipeline

```yaml
- terraform fmt -check
- terraform init
- terraform validate
- terraform plan -out=tfplan
- terraform apply -auto-approve tfplan
```

## Resource Shortcuts (Terraform Addressing)

```
aws_instance.web                    # Resource
aws_instance.web[0]                 # With count
module.vpc.aws_vpc.main             # Module resource
module.vpc.aws_subnet.public[0]     # Nested module
data.aws_ami.ubuntu                 # Data source
var.instance_type                   # Variable
local.name_prefix                   # Local
output.instance_id                  # Output
```
