# Best Practices, Security, and Terraform Cloud

## Best Practices

### 1. Project Structure

```
terraform-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ec2/
│   └── rds/
└── .gitignore
```

### 2. Use .gitignore

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
.terraform.lock.hcl
```

### 3. Pin Provider and Module Versions

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
}
```

### 4. Use tfvars Files

```hcl
# terraform.tfvars (auto-loaded)
environment = "production"
region      = "us-east-1"
```

### 5. Format and Validate

```bash
terraform fmt -recursive   # Format all files
terraform fmt -check       # Check formatting (CI)
terraform validate         # Validate configuration
terraform plan             # Preview changes
```

### 6. Use Naming Conventions

```hcl
# Resources: snake_case
resource "aws_instance" "web_server" {}

# Variables: snake_case
variable "instance_type" {}

# Outputs: snake_case
output "instance_id" {}

# Use consistent naming for resources
tags = {
  Name = "${var.project}-${var.environment}-${var.component}"
}
```

## Security Best Practices

### 1. Never Hardcode Secrets

```hcl
# BAD
resource "aws_db_instance" "main" {
  password = "supersecret123"
}

# GOOD
resource "aws_db_instance" "main" {
  password = var.db_password
}
```

### 2. Use Sensitive Variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

variable "api_key" {
  type      = string
  sensitive = true
}
```

### 3. Use External Secret Management

```hcl
# AWS SSM Parameter Store
data "aws_ssm_parameter" "db_password" {
  name = "/myapp/prod/db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}

# AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "myapp/prod/db-credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

resource "aws_db_instance" "main" {
  username = local.db_creds.username
  password = local.db_creds.password
}

# HashiCorp Vault
data "vault_generic_secret" "db" {
  path = "secret/myapp/prod/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}
```

### 4. Least Privilege IAM

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:RunInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5. Enable Encryption

```hcl
# EBS volumes
resource "aws_ebs_volume" "data" {
  encrypted   = true
  kms_key_id  = aws_kms_key.main.arn
}

# RDS
resource "aws_db_instance" "main" {
  storage_encrypted  = true
  kms_key_id         = aws_kms_key.main.arn
}

# S3
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

### 6. Private Networks

```hcl
# RDS in private subnet
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet"
  subnet_ids = module.vpc.private_subnet_ids
}

resource "aws_db_instance" "main" {
  db_subnet_group_name = aws_db_subnet_group.main.name
  publicly_accessible  = false
}
```

### 7. Security Groups (Least Access)

```hcl
resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Only app SG
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 8. Static Analysis Tools

```bash
# Checkov
checkov -d .

# tfsec
tfsec .

# Terrascan
terrascan scan -i terraform

# Terraform plan security
terraform plan -out=tfplan
infracost breakdown --path tfplan
```

## Terraform Cloud / Enterprise

### Authentication

```bash
terraform login
# or
export TF_TOKEN_app_terraform_io=xxxx
```

### Cloud Configuration

```hcl
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "my-app-production"
    }
  }
}
```

### Workspace Features

| Feature | Description |
|---------|-------------|
| Remote State | Automatic state storage and locking |
| VCS Integration | Trigger runs on Git push |
| Run Tasks | Integrate with external tools |
| Sentinel Policies | Policy as Code for governance |
| Cost Estimation | Estimate costs before apply |
| Notifications | Slack, email, webhook alerts |
| Team Access | RBAC for workspaces |

### CI/CD Integration

```yaml
# GitHub Actions
name: Terraform
on:
  push:
    branches: [main]
  pull_request:

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.6.0

    - name: Terraform Init
      run: terraform init

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

## Testing

### Terratest (Go)

```go
func TestTerraformAwsInstance(t *testing.T) {
  t.Parallel()

  terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../examples/basic",
    Vars: map[string]interface{}{
      "instance_type": "t2.micro",
    },
  })

  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)

  instanceID := terraform.Output(t, terraformOptions, "instance_id")
  assert.NotEmpty(t, instanceID)
}
```

## Troubleshooting

```bash
# Debug output
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply

# Log levels: TRACE, DEBUG, INFO, WARN, ERROR

# Check provider compatibility
terraform providers

# Graph visualization
terraform graph | dot -Tpng > graph.png
```

## Interview Questions and Answers

### Q: What are Terraform best practices for production?

1. Remote state with locking (S3 + DynamoDB)
2. Pin provider and module versions
3. Use workspaces or separate directories per environment
4. Enable encryption on all resources
5. Use least privilege IAM
6. Store secrets in external secret managers
7. Use CI/CD for plan and apply
8. Format and validate in CI
9. Review plans before applying
10. Use modules for reusability

### Q: How do you handle secrets in Terraform?

1. Use variables marked as `sensitive = true`
2. Store actual values in external secret managers (SSM, Vault, Secrets Manager)
3. Never commit `.tfvars` files with secrets
4. Use environment variables (`TF_VAR_`) in CI/CD
5. Encrypt state file at rest
6. Restrict access to state files

### Q: What is the difference between `terraform fmt` and `terraform validate`?

- `fmt` - Formats HCL files to canonical style
- `validate` - Checks configuration syntax and internal consistency

### Q: How do you prevent state file exposure?

1. Use remote backend with encryption
2. Restrict IAM access to state bucket
3. Enable bucket versioning for audit
4. Never commit to Git
5. Use Terraform Cloud for managed state
