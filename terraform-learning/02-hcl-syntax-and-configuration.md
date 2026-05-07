# HCL Syntax and Configuration

## HCL (HashiCorp Configuration Language)

HCL is the language used for Terraform configurations. It's designed to be both human-readable and machine-friendly.

## File Structure

A typical Terraform project:

```
project/
├── main.tf              # Providers, resources
├── variables.tf         # Variable declarations
├── outputs.tf           # Output definitions
├── locals.tf            # Local values
├── terraform.tfvars     # Variable values
└── modules/             # Local modules
    ├── vpc/
    └── ec2/
```

## Configuration Blocks

### terraform Block

Settings for Terraform itself.

```hcl
terraform {
  # Required provider versions
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }

  # Backend configuration
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }

  # Experimental features
  experiments = [variable_validation]
}
```

### provider Block

Configure how Terraform connects to APIs.

```hcl
# Basic provider
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy = "terraform"
      Project   = "my-project"
    }
  }
}

# Multiple configurations of same provider
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
}

# Use aliased provider
resource "aws_instance" "east" {
  provider = aws.east
  ami      = "ami-0c55b159cbfafe1f0"
}

resource "aws_instance" "west" {
  provider = aws.west
  ami      = "ami-0c55b159cbfafe1f0"
}

# Provider with authentication
provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key

  # Assume role
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
  }

  # Skip certain validations (useful for localstack, minio)
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
}
```

## Resource Blocks

The most fundamental block - defines infrastructure.

```hcl
resource "aws_instance" "web" {
  # Arguments
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # References
  subnet_id = aws_subnet.main.id

  # Meta-arguments
  count = 3

  # Tags
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

**Syntax:** `resource "<type>" "<name>" { }`
- **Type:** Provider resource type (e.g., `aws_instance`)
- **Name:** Your local reference name (e.g., `web`)
- **Reference:** `aws_instance.web` or `aws_instance.web.id`

## Resource Meta-Arguments

### count

Create multiple instances of a resource.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index}"  # 0, 1, 2
  }
}

# Reference
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]

# count.index - Current index
# count - Total count

# Conditional creation
resource "aws_instance" "optional" {
  count = var.create_instance ? 1 : 0
  ami   = "ami-0c55b159cbfafe1f0"
}
```

### for_each

Create resources from a map or set.

```hcl
# From a map
resource "aws_instance" "servers" {
  for_each = {
    web  = { ami = "ami-0c55b159cbfafe1f0", type = "t2.micro" }
    api  = { ami = "ami-0c55b159cbfafe1f0", type = "t2.small" }
    db   = { ami = "ami-0c55b159cbfafe1f0", type = "t2.large" }
  }

  ami           = each.value.ami
  instance_type = each.value.type

  tags = {
    Name = "${each.key}-server"
  }
}

# From a set
resource "aws_security_group_rule" "ingress" {
  for_each = toset([80, 443, 8080])

  type              = "ingress"
  from_port         = each.value
  to_port           = each.value
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_sg.main.id
}

# From a list (convert to set)
resource "aws_instance" "workers" {
  for_each = toset(["worker-1", "worker-2", "worker-3"])

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = each.value
  }
}
```

### count vs for_each

| Feature | count | for_each |
|---------|-------|----------|
| Input type | Number | Map or set |
| Index | `count.index` (number) | `each.key`, `each.value` |
| Refactoring | Breaking when index changes | Stable when items change |
| Best for | Identical resources | Resources with unique attributes |
| Removal | Removes by index | Removes by key |

### depends_on

Explicit dependency (use sparingly).

```hcl
# Implicit dependency (preferred)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Automatically depends on subnet
}

# Explicit dependency (when implicit isn't possible)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  depends_on = [
    aws_iam_role_policy.example,
    aws_s3_bucket.logs,
  ]
}
```

### lifecycle

Control resource lifecycle behavior.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    # Never destroy - create new before destroying old
    create_before_destroy = true

    # Ignore changes to these attributes
    ignore_changes = [
      ami,
      tags["Name"],
    ]

    # Replace when these attributes change
    replace_triggered_by = [
      aws_launch_template.web.id,
    ]

    # Prevent accidental destruction
    prevent_destroy = true
  }
}
```

### provisioner

Run scripts during resource creation/destruction (last resort).

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # Run after creation
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx",
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # Run before creation
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> inventory.txt"
  }

  # Run before destruction
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Destroying instance ${self.id}'"
  }
}
```

**Provisioner types:**
| Type | Runs On | Use Case |
|------|---------|----------|
| `local-exec` | Machine running Terraform | Local scripts, notifications |
| `remote-exec` | Remote resource | Software installation, configuration |
| `file` | Remote resource | Upload files to resource |

### provisioner Best Practices

- **Avoid provisioners** when possible - use cloud-init, AMIs, or configuration management
- Provisioners only run on creation, not on update
- Failed provisioners taint the resource
- Use `local-exec` for side effects, not infrastructure changes

## Data Types

### Primitive Types

| Type | Example | Description |
|------|---------|-------------|
| `string` | `"hello"` | Text |
| `number` | `42`, `3.14` | Numeric value |
| `bool` | `true`, `false` | Boolean |

### Complex Types

| Type | Example | Description |
|------|---------|-------------|
| `list` / `tuple` | `["a", "b", "c"]` | Ordered collection |
| `map` / `object` | `{key = "value"}` | Key-value collection |
| `set` | `toset(["a", "b"])` | Unordered unique collection |
| `null` | `null` | Absence of value |

```hcl
# List (same type)
variable "subnets" {
  type    = list(string)
  default = ["subnet-1", "subnet-2", "subnet-3"]
}

# Map (same type values)
variable "tags" {
  type    = map(string)
  default = {
    Environment = "production"
    Team        = "platform"
  }
}

# Tuple (mixed types)
variable "config" {
  type = tuple([string, number, bool])
  default = ["enabled", 5, true]
}

# Object (mixed types with named keys)
variable "instance_config" {
  type = object({
    name    = string
    size    = number
    enabled = bool
    tags    = map(string)
  })
}
```

## Expressions and References

### References

```hcl
# Resource attributes
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id
  vpc_id    = aws_subnet.main.vpc_id
}

# Output references
value = aws_instance.web.public_ip

# Data source references
value = data.aws_ami.ubuntu.id

# Variable references
value = var.instance_type

# Local value references
value = local.common_tags
```

### Splat Expressions

```hcl
# Legacy splat (lists)
value = aws_instance.web[*].id

# Modern splat (works with all collections)
value = aws_instance.web[*].public_ip
value = aws_instance.web.*.public_ip  # Legacy syntax

# For maps (for_each)
value = { for k, v in aws_instance.servers : k => v.id }
```

### For Expressions

```hcl
# Transform list
locals {
  upper_names = [for name in var.names : upper(name)]
}

# Transform map
locals {
  port_map = { for name, port in var.ports : name => port }
}

# Filter
locals {
  public_subnets = [
    for s in var.subnets : s
    if s.public == true
  ]
}

# Map with condition
locals {
  active_instances = {
    for k, v in aws_instance.servers : k => v.id
    if v.tags.Environment == "production"
  }
}
```

### Conditional Expressions

```hcl
# Ternary
instance_type = var.environment == "production" ? "t2.large" : "t2.micro"

# Null coalescing
name = var.name != null ? var.name : "default-name"

# Multiple conditions
environment = var.env == "prod" ? "production" : var.env == "stg" ? "staging" : "development"
```

### Functions

#### String Functions

```hcl
upper("hello")         # "HELLO"
lower("HELLO")         # "hello"
title("hello world")   # "Hello World"
join("-", ["a", "b"])  # "a-b"
split("-", "a-b-c")    # ["a", "b", "c"]
replace("hello", "ll", "rr")  # "herro"
substr("hello", 1, 3)  # "ell"
trim("  hello  ")      # "hello"
format("name: %s", "app")  # "name: app"
format("%02d", 5)      # "05"
```

#### Collection Functions

```hcl
length(["a", "b"])     # 2
concat([1, 2], [3])    # [1, 2, 3]
distinct([1, 1, 2])    # [1, 2]
flatten([[1], [2, 3]]) # [1, 2, 3]
merge({a = 1}, {b = 2}) # {a = 1, b = 2}
lookup({a = 1}, "a", "default")  # 1
keys({a = 1, b = 2})   # ["a", "b"]
values({a = 1, b = 2}) # [1, 2]
contains([1, 2], 1)    # true
toset(["a", "a", "b"]) # set("a", "b")
element(["a", "b"], 0) # "a"
```

#### Numeric Functions

```hcl
max(1, 2, 3)     # 3
min(1, 2, 3)     # 1
abs(-5)          # 5
floor(3.7)       # 3
ceil(3.2)        # 4
log(100, 10)     # 2
pow(2, 3)        # 8
```

#### File/System Functions

```hcl
file("config.json")              # Read file contents
filebase64("image.png")          # Read file as base64
jsondecode(file("config.json"))  # Parse JSON
yamldecode(file("config.yaml"))  # Parse YAML
jsonencode({a = 1})              # Encode to JSON
yamlencode({a = 1})              # Encode to YAML
templatefile("template.tpl", {name = "app"})  # Render template
path.module                      # Path to current module
path.root                        # Path to root module
path.cwd                         # Path to working directory
```

#### Hash/Crypto Functions

```hcl
base64encode("hello")     # "aGVsbG8="
base64decode("aGVsbG8=")  # "hello"
md5("hello")              # "5d41402abc4b2a76b9719d911017c592"
sha256("hello")           # hash
uuid()                    # Random UUID
bcrypt("password")        # Bcrypt hash
```

#### Type Conversion

```hcl
tostring(42)         # "42"
tonumber("42")       # 42
tobool("true")       # true
tolist(["a", "b"])   # list
toset(["a", "b"])    # set
tomap({a = 1})       # map
```

## Comments

```hcl
# Single line comment

/*
Multi-line comment
*/
```
