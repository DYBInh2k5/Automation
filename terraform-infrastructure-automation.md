# Terraform - Infrastructure as Code Automation

## Tổng quan về Terraform

Terraform là công cụ Infrastructure as Code (IaC) mã nguồn mở của HashiCorp, cho phép định nghĩa và quản lý infrastructure thông qua configuration files. Terraform hỗ trợ nhiều cloud providers và services thông qua provider ecosystem.

## Cài đặt và Cấu hình

### Installation
```bash
# Download và install (Linux)
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# macOS với Homebrew
brew install terraform

# Windows với Chocolatey
choco install terraform

# Verify installation
terraform version
```

### Provider Configuration
```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }
  
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```

## Basic Infrastructure Components

### VPC và Networking
```hcl
# vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Type = "private"
  }
}

resource "aws_eip" "nat" {
  count = length(var.availability_zones)
  
  domain = "vpc"
  
  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = {
    Name = "${var.project_name}-nat-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count = length(var.availability_zones)
  
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "${var.project_name}-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### Security Groups
```hcl
# security_groups.tf
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for web servers"
  
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description     = "SSH"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-web-sg"
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "database" {
  name_prefix = "${var.project_name}-db-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for database servers"
  
  ingress {
    description     = "MySQL/Aurora"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-db-sg"
  }
}

resource "aws_security_group" "bastion" {
  name_prefix = "${var.project_name}-bastion-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for bastion host"
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-bastion-sg"
  }
}
```

## Modules Development

### Module Structure
```
modules/
└── eks-cluster/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    └── README.md
```

### EKS Module Implementation
```hcl
# modules/eks-cluster/main.tf
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn
  version  = var.kubernetes_version
  
  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = var.endpoint_private_access
    endpoint_public_access  = var.endpoint_public_access
    public_access_cidrs     = var.public_access_cidrs
    security_group_ids      = [aws_security_group.cluster.id]
  }
  
  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }
  
  enabled_cluster_log_types = var.cluster_log_types
  
  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
    aws_iam_role_policy_attachment.cluster_AmazonEKSVPCResourceController,
    aws_cloudwatch_log_group.cluster,
  ]
  
  tags = var.tags
}

resource "aws_eks_node_group" "main" {
  for_each = var.node_groups
  
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = each.key
  node_role_arn   = aws_iam_role.node_group.arn
  subnet_ids      = var.private_subnet_ids
  
  instance_types = each.value.instance_types
  capacity_type  = each.value.capacity_type
  disk_size      = each.value.disk_size
  
  scaling_config {
    desired_size = each.value.desired_size
    max_size     = each.value.max_size
    min_size     = each.value.min_size
  }
  
  update_config {
    max_unavailable_percentage = each.value.max_unavailable_percentage
  }
  
  remote_access {
    ec2_ssh_key               = var.ssh_key_name
    source_security_group_ids = [aws_security_group.node_group_ssh.id]
  }
  
  labels = merge(
    {
      "node-group" = each.key
    },
    each.value.labels
  )
  
  dynamic "taint" {
    for_each = each.value.taints
    content {
      key    = taint.value.key
      value  = taint.value.value
      effect = taint.value.effect
    }
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.node_group_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.node_group_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.node_group_AmazonEC2ContainerRegistryReadOnly,
  ]
  
  tags = var.tags
}

# IAM Roles
resource "aws_iam_role" "cluster" {
  name = "${var.cluster_name}-cluster-role"
  
  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
  
  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cluster.name
}

resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSVPCResourceController" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSVPCResourceController"
  role       = aws_iam_role.cluster.name
}

# KMS Key for encryption
resource "aws_kms_key" "eks" {
  description             = "EKS Secret Encryption Key"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  
  tags = var.tags
}

resource "aws_kms_alias" "eks" {
  name          = "alias/${var.cluster_name}-eks"
  target_key_id = aws_kms_key.eks.key_id
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "cluster" {
  name              = "/aws/eks/${var.cluster_name}/cluster"
  retention_in_days = var.log_retention_days
  
  tags = var.tags
}

# modules/eks-cluster/variables.tf
variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "subnet_ids" {
  description = "List of subnet IDs for the EKS cluster"
  type        = list(string)
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs for node groups"
  type        = list(string)
}

variable "endpoint_private_access" {
  description = "Enable private API server endpoint"
  type        = bool
  default     = true
}

variable "endpoint_public_access" {
  description = "Enable public API server endpoint"
  type        = bool
  default     = true
}

variable "public_access_cidrs" {
  description = "List of CIDR blocks for public access"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "cluster_log_types" {
  description = "List of control plane logging types to enable"
  type        = list(string)
  default     = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}

variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 7
}

variable "node_groups" {
  description = "Map of EKS node group configurations"
  type = map(object({
    instance_types               = list(string)
    capacity_type               = string
    disk_size                   = number
    desired_size                = number
    max_size                    = number
    min_size                    = number
    max_unavailable_percentage  = number
    labels                      = map(string)
    taints = list(object({
      key    = string
      value  = string
      effect = string
    }))
  }))
  default = {}
}

variable "ssh_key_name" {
  description = "EC2 Key Pair name for SSH access"
  type        = string
  default     = null
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

# modules/eks-cluster/outputs.tf
output "cluster_id" {
  description = "EKS cluster ID"
  value       = aws_eks_cluster.main.id
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.main.arn
}

output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.main.name
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_certificate_authority_data" {
  description = "EKS cluster certificate authority data"
  value       = aws_eks_cluster.main.certificate_authority[0].data
}

output "cluster_security_group_id" {
  description = "Security group ID attached to the EKS cluster"
  value       = aws_security_group.cluster.id
}

output "node_groups" {
  description = "EKS node groups"
  value       = aws_eks_node_group.main
}
```

### Using Modules
```hcl
# environments/production/main.tf
module "vpc" {
  source = "../../modules/vpc"
  
  project_name       = var.project_name
  environment        = var.environment
  vpc_cidr          = var.vpc_cidr
  availability_zones = var.availability_zones
  
  tags = local.common_tags
}

module "eks" {
  source = "../../modules/eks-cluster"
  
  cluster_name           = "${var.project_name}-${var.environment}"
  kubernetes_version     = var.kubernetes_version
  subnet_ids            = module.vpc.public_subnet_ids
  private_subnet_ids    = module.vpc.private_subnet_ids
  
  node_groups = {
    general = {
      instance_types              = ["t3.medium"]
      capacity_type              = "ON_DEMAND"
      disk_size                  = 50
      desired_size               = 2
      max_size                   = 10
      min_size                   = 1
      max_unavailable_percentage = 25
      labels = {
        role = "general"
      }
      taints = []
    }
    
    spot = {
      instance_types              = ["t3.large", "t3a.large", "t2.large"]
      capacity_type              = "SPOT"
      disk_size                  = 50
      desired_size               = 3
      max_size                   = 20
      min_size                   = 0
      max_unavailable_percentage = 50
      labels = {
        role = "spot"
      }
      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }
  
  tags = local.common_tags
}

module "rds" {
  source = "../../modules/rds"
  
  identifier     = "${var.project_name}-${var.environment}"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true
  
  db_name  = var.database_name
  username = var.database_username
  password = var.database_password
  
  vpc_security_group_ids = [module.vpc.database_security_group_id]
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  tags = local.common_tags
}
```

## Advanced Terraform Patterns

### Workspaces và Environment Management
```bash
# Create workspaces
terraform workspace new development
terraform workspace new staging
terraform workspace new production

# List workspaces
terraform workspace list

# Switch workspace
terraform workspace select production

# Show current workspace
terraform workspace show
```

```hcl
# Environment-specific configuration
locals {
  environment = terraform.workspace
  
  # Environment-specific settings
  env_config = {
    development = {
      instance_count = 1
      instance_type  = "t3.micro"
      min_size      = 1
      max_size      = 2
    }
    staging = {
      instance_count = 2
      instance_type  = "t3.small"
      min_size      = 1
      max_size      = 4
    }
    production = {
      instance_count = 3
      instance_type  = "t3.medium"
      min_size      = 2
      max_size      = 10
    }
  }
  
  current_env = local.env_config[local.environment]
  
  common_tags = {
    Environment = local.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}

resource "aws_instance" "web" {
  count = local.current_env.instance_count
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.current_env.instance_type
  
  tags = merge(local.common_tags, {
    Name = "${var.project_name}-web-${count.index + 1}"
  })
}
```

### Data Sources và External Data
```hcl
# Data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

# External data source
data "external" "git_info" {
  program = ["bash", "-c", <<EOF
echo '{
  "commit": "'$(git rev-parse HEAD)'",
  "branch": "'$(git rev-parse --abbrev-ref HEAD)'",
  "tag": "'$(git describe --tags --exact-match 2>/dev/null || echo "none")'"
}'
EOF
  ]
}

# Using data in resources
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  
  tags = {
    Name       = "web-server"
    GitCommit  = data.external.git_info.result.commit
    GitBranch  = data.external.git_info.result.branch
    GitTag     = data.external.git_info.result.tag
    AccountId  = data.aws_caller_identity.current.account_id
    Region     = data.aws_region.current.name
  }
}
```

### Dynamic Blocks và Conditional Resources
```hcl
# Dynamic blocks
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = var.vpc_id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
  
  dynamic "egress" {
    for_each = var.egress_rules
    content {
      description = egress.value.description
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }
}

# Conditional resources
resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = var.public_subnet_id
  
  tags = {
    Name = "bastion-host"
  }
}

# Conditional configuration
resource "aws_db_instance" "main" {
  identifier = var.db_identifier
  
  engine         = var.db_engine
  engine_version = var.db_engine_version
  instance_class = var.db_instance_class
  
  allocated_storage = var.db_allocated_storage
  storage_type      = var.db_storage_type
  storage_encrypted = var.environment == "production" ? true : false
  
  # Multi-AZ only for production
  multi_az = var.environment == "production" ? true : false
  
  # Backup retention based on environment
  backup_retention_period = var.environment == "production" ? 30 : 7
  
  # Performance Insights for production
  performance_insights_enabled = var.environment == "production" ? true : false
  
  tags = {
    Name        = var.db_identifier
    Environment = var.environment
  }
}
```

## State Management

### Remote State Configuration
```hcl
# Backend configuration
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    
    # Workspace-specific state files
    workspace_key_prefix = "workspaces"
  }
}

# State locking table
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-state-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name = "Terraform State Locks"
  }
}
```

### State Import và Migration
```bash
# Import existing resources
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_vpc.main vpc-12345678

# State manipulation
terraform state list
terraform state show aws_instance.web
terraform state mv aws_instance.web aws_instance.web_server
terraform state rm aws_instance.old

# State pull/push
terraform state pull > terraform.tfstate.backup
terraform state push terraform.tfstate.backup
```

## Testing và Validation

### Terratest Integration
```go
// test/terraform_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestTerraformVPC(t *testing.T) {
    t.Parallel()
    
    // Construct the terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/vpc",
        
        Vars: map[string]interface{}{
            "project_name": "test-project",
            "environment":  "test",
            "vpc_cidr":     "10.0.0.0/16",
        },
        
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "us-west-2",
        },
    })
    
    // Clean up resources with "terraform destroy" at the end of the test
    defer terraform.Destroy(t, terraformOptions)
    
    // Run "terraform init" and "terraform apply"
    terraform.InitAndApply(t, terraformOptions)
    
    // Get the outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    publicSubnetIds := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
    
    // Verify the VPC exists
    vpc := aws.GetVpcById(t, vpcId, "us-west-2")
    assert.Equal(t, "10.0.0.0/16", vpc.CidrBlock)
    
    // Verify subnets exist
    assert.Equal(t, 3, len(publicSubnetIds))
    assert.Equal(t, 3, len(privateSubnetIds))
    
    // Test connectivity
    for _, subnetId := range publicSubnetIds {
        subnet := aws.GetSubnetById(t, subnetId, "us-west-2")
        assert.True(t, subnet.MapPublicIpOnLaunch)
    }
}

func TestTerraformEKS(t *testing.T) {
    t.Parallel()
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/eks",
        
        Vars: map[string]interface{}{
            "cluster_name": "test-cluster",
            "node_groups": map[string]interface{}{
                "general": map[string]interface{}{
                    "instance_types": []string{"t3.medium"},
                    "desired_size":   1,
                    "max_size":      2,
                    "min_size":      1,
                },
            },
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // Test EKS cluster
    clusterName := terraform.Output(t, terraformOptions, "cluster_name")
    assert.NotEmpty(t, clusterName)
    
    // Verify cluster is active
    cluster := aws.GetEksCluster(t, "us-west-2", clusterName)
    assert.Equal(t, "ACTIVE", cluster.Status)
}
```

### Validation và Linting
```bash
# Terraform validation
terraform validate

# Format code
terraform fmt -recursive

# Security scanning with tfsec
tfsec .

# Policy validation with OPA
opa test policies/

# Cost estimation with Infracost
infracost breakdown --path .
```

### Custom Validation Rules
```hcl
# variables.tf with validation
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition = can(regex("^[tm][0-9]+\\.", var.instance_type))
    error_message = "Instance type must be t or m series (e.g., t3.micro, m5.large)."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  
  validation {
    condition = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block."
  }
}
```

## CI/CD Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  TF_VERSION: 1.6.0
  AWS_REGION: us-west-2

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
        
    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      
    - name: Terraform Init
      run: terraform init -backend=false
      
    - name: Terraform Validate
      run: terraform validate
      
    - name: Security Scan
      uses: aquasecurity/tfsec-action@v1.0.3
      
  plan:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    strategy:
      matrix:
        environment: [development, staging]
        
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
        
    - name: Terraform Init
      run: terraform init
      working-directory: environments/${{ matrix.environment }}
      
    - name: Terraform Plan
      run: terraform plan -no-color
      working-directory: environments/${{ matrix.environment }}
      
  apply:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    strategy:
      matrix:
        environment: [development, staging, production]
        
    environment: ${{ matrix.environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
        
    - name: Terraform Init
      run: terraform init
      working-directory: environments/${{ matrix.environment }}
      
    - name: Terraform Apply
      run: terraform apply -auto-approve
      working-directory: environments/${{ matrix.environment }}
```

Terraform cung cấp một nền tảng IaC mạnh mẽ và linh hoạt cho việc quản lý infrastructure với khả năng mở rộng cao thông qua modules và providers.