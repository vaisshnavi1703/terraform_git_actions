# terraform_git_actions
terraform github actions
# Terraform EC2 Deployment with GitHub Actions

## Overview

This project demonstrates how to provision an AWS EC2 instance using Terraform, fully automated through GitHub Actions.

## Prerequisites

* An AWS account with programmatic access (IAM user)
* A key pair named `database` created in the AWS region `us-east-1`
* Terraform CLI
* GitHub repository with Secrets configured:

  * `AWS_ACCESS_KEY_ID`
  * `AWS_SECRET_ACCESS_KEY`

## Key Files

### main.tf

```hcl
data "aws_key_pair" "example_key" {
  key_name = "database"
}

resource "aws_security_group" "de" {
  name        = "de"
  description = "Security group def for example EC2"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

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

  tags = {
    Name = "de"
  }
}

resource "aws_instance" "example_ec21" {
  ami                    = "ami-0953476d60561c955"
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnets.default.ids, 0)
  vpc_security_group_ids = [aws_security_group.de.id]
  key_name               = data.aws_key_pair.example_key.key_name
}
```

### GitHub Actions Workflow (.github/workflows/main.yml)

```yaml
name: Terraform CI

on:
  push:
    branches:
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest

    env:
      AWS_REGION: us-east-1
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

## IAM Permissions Required

Ensure the IAM user has permissions for:

* `ec2:DescribeVpcs`
* `ec2:DescribeSubnets`
* `ec2:DescribeKeyPairs`
* `ec2:CreateSecurityGroup`
* `ec2:RunInstances`
* `ec2:CreateTags`

You may also temporarily attach `AmazonEC2FullAccess` policy for convenience.

## Common Errors & Fixes

| Error                            | Cause                       | Fix                                             |
| -------------------------------- | --------------------------- | ----------------------------------------------- |
| `terraform: command not found`   | Terraform not installed     | Use GitHub Actions to automate                  |
| `Updates were rejected...`       | Local branch behind remote  | Run `git pull origin main --rebase`             |
| `UnauthorizedOperation`          | Missing IAM permissions     | Attach correct IAM policies                     |
| `no matching EC2 Key Pair found` | Key not available in region | Create key pair named `database` in `us-east-1` |

## Output

On successful execution, an EC2 instance is created in your AWS account with the specified AMI, key pair, security group, and subnet.

