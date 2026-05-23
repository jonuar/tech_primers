# Terraform Cheatsheet

## Mental Model

Terraform is an **infrastructure-as-code** tool: you describe the desired state of your infrastructure in `.tf` files and Terraform figures out how to get there. It maintains a **state file** that tracks what actually exists, diffs it against your config, and creates/updates/destroys resources to close the gap. The core loop is: `write → plan → apply`. Everything is a resource, and resources can reference each other.

---

## Install & Minimal Setup

```bash
# Linux (apt)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# macOS
brew install terraform

# Verify
terraform version

# Initialize a working directory (downloads providers)
terraform init

# Preview changes without applying
terraform plan

# Apply changes
terraform apply

# Destroy all managed resources
terraform destroy
```

---

## Core Concepts

### 1. Providers
Plugins that let Terraform talk to a specific platform (AWS, GCP, Azure, etc.).

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"         # ~> means >= 5.0, < 6.0
    }
  }

  # Remote state (recommended for teams)
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"   # prevents concurrent applies
  }
}

provider "aws" {
  region = "us-east-1"
  # Credentials from ~/.aws/credentials or environment variables
  # Never hardcode access keys here
}
```

### 2. Resources
The core building block — represents a single infrastructure object.

```hcl
# resource "<provider>_<type>" "<local_name>"
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake"

  tags = {
    Environment = "production"
    Team        = "data"
  }
}

# Reference another resource's attributes
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id   # <type>.<name>.<attribute>

  versioning_configuration {
    status = "Enabled"
  }
}
```

### 3. Variables

```hcl
# variables.tf
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "allowed_ips" {
  type    = list(string)
  default = []
}

variable "db_password" {
  type      = string
  sensitive = true    # redacted from plan output and logs
}
```

```hcl
# terraform.tfvars — values for variables (don't commit sensitive ones)
region         = "us-east-1"
instance_count = 3
allowed_ips    = ["10.0.0.0/8"]
```

```bash
# Pass at runtime
terraform apply -var="region=eu-west-1"
terraform apply -var-file="prod.tfvars"

# Environment variable (TF_VAR_ prefix)
export TF_VAR_db_password="supersecret"
terraform apply
```

### 4. Outputs

```hcl
# outputs.tf
output "bucket_name" {
  value       = aws_s3_bucket.data_lake.bucket
  description = "Name of the data lake S3 bucket"
}

output "load_balancer_dns" {
  value = aws_lb.main.dns_name
}

# Sensitive output — hidden in normal apply, shown with terraform output
output "db_connection_string" {
  value     = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive = true
}
```

```bash
terraform output                     # show all outputs
terraform output load_balancer_dns   # show specific output
terraform output -json               # JSON format
```

### 5. Locals

```hcl
locals {
  env     = "production"
  project = "mlops-platform"

  # Computed values
  common_tags = {
    Environment = local.env
    Project     = local.project
    ManagedBy   = "terraform"
  }

  # String interpolation
  bucket_name = "${local.project}-${local.env}-data"
}

resource "aws_s3_bucket" "data" {
  bucket = local.bucket_name
  tags   = local.common_tags
}
```

### 6. Data Sources
Read existing infrastructure that Terraform doesn't manage.

```hcl
# Fetch existing VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Fetch latest Amazon Linux AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use in a resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_vpc.main.id  # reference data source output
}
```

### 7. Modules
Reusable, parameterized groups of resources. The building block for DRY infrastructure.

```hcl
# Call a module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # from registry
  version = "~> 5.0"

  name = "production-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}

# Reference module outputs
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnets[0]
}
```

```hcl
# Local module structure
module "lambda" {
  source = "./modules/lambda"   # relative path to local module

  function_name = "my-function"
  image_uri     = var.image_uri
}
```

### 8. Count & For Each (Multiple Resources)

```hcl
# count — simple repetition
resource "aws_iam_user" "devs" {
  count = 3
  name  = "developer-${count.index}"
}

# for_each — iterate over a map or set (preferred over count for named resources)
variable "environments" {
  default = {
    dev  = "t3.micro"
    prod = "t3.large"
  }
}

resource "aws_instance" "env" {
  for_each      = var.environments
  instance_type = each.value
  ami           = data.aws_ami.amazon_linux.id

  tags = {
    Name = "app-${each.key}"
  }
}
# Access: aws_instance.env["prod"]
```

---

## Most-Used Patterns

### Project Structure

```
infra/
├── main.tf           # resources
├── variables.tf      # input variables
├── outputs.tf        # output values
├── versions.tf       # provider + backend config
├── terraform.tfvars  # variable values (don't commit secrets)
└── modules/
    ├── lambda/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── rds/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Import Existing Resources

```bash
# Bring manually created resources under Terraform management
terraform import aws_s3_bucket.data_lake my-existing-bucket-name

# Terraform 1.5+ — import block (declarative)
import {
  to = aws_s3_bucket.data_lake
  id = "my-existing-bucket-name"
}
```

### Useful CLI Commands

```bash
terraform init              # initialize (after changes to providers/backend)
terraform fmt               # format all .tf files
terraform validate          # check syntax and config validity
terraform plan -out=plan.tfplan   # save plan to file
terraform apply plan.tfplan       # apply the saved plan exactly
terraform state list        # list all resources in state
terraform state show aws_s3_bucket.data_lake  # inspect resource state
terraform taint aws_s3_bucket.data_lake       # mark resource for recreation
terraform refresh           # sync state with real infrastructure
```

---

## Gotchas

- **State is the source of truth** — never manually edit the state file. Use `terraform state` commands.
- **Plan before every apply** — `terraform apply` without a saved plan runs plan again, which can change if something in the environment changed.
- **Remote state for teams** — local state files can't be shared safely. Use S3 + DynamoDB locking from the start.
- **`count` vs `for_each`** — if you remove an item from the middle of a `count` list, Terraform renumbers everything after it and destroys/recreates them. Use `for_each` with maps or sets of strings to avoid this.
- **`terraform destroy` is irreversible** — protect production state with `prevent_destroy = true` lifecycle rules.
- **Provider version pinning** — `version = "~> 5.0"` is safe; `version = "latest"` will break you on a major version bump.
- **Sensitive variables in state** — even `sensitive = true` values are stored in plaintext in state. Encrypt your S3 state bucket.

```hcl
# Protect against accidental deletion
resource "aws_rds_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

---

## Quick Links

- [Terraform Docs](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry](https://registry.terraform.io) — providers, modules, policies
- [AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [terraform-aws-modules](https://github.com/terraform-aws-modules) — community-maintained AWS modules
- [Terragrunt](https://terragrunt.gruntwork.io) — wrapper for DRY Terraform configs across environments
- [Infracost](https://infracost.io) — cost estimation from Terraform plans
