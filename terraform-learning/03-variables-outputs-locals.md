# Variables, Outputs, and Locals

## Variables

### Variable Declaration

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"

  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium", "t2.large"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, t2.medium, or t2.large."
  }
}
```

### Variable Types

```hcl
# String
variable "name" {
  type    = string
  default = "my-app"
}

# Number
variable "port" {
  type    = number
  default = 80
}

# Boolean
variable "enabled" {
  type    = bool
  default = true
}

# List
variable "subnets" {
  type    = list(string)
  default = ["subnet-1", "subnet-2"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    Environment = "production"
    Team        = "platform"
  }
}

# Object
variable "server" {
  type = object({
    name          = string
    instance_type = string
    disk_size     = number
    tags          = map(string)
  })
}

# Any type (avoid when possible)
variable "config" {
  type = any
}

# Nullable (allows null as valid value)
variable "optional_name" {
  type     = string
  nullable = true
  default  = null
}

# Sensitive (hides in output)
variable "db_password" {
  type      = string
  sensitive = true
}
```

### Passing Variables

```bash
# 1. Command line
terraform apply -var="instance_type=t2.large"

# 2. Variable file
terraform apply -var-file=prod.tfvars
terraform apply -var-file=prod.tfvars.json

# 3. Environment variables
export TF_VAR_instance_type="t2.large"

# 4. terraform.tfvars or terraform.tfvars.json (auto-loaded)

# 5. .auto.tfvars files (auto-loaded, sorted alphabetically)

# 6. Interactive prompt
terraform apply  # Prompts for unset variables
```

### Loading Order (last wins)

1. Environment variables (`TF_VAR_`)
2. `terraform.tfvars`
3. `terraform.tfvars.json`
4. `*.auto.tfvars` (alphabetical)
5. `*.auto.tfvars.json` (alphabetical)
6. `-var` and `-var-file` flags

### Variable Precedence

```
Highest
  │
  ├── -var and -var-file flags
  ├── *.auto.tfvars files
  ├── terraform.tfvars.json
  ├── terraform.tfvars
  ├── TF_VAR_* environment variables
  ├── default value in variable block
  │
Lowest
```

### Sensitive Variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

# Mark outputs as sensitive too
output "password" {
  value     = var.db_password
  sensitive = true
}

# In resources
resource "aws_db_instance" "main" {
  password = var.db_password  # Terraform hides this in output
}
```

## Outputs

Output values are returned after `terraform apply`.

```hcl
output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
  sensitive   = false
}

# Output entire resource
output "instance" {
  description = "EC2 instance details"
  value       = aws_instance.web
  sensitive   = true
}

# Output with function
output "connection_string" {
  value = "postgresql://${aws_db_instance.main.username}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive = true
}

# Output from module
output "vpc_id" {
  value = module.vpc.vpc_id
}

# Output with condition
output "lb_url" {
  value = var.environment == "production" ? "https://app.example.com" : "http://${aws_lb.main.dns_name}"
}
```

### Accessing Outputs

```bash
# After apply
terraform output
terraform output instance_public_ip
terraform output -json

# Use in other Terraform (data source)
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Access output from remote state
value = data.terraform_remote_state.vpc.outputs.vpc_id
```

## Locals

Local values are temporary variables within a configuration.

```hcl
locals {
  # Computed values
  name_prefix = "${var.project}-${var.environment}"

  # Merged tags
  common_tags = merge(
    var.common_tags,
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  )

  # Transformations
  subnet_ids = aws_subnet.private[*].id

  # Conditional
  instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
}

# Usage
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = local.instance_type

  tags = local.common_tags
}
```

### When to Use Locals

| Use Case | Example |
|----------|---------|
| Reduce repetition | `local.name_prefix` |
| Combine data | `merge()` of tags |
| Transform data | `local.subnet_ids` |
| Simplify complex expressions | Conditional values |

### Locals vs Variables

| Feature | Variable | Local |
|---------|----------|-------|
| Input from outside | Yes | No |
| Can have default | Yes | N/A |
| Can be overridden | Yes | No |
| Scope | Module-wide | Module-wide |
| Purpose | Parameterization | Internal computation |

## Complete Example

```hcl
# variables.tf
variable "project" {
  type        = string
  description = "Project name"
}

variable "environment" {
  type        = string
  description = "Environment name"
  default     = "development"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "instance_type" {
  type        = string
  default     = "t2.micro"
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type    = map(string)
  default = {}
}

# locals.tf
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = merge(
    var.tags,
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      Terraform   = "true"
    }
  )

  instance_config = {
    development = "t2.micro"
    staging     = "t2.small"
    production  = "t2.large"
  }

  effective_instance_type = var.instance_type != "" ? var.instance_type : local.instance_config[var.environment]
}

# main.tf
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.effective_instance_type

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web"
    }
  )
}

resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidrs
  }

  tags = local.common_tags
}

# outputs.tf
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.web.id
}
```

## Interview Questions and Answers

### Q: How do you pass variables to Terraform?

1. `-var` flag: `terraform apply -var="name=value"`
2. `-var-file` flag: `terraform apply -var-file=prod.tfvars`
3. Environment variables: `export TF_VAR_name=value`
4. `terraform.tfvars` file (auto-loaded)
5. `*.auto.tfvars` files (auto-loaded)
6. Interactive prompt

### Q: What is the difference between variables and locals?

Variables are input parameters that can be passed from outside. Locals are internal computed values within a module that reduce repetition but cannot be overridden.

### Q: How do you validate variables?

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = contains(["t2.micro", "t2.large"], var.instance_type)
    error_message = "Invalid instance type."
  }
}
```

### Q: How to mark a variable as sensitive?

```hcl
variable "password" {
  type      = string
  sensitive = true
}
```

Terraform will hide the value in CLI output and plan/apply logs.

### Q: What is the loading order for terraform.tfvars files?

1. `terraform.tfvars`
2. `terraform.tfvars.json`
3. `*.auto.tfvars` (alphabetical)
4. `*.auto.tfvars.json` (alphabetical)
5. `-var-file` (highest priority)

Later values override earlier ones.

### Q: How do you access outputs from another Terraform configuration?

Use the `terraform_remote_state` data source:
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "state-bucket"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

value = data.terraform_remote_state.vpc.outputs.vpc_id
```
