# Terraform Interview Questions and Answers

## Core Concepts

### Q1: What is Terraform?

Terraform is an open-source Infrastructure as Code (IaC) tool by HashiCorp. It uses a declarative configuration language (HCL) to define, provision, and manage infrastructure across multiple cloud providers and services.

### Q2: What is Infrastructure as Code?

IaC is managing and provisioning infrastructure through code rather than manual processes. Benefits include version control, reproducibility, automated testing, documentation, and faster deployment.

### Q3: How does Terraform work?

1. User writes `.tf` configuration files
2. `terraform init` downloads providers and modules
3. `terraform plan` compares desired state with current state
4. `terraform apply` creates/updates/destroys resources to match desired state
5. State is updated to reflect current infrastructure

### Q4: What are the key components of Terraform architecture?

- **Terraform Core** - Manages state, builds dependency graph, plans execution
- **Providers** - Plugins that interact with APIs (AWS, Azure, GCP)
- **State** - Maps configuration to real resources
- **Modules** - Reusable configuration packages
- **Provisioners** - Execute scripts on resources (last resort)

### Q5: What is the difference between Terraform and Ansible?

| Feature | Terraform | Ansible |
|---------|-----------|---------|
| Type | Provisioning (IaC) | Configuration Management |
| Approach | Declarative | Imperative |
| State | Maintains state | Stateless |
| Best for | Creating infrastructure | Configuring servers |
| Language | HCL | YAML |

### Q6: What is the Terraform state file?

A JSON file that maps Terraform configuration to real-world resources. It tracks resource IDs, attributes, dependencies, and metadata. It's Terraform's source of truth for what exists.

### Q7: Why is state stored locally by default?

For simplicity and ease of getting started. For production, remote backends (S3, GCS, Terraform Cloud) are strongly recommended for team collaboration, locking, and security.

## Commands and Workflow

### Q8: Explain the Terraform workflow.

1. **Write** - Create `.tf` configuration files
2. **Init** - `terraform init` (download providers, modules, configure backend)
3. **Plan** - `terraform plan` (preview changes)
4. **Apply** - `terraform apply` (execute changes)
5. **Destroy** - `terraform destroy` (remove all resources)

### Q9: What does `terraform init` do?

1. Initializes the working directory
2. Downloads and installs provider plugins
3. Downloads child modules
4. Configures the backend for state storage
5. Checks plugin compatibility

### Q10: What is `terraform plan`?

Creates an execution plan by comparing the desired state (configuration) with the current state (state file + real infrastructure). Shows what will be created, updated, or destroyed without making changes.

### Q11: What is `terraform apply`?

Executes the plan, making actual changes to infrastructure. Creates, updates, or deletes resources to match the desired state. Updates the state file.

### Q12: How do you destroy a specific resource?

```bash
terraform destroy -target=aws_instance.web
```

### Q13: What is `terraform refresh`?

Updates the state file to match real infrastructure. It does NOT modify infrastructure. Note: `terraform plan` automatically refreshes before planning.

### Q14: What is `terraform import`?

Brings existing infrastructure under Terraform management by adding it to the state file.

```bash
terraform import aws_instance.web i-0abc123def456
```

### Q15: How do you force recreate a resource?

```bash
terraform apply -replace=aws_instance.web
```

## State Management

### Q16: Why should you never commit state files to Git?

1. **Sensitive data** - State contains resource attributes, possibly passwords and keys
2. **Concurrency issues** - Multiple team members would conflict
3. **Large files** - State files grow with infrastructure
4. **Drift** - Committed state may differ from actual infrastructure

### Q17: What is state locking?

Prevents concurrent operations that could corrupt the state file. When `terraform apply` runs, it acquires a lock. Other operations must wait. S3 uses DynamoDB, Azure uses blob leases.

### Q18: How do you migrate state between backends?

1. Update the backend configuration in the `terraform` block
2. Run `terraform init -migrate-state`
3. Terraform copies state from old to new backend

### Q19: What happens if the state file is deleted?

Terraform loses track of all managed resources. Next `terraform apply` will try to create everything again (likely causing conflicts). Recovery:
1. Restore from backend versioning
2. Re-import all resources
3. Reconcile manually

### Q20: What is the difference between `terraform state rm` and `terraform destroy`?

- `state rm` - Removes resource from state only; real resource stays
- `destroy` - Removes resource from state AND destroys the real resource

### Q21: What are Terraform workspaces?

Workspaces allow multiple separate states from the same configuration. Each workspace has its own state file.

```bash
terraform workspace new dev
terraform workspace select prod
```

### Q22: When should you NOT use workspaces?

- Different team ownership per environment
- Different access controls
- Significantly different configurations
- Production vs non-production (use separate directories)

## HCL and Configuration

### Q23: What is HCL?

HashiCorp Configuration Language - a declarative language designed for Terraform. It's human-readable and machine-parseable.

### Q24: What is the difference between `count` and `for_each`?

| Feature | count | for_each |
|---------|-------|----------|
| Input | Number | Map or set |
| Index | `count.index` (number) | `each.key`, `each.value` |
| Refactoring | Breaking when position changes | Safe (key-based) |
| Best for | Identical resources | Resources with unique attributes |

### Q25: What is `depends_on`?

An explicit dependency declaration used when Terraform can't automatically detect dependencies between resources.

```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.example]
}
```

Use sparingly - implicit dependencies through references are preferred.

### Q26: What are lifecycle rules?

```hcl
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [ami, tags]
  }
}
```

- `create_before_destroy` - Create replacement before destroying old
- `prevent_destroy` - Block destroy operations
- `ignore_changes` - Ignore changes to specific attributes

### Q27: What is the difference between `null` and an empty string?

- `null` - Absence of a value (provider uses default)
- `""` - Explicit empty string (provider uses empty value)

### Q28: How do you validate variables?

```hcl
variable "instance_type" {
  type = string
  validation {
    condition     = contains(["t2.micro", "t2.large"], var.instance_type)
    error_message = "Invalid instance type."
  }
}
```

## Modules

### Q29: What is a Terraform module?

A container for multiple resources that are used together. Every Terraform configuration is a module (the root module). Modules called from within are child modules.

### Q30: What are the benefits of modules?

1. **Code reuse** - Write once, use many times
2. **Organization** - Group related resources
3. **Abstraction** - Hide complexity
4. **Consistency** - Enforce standards
5. **Versioning** - Pin stable configurations

### Q31: How do you use a module from the Terraform Registry?

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

### Q32: How do you version modules?

Use the `version` attribute for registry modules and Git tags for source modules:
```hcl
version = "~> 5.0"    # >= 5.0.0, < 6.0.0
version = "5.1.0"     # Exactly 5.1.0
```

## Security

### Q33: How do you manage secrets in Terraform?

1. Use `sensitive = true` on variables and outputs
2. Store secrets in external managers (SSM, Vault, Secrets Manager)
3. Use environment variables in CI/CD (`TF_VAR_`)
4. Encrypt state files at rest
5. Never commit `.tfvars` with secrets
6. Use IAM policies with least privilege

### Q34: What is a taint?

A taint marks a resource for recreation on the next `terraform apply`.

```bash
# Old (deprecated)
terraform taint aws_instance.web

# New (recommended)
terraform apply -replace=aws_instance.web
```

### Q35: How do you handle drift detection?

```bash
terraform plan -detailed-exitcode
# Exit code 2 means changes detected (drift)
```

Or use automated tools like Terraform Cloud's drift detection, or custom scripts with scheduled `terraform plan`.

## Providers

### Q36: What is a Terraform provider?

A plugin that translates Terraform configuration into API calls for a specific platform (AWS, Azure, GCP, etc.).

### Q37: How do you manage multiple providers?

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
    azurerm = { source = "hashicorp/azurerm"; version = "~> 3.0" }
  }
}
```

### Q38: How do you use multiple configurations of the same provider?

```hcl
provider "aws" { region = "us-east-1"; alias = "east" }
provider "aws" { region = "us-west-2"; alias = "west" }

resource "aws_instance" "east" {
  provider = aws.east
  # ...
}
```

## Advanced

### Q39: What are provisioners? When should you use them?

Provisioners run scripts during resource creation/destruction. Types: `local-exec`, `remote-exec`, `file`.

**Use as last resort** - prefer cloud-init, Packer AMIs, or configuration management tools. Provisioners only run on creation, not on updates.

### Q40: What is a data source?

A read-only lookup of existing infrastructure or external data.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}
```

### Q41: How do you share data between Terraform configurations?

1. Remote state data source
2. Terraform Cloud outputs
3. External secret managers
4. S3 bucket with JSON files

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = { bucket = "state"; key = "vpc/terraform.tfstate"; region = "us-east-1" }
}
```

### Q42: What is the Terraform lock file?

`.terraform.lock.hcl` records exact provider versions and checksums for reproducible builds. Should be committed to version control.

### Q43: How do you handle circular dependencies?

Terraform detects and rejects circular dependencies. Fix by:
1. Restructuring resources to break the cycle
2. Using separate Terraform configurations
3. Using data sources instead of direct references

### Q44: What is `terraform graph`?

Generates a visual representation of the dependency graph in DOT format. Useful for debugging complex configurations.

```bash
terraform graph | dot -Tpng > graph.png
```

### Q45: How do you test Terraform code?

1. **Unit testing** - `terraform validate`, `terraform plan`
2. **Integration testing** - Terratest (Go library)
3. **Static analysis** - Checkov, tfsec, tflint
4. **Policy testing** - OPA/Conftest, Sentinel
5. **Cost estimation** - Infracost

### Q46: What is the difference between `terraform plan` exit codes?

- `0` - Success, no changes
- `1` - Error
- `2` - Success, changes detected

### Q47: How do you handle large state files?

1. Split into multiple configurations/modules
2. Separate state by domain (network, compute, database)
3. Use workspaces carefully
4. Remove unnecessary resources from state

### Q48: What are Terraform Cloud run tasks?

External integrations triggered during plan/apply. Can run security scans, cost estimates, custom validations, or notifications.

### Q49: What is Sentinel?

HashiCorp's policy-as-code framework for Terraform Enterprise/Cloud. Enforces rules like:
- No resources in production without tags
- All S3 buckets must be encrypted
- Only approved instance types

### Q50: How do you do zero-downtime deployments with Terraform?

```hcl
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "web" {
  min_size         = 2
  max_size         = 4
  desired_capacity = 2

  health_check_grace_period = 300  # Wait for health checks
}
```

1. Use `create_before_destroy`
2. Set appropriate health check grace periods
3. Use ALB with health checks
4. Deploy with ASG rolling updates
