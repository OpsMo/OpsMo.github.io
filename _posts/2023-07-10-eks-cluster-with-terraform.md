---
title: Automating AWS EKS Infrastructure with Terraform
tags:
- EKS
- aws
- Terraform
- Iac
---

Here, we explore a comprehensive solution to automate the provisioning of Amazon EKS Cluster using Terraform. This repository, [terraform-aws-eks-infra](https://github.com/OpsMo/terraform-aws-eks-infra/tree/main), simplifies Spinning up a production-grade EKS cluster with the necessary networking, security and IAM configurations. 

### What We'll Provision

- **EKS Cluster and Managed Node Group**
- **VPC with Public and Private Subnets**
- **Internet Gateway and NAT Gateway**
- **Route Tables**
- **IAM Roles and Policies**
- **Remote Terraform state storage and locking** (S3 Bucket and DynamoDB Table)

## Prerequisites

Ensure you have the following installed:

- [Terraform](https://www.terraform.io/downloads)
- [AWS CLI](https://aws.amazon.com/cli/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) (to interact with the EKS cluster)

## So, Let’s getting Started

### 1. Clone the Repository

```bash
git clone <https://github.com/OpsMo/terraform-aws-eks-infra.git>
cd terraform-aws-eks-infra
```

### 2. Configure Backend for Remote State

Before running Terraform, set up an S3 bucket and DynamoDB table for remote state storage and locking. Update the `backend.tf` file with your S3 bucket and DynamoDB table details.

### 3. Set Variables

Edit the `variables.tf` file or create a `terraform.tfvars` file to customize inputs such as:

- AWS region
- VPC CIDR block
- EKS cluster name
- Node group configurations

### 4. Initialize Terraform

Initialize the Terraform workspace and download necessary providers:

```bash
terraform init
```

### 5. Plan and Apply

Review the execution plan:

```bash
terraform plan
```

Apply the configuration to provision the resources:

```bash
terraform apply
```

### 6. Access the EKS Cluster

After the infrastructure is provisioned, retrieve the kubeconfig for the cluster:

```bash
aws eks --region <your-region> update-kubeconfig --name <your-cluster-name>
```

You can now interact with the cluster using `kubectl`.

## Here's an overview of the repository structure with explanations for each file:

```
terraform-aws-eks-infra/
├── backend.tf       # Configuration for remote state storage
├── eks.tf           # EKS cluster and node group definitions
├── locals.tf        # Local variables
├── main.tf          # Main Terraform configuration
├── networking.tf    # VPC and networking setup
├── node_group.tf    # Node group configurations
├── outputs.tf       # Output variables
├── variables.tf     # Input variables for customization
└── README.md        # Detailed project documentation

```

### Now Here is a Simple Explanation of Each Provisioning Part to Follow Along

### 1. `backend.tf`

```hcl
## backend for remote state file s3 and dynamodb.
terraform {
    backend "s3" {
        bucket = "s3-bucket-name"
        key    = "terraform.tfstate"
        region = "us-east-1" # we can't use var.aws_region variable here directly
        encrypt = true
        dynamodb_table = "tf-locking-state"
    }
}

```

Here we are configuring Terraform to store its state file in an S3 bucket and use a DynamoDB table for locking to ensure safe remote state management.

### 2. `eks.tf`

```hcl
######## EKS iam role and policy  ########
resource "aws_iam_role" "eks" {
    name = "${var.env}-${var.eks_name}-iam-role"
    assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY

}

resource "aws_iam_role_policy_attachment" "EKSClusterPolicy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
    role = aws_iam_role.eks.name
}

######## EKS cluster  ########
resource "aws_eks_cluster" "eks" {
    name = "${var.env}-${var.eks_name}"
    role_arn = aws_iam_role.eks.arn
    version = var.eks_version
    vpc_config {
      subnet_ids = [aws_subnet.private_AZ1.id, aws_subnet.private_AZ2.id]
    }

    access_config {
      authentication_mode = "API"
      bootstrap_cluster_creator_admin_permissions = true ## Default is True
    }

    depends_on = [ aws_iam_role_policy_attachment.EKSClusterPolicy ]
}

```

Here we are defining an EKS cluster and its required IAM role:

- **IAM Role**: Grants EKS the necessary permissions to manage resources.
- **EKS Cluster**: Configures the cluster name, version and networking settings (using private subnets). It ensures secure authentication and admin permissions for initial setup.

### 3. `locals.tf`

```hcl
locals {
    tags = {
        env = var.env
        aws_region = var.aws_region
        eks_name = var.eks_name
        project = "terraform-eks-demo"
        created_by = "Mo - DevOps_Team"
    }
    aws_account_id = data.aws_caller_identity.current.account_id
}

data "aws_caller_identity" "current" {}

```

**in locals {} block** we define reusable local variables like tags for resources and the AWS account ID for use across the configuration.

### 4. `main.tf`

```hcl
### check terraform version
provider "aws" {
  region = var.aws_region
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

```

So, here we configure the AWS provider and specify the required version of the AWS provider plugin for Terraform.

### 5. `networking.tf`

```hcl
##### VPC   ####
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = {
    Name = "${var.env}-vpc"
  }
}

##### Subnets  #####
resource "aws_subnet" "private_AZ1" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.0.0/19"
  availability_zone = var.availability_zones[0]

  tags = {
    Name = "${var.env}-private-${var.availability_zones[0]}-subnet"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/${var.env}-${var.eks_name}" = "owned"
  }
}

resource "aws_subnet" "private_AZ2" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.32.0/19"
  availability_zone = var.availability_zones[1]

  tags = {
    Name = "${var.env}-private-${var.availability_zones[1]}-subnet"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/${var.env}-${var.eks_name}" = "owned"
  }
}

##### Internet gateway   ####
resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.main.id

    tags = {
      Name = "${var.env}-igw"
    }
}

```

Here we set up the VPC, subnets and an internet gateway(igw) to create the foundational network infrastructure for EKS.

### 6. `node_group.tf`

```hcl
##### IAM role for EKS node group (for workers) ########
resource "aws_iam_role" "eks-nodegroup" {
    name = "${var.env}-${var.eks_name}-nodegroup"

    assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY

}

######## EKS node group (for worker nodes) ########
resource "aws_eks_node_group" "eks-nodegroup" {
    cluster_name = "${var.env}-${var.eks_name}"
    node_group_name = "${var.env}-${var.eks_name}-nodegroup"
    node_role_arn = aws_iam_role.eks-nodegroup.arn
    version = var.eks_version
    subnet_ids = [aws_subnet.private_AZ1.id, aws_subnet.private_AZ2.id]

    scaling_config {
      desired_size = 2
      max_size = 3
      min_size = 1
    }

    labels = {
      "role" = "${var.eks_name}-workers"
    }
}

```

Here we define the IAM role for EKS worker nodes and the node group. This ensures worker nodes can operate securely within the cluster and scale as needed.

### 7. `outputs.tf`

```hcl
output "eks_cluster_name" {
  value = aws_eks_cluster.eks.name
  description = "EKS cluster name"
}

output "kubeconfig_command" {
  value = "aws eks update-kubeconfig --name ${var.env}-${var.eks_name} --region ${var.aws_region}"
  description = "To configure kubectl for the EKS cluster"
}

```

Outputs {} used to display key information, such as the EKS cluster name and a command to configure `kubectl` for cluster access.

### 8. `variables.tf`

```hcl
variable "env" {
    type = string
    default = "dev"
}

variable "eks_name" {
    type = string
    default = "eks-cluster"
}

variable "eks_version" {
    type = string
    default = "1.26"
}

variable "aws_region" {
    type = string
    default = "us-east-1"
}

variable "availability_zones" {
    type = list
    default = ["us-east-1a", "us-east-1b"]
}

```

Here we define variables for reusable configurations like environment name, cluster name, version, region, and availability zones.

So, By following the steps outlined above, you can deploy a robust Kubernetes environment tailored to your needs.

Good luck.
