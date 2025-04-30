# Terraform Dev Environment – Quick Start Guide

This folder stands up all AWS infrastructure for the dev environment of Cargo‑Connect using Terraform.
Everything is wired through two reusable sub‑modules:
- aws_network – VPC + subnets
- aws_compute – ECR repository + single EC2 instance that hosts your application image
The state of this environment is stored remotely in S3 (so the whole team shares one source of truth) and locked with DynamoDB (so only one plan/apply can run at a time).

## Prerequisites
### Tool / Access
- Terraform ≥ 1.0: Install from https://developer.hashicorp.com/terraform/downloads
- AWS CLI: Configure a profile or export AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY
### IAM permissions
- s3:GetObject PutObject ListBucket on bucket terraform-state-cargo-connect  
- dynamodb:GetItem PutItem DeleteItem on table terraform-lock-cargo-connect
### SSH key pair
- Name must match ec2_key_name in terraform.tfvars so you can log in to the EC2 instance
### Tip: Use an AWS role with the above permissions when working in CI or GitHub Actions.

# Folder Tour and Roles

File: `backend.tf`
Role: Remote state & lock configuration (S3 + DynamoDB). S3 object stores terraform.tfstate so everyone works from the same snapshot.DynamoDB table keeps a lock row—ensures plans/applies run one‑at‑a‑time.


Typical edits: Rare—only when you rename the bucket/table

File: `providers.tf`
Role: Pins provider & Terraform versions; sets default AWS region
Typical edits: Almost never

File: `variables.tf`
Role: Declares all input variables for this environment
Typical edits: Add new vars when modules need them

File: `terraform.tfvars`
Role: Actual values for the dev environment
Typical edits: Edit these the most

File: `main.tf`
Role: Calls the two sub‑modules & wires vars/outputs
Typical edits: Touch when you add/remove modules

File: `outputs.tf`
Role: Prints friendly values after apply
Typical edits: Extend if you need extra outputs