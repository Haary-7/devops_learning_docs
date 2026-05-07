# State Management

## State Fundamentals

### What State Contains

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 5,
  "lineage": "abc-123-def",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [{
        "schema_version": 1,
        "attributes": {
          "id": "i-0abc123",
          "ami": "ami-0c55b159",
          "instance_type": "t2.micro"
        },
        "sensitive_attributes": [],
        "private": "eyJzY2hl..."
      }]
    }
  ],
  "check_results": null
}
```

### State Commands Reference

```bash
# View state
terraform show
terraform show -json

# List resources
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Pull/push
terraform state pull > state.json
terraform state push state.json

# Move resource
terraform state mv aws_instance.old aws_instance.new
terraform state mv aws_instance.web 'module.ec2.aws_instance.web'

# Remove from state (doesn't destroy)
terraform state rm aws_instance.web

# Replace (mark for recreation)
terraform apply -replace=aws_instance.web

# Import
terraform import aws_instance.web i-0abc123

# Force unlock
terraform force-unlock <LOCK_ID>
```

## State Backends

### Local Backend (Default)

```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

### S3 Backend (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    acl            = "private"

    # Prevent accidental overwrite
    workspace_key_prefix = "env"
  }
}

# DynamoDB table for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Lock"
  }
}
```

### Azure Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "terraformstate123"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### GCS Backend

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
  }
}
```

### Terraform Cloud Backend

```hcl
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "my-app-prod"
    }
  }
}
```

## State Migration

### Migrating Backend

```bash
# 1. Pull current state
terraform state pull > current.tfstate

# 2. Update backend configuration in terraform block

# 3. Initialize with migration
terraform init -migrate-state

# 4. Verify
terraform state list
```

### Partial Configuration

```hcl
terraform {
  backend "s3" {
    # Bucket and key omitted - passed via CLI
  }
}

# Initialize
terraform init -backend-config="bucket=my-bucket" \
               -backend-config="key=prod/state.tfstate" \
               -backend-config="region=us-east-1"
```

## Workspaces

Workspaces allow multiple states from the same configuration.

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Select workspace
terraform workspace select prod

# Show current
terraform workspace show

# Delete workspace
terraform workspace delete dev

# State per workspace
terraform.workspace  # Reference in config
```

### Workspace-Aware Configuration

```hcl
# locals.tf
locals {
  environment = terraform.workspace

  instance_config = {
    dev  = { type = "t2.micro", count = 1 }
    stg  = { type = "t2.small", count = 2 }
    prod = { type = "t2.large", count = 3 }
  }

  config = local.instance_config[terraform.workspace]
}

resource "aws_instance" "web" {
  count         = local.config.count
  instance_type = local.config.type
  ami           = data.aws_ami.ubuntu.id

  tags = {
    Name        = "web-${terraform.workspace}-${count.index}"
    Environment = terraform.workspace
  }
}
```

### When to Use Workspaces

| Use Workspaces | Don't Use Workspaces |
|---------------|---------------------|
| Same config, different environments | Different configurations |
| Quick environment testing | Environments with different access |
| Feature branches | Environments with different teams |
| | Production vs non-production (use separate configs) |

**Best Practice:** For production environments, use separate configurations/directories instead of workspaces for better isolation and access control.

## State File Security

### Security Checklist

1. **Never commit state to Git** - Add to `.gitignore`
2. **Encrypt at rest** - S3 SSE, GCS encryption
3. **Enable versioning** - For rollback capability
4. **Restrict access** - IAM policies, bucket policies
5. **Enable access logging** - S3 server access logging
6. **Use remote backend** - Prevents local state exposure
7. **Rotate credentials** - State may contain sensitive data

### .gitignore

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
.terraform.lock.hcl
```

### S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::terraform-state-bucket",
        "arn:aws:s3:::terraform-state-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

## State Operations

### taint (deprecated)

Mark a resource for recreation on next apply.

```bash
# Old method (deprecated)
terraform taint aws_instance.web
terraform untaint aws_instance.web

# New method (recommended)
terraform apply -replace=aws_instance.web
```

### Refresh

Update state to match real infrastructure.

```bash
# Refresh state only
terraform refresh

# Plan with refresh (default)
terraform plan

# Plan without refresh
terraform plan -refresh=false
```

### Drift Detection

```bash
# Check for drift
terraform plan -detailed-exitcode

# Exit codes:
# 0 - No changes
# 1 - Error
# 2 - Changes detected

# Automated drift detection
terraform plan -out=tfplan -input=false
if [ $? -eq 2 ]; then
  echo "Drift detected!"
fi
```

### State Backup

```bash
# Automatic backup (local backend)
# Creates terraform.tfstate.backup before each write

# Manual backup
terraform state pull > backup-$(date +%Y%m%d).tfstate

# S3 versioning provides automatic backups
```

## State Best Practices

### 1. Use Remote State

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "${var.environment}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### 2. Separate State per Environment

```
environments/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    ├── variables.tf
    └── terraform.tfvars
```

### 3. Use Remote State Data Sources

```hcl
# Network team manages VPC
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Application team uses VPC outputs
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_id
}
```

### 4. State File Size Management

- State files grow with infrastructure size
- For large state files, consider splitting into multiple configurations
- Use `terraform state rm` to remove resources no longer managed
- Consider state file splitting by domain (network, compute, database)

## Interview Questions and Answers

### Q: Why shouldn't you commit state files to version control?

1. **Sensitive data** - State contains resource attributes, possibly passwords
2. **Concurrency** - Multiple team members would conflict
3. **Size** - State files can grow large
4. **Drift** - Real state may differ from committed state

### Q: How does Terraform state locking work?

When running `terraform apply` or `terraform plan`:
1. Terraform attempts to acquire a lock in the backend
2. If lock acquired, operation proceeds
3. If lock held, operation waits or fails
4. After operation completes (success or failure), lock is released

S3 uses DynamoDB, Azure uses blob leases, GCS uses object generation numbers.

### Q: How do you handle state file corruption?

1. Restore from backend versioning (S3 versions, GCS object versions)
2. Use `terraform state push` to restore from backup
3. If no backup available, re-import resources

### Q: What is the difference between `terraform state rm` and `terraform destroy`?

- `state rm` - Removes resource from state only; real resource remains
- `destroy` - Removes resource from state AND destroys the real resource

### Q: How to share state between teams?

Use remote state data sources:
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = { bucket = "state"; key = "vpc/state.tfstate"; region = "us-east-1" }
}
value = data.terraform_remote_state.vpc.outputs.vpc_id
```

Or use Terraform Cloud/Enterprise for shared state access.
