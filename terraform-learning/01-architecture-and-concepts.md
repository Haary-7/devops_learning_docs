# Terraform Architecture and Core Concepts

## What is Terraform?

Terraform is an Infrastructure as Code (IaC) tool by HashiCorp that allows you to define and provision infrastructure using a declarative configuration language (HCL).

**Key Characteristics:**
- **Declarative** - You define the desired end state, Terraform figures out how to get there
- **Provider-agnostic** - Works with AWS, Azure, GCP, Kubernetes, and 1000+ providers
- **Immutable Infrastructure** - Changes are applied by creating new resources, not modifying existing ones
- **Plan before apply** - See what will change before making changes

## Terraform vs Other IaC Tools

| Feature | Terraform | CloudFormation | Pulumi | Ansible |
|---------|-----------|----------------|--------|---------|
| Language | HCL | YAML/JSON | Python/TS/Go | YAML |
| Declarative | Yes | Yes | Yes | Imperative |
| Multi-cloud | Yes | No (AWS only) | Yes | Yes |
| State management | Yes | Yes (Stack) | Yes | No |
| Agent required | No | No | No | Yes |
| Plan/Preview | Yes | Yes (changeset) | Yes | No |

## Terraform Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Terraform CLI                                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Terraform Core                                │ │
│  │                                                            │ │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐ │ │
│  │  │  Parser    │  │  Graph     │  │   Plan & Apply       │ │ │
│  │  │  (HCL→JSON)│  │  Builder   │  │   Engine             │ │ │
│  │  └────────────┘  └────────────┘  └──────────────────────┘ │ │
│  └────────────────────────┬───────────────────────────────────┘ │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────┐ ┌──────────────────┐
│   Providers     │ │   State     │ │   Registries     │
│                 │ │             │ │                  │
│  ┌───────────┐  │ │  ┌───────┐  │ │  ┌────────────┐  │
│  │ AWS       │  │ │  │Local  │  │ │  │Public      │  │
│  │ Azure     │  │ │  │File   │  │ │  │Registry    │  │
│  │ GCP       │  │ │  │(tfstate)│ │ │  │            │  │
│  │ Kubernetes│  │ │  ├───────┤  │ │  │Private     │  │
│  │ GitHub    │  │ │  │Remote │  │ │  │Registry    │  │
│  │ Datadog   │  │ │  │(S3,   │  │ │  │(TFC,       │  │
│  │ etc.      │  │ │  │ Consul│  │ │  │ Artifactory)│ │
│  └───────────┘  │ │  │ etc.) │  │ │  └────────────┘  │
└─────────────────┘ └─────────────┘ └──────────────────┘
```

## Terraform Workflow

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Init   │───▶│  Plan   │───▶│  Apply  │───▶│ Destroy │
│         │    │         │    │         │    │         │
│ Download│    │ Preview │    │ Create/ │    │ Remove  │
│ providers│   │ changes │    │ Update/ │    │ all     │
│ & modules│   │ before  │    │ Delete  │    │ resources│
│         │    │ applying│    │ resources│   │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
```

### 1. `terraform init`

- Initializes the working directory
- Downloads provider plugins
- Downloads modules
- Configures the backend
- **Idempotent** - Safe to run multiple times

```bash
terraform init
terraform init -upgrade    # Upgrade providers
terraform init -backend=false  # Skip backend init
terraform init -migrate-state  # Migrate state to new backend
```

### 2. `terraform plan`

- Creates an execution plan
- Compares current state with desired configuration
- Shows what will be created, updated, or destroyed
- **No changes** are made to infrastructure

```bash
terraform plan
terraform plan -out=tfplan          # Save plan to file
terraform plan -target=aws_instance.web  # Plan specific resource
terraform plan -var-file=prod.tfvars
terraform plan -refresh=false       # Skip state refresh
```

### 3. `terraform apply`

- Executes the plan
- Creates, updates, or deletes resources
- Updates the state file

```bash
terraform apply
terraform apply tfplan              # Apply saved plan
terraform apply -auto-approve       # Skip confirmation
terraform apply -var-file=prod.tfvars
terraform apply -target=aws_instance.web
```

### 4. `terraform destroy`

- Destroys all resources managed by the configuration
- Removes resources from state

```bash
terraform destroy
terraform destroy -auto-approve
terraform destroy -target=aws_instance.web  # Destroy specific resource
```

## Terraform State

### What is State?

The state file (`terraform.tfstate`) is Terraform's source of truth. It contains:
- Resource IDs and attributes
- Dependencies between resources
- Metadata (provider version, module info)
- Outputs

### State File Example

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 3,
  "lineage": "abc123",
  "outputs": {
    "instance_ip": {
      "value": "54.210.100.50",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro",
            "tags": {
              "Name": "web-server"
            }
          }
        }
      ]
    }
  ]
}
```

### Why State Matters

1. **Resource tracking** - Maps config to real-world resources
2. **Metadata** - Stores dependencies for correct ordering
3. **Performance** - Caches attribute values to avoid API calls
4. **Plan accuracy** - Compares desired vs actual state

### State Backends

| Backend | Description | Best For |
|---------|-------------|----------|
| `local` | File on disk (default) | Development, single user |
| `s3` | AWS S3 with DynamoDB locking | AWS teams |
| `azurerm` | Azure Blob Storage | Azure teams |
| `gcs` | Google Cloud Storage | GCP teams |
| `consul` | HashiCorp Consul | On-premises |
| `remote` | Terraform Cloud/Enterprise | Enterprise teams |
| `http` | HTTP server with locks | Custom setups |

### Remote State (S3 Example)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    acl            = "private"
  }
}
```

### State Locking

Prevents concurrent operations that could corrupt state:

| Backend | Locking Mechanism |
|---------|-------------------|
| S3 | DynamoDB table |
| Azure | Blob lease |
| GCS | Object generation |
| Consul | KV store |
| Terraform Cloud | Built-in |

```bash
# Force unlock (dangerous!)
terraform force-unlock <LOCK_ID>
```

### State Commands

```bash
# List resources in state
terraform state list

# Show a resource
terraform state show aws_instance.web

# Move a resource
terraform state mv aws_instance.old aws_instance.new

# Remove from state (doesn't destroy)
terraform state rm aws_instance.web

# Pull state
terraform state pull

# Push state
terraform state push

# Import existing resource
terraform import aws_instance.web i-0abc123def456

# Show state
terraform show
terraform show -json
```

### State File Best Practices

1. **Never commit to version control** - Contains sensitive data
2. **Use remote backend** - Team collaboration, locking, encryption
3. **Enable versioning** - S3 bucket versioning for rollback
4. **Restrict access** - IAM policies for state bucket
5. **Encrypt at rest** - S3 SSE, GCS encryption
6. **Use workspaces** - Separate state per environment

## Key Terminology

| Term | Definition |
|------|------------|
| **Provider** | Plugin that interacts with an API (AWS, Azure, etc.) |
| **Resource** | A component of your infrastructure (EC2, S3, etc.) |
| **Data Source** | Read-only lookup of existing resources |
| **Module** | Reusable configuration package |
| **Variable** | Parameter for your configuration |
| **Output** | Value returned after apply |
| **Local** | Temporary value within configuration |
| **Backend** | Where state is stored and operations are run |
| **Workspace** | Named state within the same configuration |
| **Provisioner** | Run scripts on resources during creation/destruction |
| **Tainted** | Resource marked for recreation |

## Interview Questions and Answers

### Q: What is Infrastructure as Code (IaC)?

IaC is managing and provisioning infrastructure through machine-readable definition files rather than physical hardware configuration or interactive configuration tools. Benefits include version control, reproducibility, documentation, and automation.

### Q: Why does Terraform use state?

State maps your configuration to real-world resources, tracks metadata and dependencies, improves performance by caching attributes, and enables Terraform to determine what changes need to be made during plan/apply.

### Q: What happens when you run `terraform init`?

1. Initializes the working directory
2. Downloads and installs provider plugins
3. Downloads child modules
4. Configures the backend
5. Checks plugin compatibility

### Q: What is the difference between `terraform plan` and `terraform apply`?

- **plan** - Shows what changes will be made without making them
- **apply** - Executes the plan and makes actual changes to infrastructure

### Q: How does Terraform handle dependencies between resources?

Terraform automatically detects implicit dependencies through interpolation:
```hcl
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Depends on aws_subnet.main
}
```
Explicit dependencies use `depends_on`.

### Q: What happens if two people run `terraform apply` at the same time?

State locking prevents this. The first operation acquires a lock; the second waits or fails if the lock is held. If a lock is stale, `terraform force-unlock` can release it (use with caution).
