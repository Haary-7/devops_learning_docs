# Terraform Modules

## What are Modules?

Modules are containers for multiple resources used together. They enable code reuse, organization, and abstraction.

**Every Terraform configuration is a module:**
- **Root module** - Your `.tf` files in the working directory
- **Child modules** - Modules called from the root module
- **Published modules** - From the Terraform Registry or private registries

## Module Structure

```
modules/
└── vpc/
    ├── main.tf          # Resources
    ├── variables.tf     # Input variables
    ├── outputs.tf       # Output values
    ├── locals.tf        # Local values
    ├── README.md        # Documentation
    └── examples/        # Usage examples
        └── main.tf
```

## Creating a Module

### VPC Module

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "environment" {
  type        = string
  description = "Environment name"
}

variable "enable_nat_gateway" {
  type        = bool
  description = "Enable NAT Gateway"
  default     = true
}

variable "public_subnet_count" {
  type        = number
  description = "Number of public subnets"
  default     = 2
}

variable "private_subnet_count" {
  type        = number
  description = "Number of private subnets"
  default     = 2
}

variable "tags" {
  type        = map(string)
  description = "Additional tags"
  default     = {}
}

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.environment}-igw"
  }
}

resource "aws_subnet" "public" {
  count = var.public_subnet_count

  vpc_id                  = aws_vpc.this.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${count.index}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count = var.private_subnet_count

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + var.public_subnet_count)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.environment}-private-${count.index}"
    Type = "private"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count = var.public_subnet_count

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# modules/vpc/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.this.id
}
```

## Using a Module

### Local Module

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = "10.0.0.0/16"
  environment          = "production"
  enable_nat_gateway   = true
  public_subnet_count  = 3
  private_subnet_count = 3

  tags = {
    Project = "my-app"
    Team    = "platform"
  }
}

# Use module outputs
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]

  tags = {
    Name = "web-server"
  }
}
```

### Registry Module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "production-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = "production"
  }
}

# EC2 Instance module
module "ec2" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.6.0"

  name           = "web-server"
  ami            = data.aws_ami.ubuntu.id
  instance_type  = "t2.micro"
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_id      = module.vpc.public_subnets[0]

  tags = {
    Environment = "production"
  }
}

# RDS module
module "db" {
  source  = "terraform-aws-modules/rds/aws"
  version = "6.5.0"

  identifier = "myapp-db"

  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.medium"
  allocated_storage    = 50
  max_allocated_storage = 100

  db_name  = "myapp"
  username = "admin"
  password = var.db_password

  vpc_security_group_ids = [module.vpc.default_security_group_id]
  db_subnet_group_name   = module.vpc.db_subnet_group_name

  tags = {
    Environment = "production"
  }
}
```

### GitHub Module

```hcl
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"

  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
}
```

### Private Registry Module

```hcl
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.2.0"

  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
}
```

## Module Versioning

### Version Constraints

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0, < 6.0.0"  # Version range
  # version = "~> 5.0"           # >= 5.0.0, < 6.0.0
  # version = "~> 5.1.0"         # >= 5.1.0, < 5.2.0
  # version = ">= 5.0.0"         # 5.0.0 or higher
  # version = "= 5.1.0"          # Exactly 5.1.0
}
```

### Version Lock File

```
.terraform.lock.hcl

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}
```

**Commit this file to version control** for reproducible builds.

## Module Patterns

### Conditional Module

```hcl
module "redis" {
  source = "cloudposse/elasticache-redis/aws"
  version = "0.52.0"

  enabled    = var.enable_cache  # Set to false to skip module
  name       = "myapp-cache"
  node_type  = "cache.t3.micro"
}
```

### Module with for_each

```hcl
module "environment" {
  for_each = var.environments

  source      = "./modules/environment"
  environment = each.key
  vpc_cidr    = each.value.vpc_cidr
  tags        = each.value.tags
}

# Access outputs
output "vpc_ids" {
  value = { for k, v in module.environment : k => v.vpc_id }
}
```

### Nested Modules

```
modules/
└── platform/
    ├── main.tf
    ├── vpc/          # Nested module
    │   └── main.tf
    └── eks/          # Nested module
        └── main.tf
```

```hcl
# modules/platform/main.tf
module "vpc" {
  source = "./vpc"
  # ...
}

module "eks" {
  source  = "./eks"
  vpc_id  = module.vpc.vpc_id
  # ...
}
```

## Module Best Practices

### 1. Keep Modules Focused

Each module should do ONE thing well:
- `vpc` - Network infrastructure
- `ec2` - Compute instances
- `rds` - Database
- `eks` - Kubernetes cluster

### 2. Version Your Modules

```hcl
# Always pin versions
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"  # Not "latest"
}
```

### 3. Document Your Modules

```markdown
# VPC Module

Creates a VPC with public and private subnets.

## Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| vpc_cidr | CIDR block | string | - | yes |
```

### 4. Use Output Validation

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.this.id
}

# Validate inputs
variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### 5. Avoid Hardcoded Values

```hcl
# Bad
resource "aws_instance" "web" {
  instance_type = "t2.micro"  # Hardcoded
}

# Good
variable "instance_type" {
  type    = string
  default = "t2.micro"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

### 6. Use Published Modules

Before writing your own, check:
- [Terraform Registry](https://registry.terraform.io)
- AWS modules by `terraform-aws-modules`
- Azure modules by `Azure`
- GCP modules by `terraform-google-modules`

## Complete Module Example

```hcl
# modules/web-app/variables.tf
variable "name" {
  type        = string
  description = "Application name"
}

variable "environment" {
  type        = string
  description = "Environment"
}

variable "vpc_id" {
  type        = string
  description = "VPC ID"
}

variable "subnet_ids" {
  type        = list(string)
  description = "Subnet IDs for the ALB"
}

variable "instance_type" {
  type        = string
  default     = "t2.micro"
}

variable "instance_count" {
  type        = number
  default     = 2
}

variable "ami_id" {
  type        = string
  description = "AMI ID for EC2 instances"
}

variable "health_check_path" {
  type    = string
  default = "/health"
}

variable "tags" {
  type    = map(string)
  default = {}
}

# modules/web-app/main.tf
locals {
  name_prefix = "${var.name}-${var.environment}"
}

# Launch Template
resource "aws_launch_template" "this" {
  name_prefix   = local.name_prefix
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.web.id]

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.tags, {
      Name        = local.name_prefix
      Environment = var.environment
    })
  }
}

# ASG
resource "aws_autoscaling_group" "this" {
  name                = local.name_prefix
  min_size            = var.instance_count
  max_size            = var.instance_count * 2
  desired_capacity    = var.instance_count
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.this.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = local.name_prefix
    propagate_at_launch = true
  }
}

# ALB
resource "aws_lb" "this" {
  name               = local.name_prefix
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.subnet_ids

  tags = var.tags
}

resource "aws_lb_target_group" "this" {
  name     = local.name_prefix
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = var.health_check_path
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
  }
}

resource "aws_lb_listener" "this" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.this.arn
  }
}

resource "aws_autoscaling_attachment" "this" {
  autoscaling_group_name = aws_autoscaling_group.this.id
  lb_target_group_arn    = aws_lb_target_group.this.arn
}

# Security Groups
resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 80
    to_port         = 80
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

resource "aws_security_group" "alb" {
  name   = "${local.name_prefix}-alb"
  vpc_id = var.vpc_id

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

# modules/web-app/outputs.tf
output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.this.dns_name
}

output "alb_arn" {
  description = "ALB ARN"
  value       = aws_lb.this.arn
}

output "asg_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.this.name
}

output "security_group_ids" {
  description = "Security group IDs"
  value = {
    web = aws_security_group.web.id
    alb = aws_security_group.alb.id
  }
}
```

## Interview Questions and Answers

### Q: What is a Terraform module?

A module is a container for multiple resources used together. Every Terraform configuration is a module. Modules enable code reuse, organization, and abstraction of infrastructure patterns.

### Q: What are the benefits of using modules?

1. **Code reuse** - Write once, use many times
2. **Organization** - Group related resources
3. **Abstraction** - Hide complexity behind simple interfaces
4. **Consistency** - Enforce standards across environments
5. **Testing** - Modules can be tested independently
6. **Versioning** - Pin module versions for stability

### Q: How do you version modules?

Use `version` attribute for registry modules, and Git tags/refs for source modules:
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
}
```

### Q: What is the difference between local and published modules?

- **Local module** - `source = "./modules/vpc"` - relative path
- **Published module** - `source = "terraform-aws-modules/vpc/aws"` - from registry
- **Git module** - `source = "git::https://..."` - from Git repository

### Q: When should you NOT use workspaces?

Workspaces are not suitable for:
- Different team ownership per environment
- Different access controls per environment
- Environments with significantly different configurations
- Production vs non-production (use separate directories)
