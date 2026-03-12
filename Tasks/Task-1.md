# Task 1

## GitHub
  * [ ] Create directory Dev in environments Repo

## AWS

  * [ ] Create S3 bucket in root account for TF State
  * [ ] Create DynamoDB in root account for Tf State Lock
  * [ ] Create Admin role in Dev account
  * [ ] Grant 'terraform' user permissions to assume Admin role in Dev account

## Terraform

### Configure provider

  * [ ] Define an AWS TF provider in a separate file 'providers.tf'
  * [ ] Define required version for AWS Provider with major version of 6 and minor to be the latest
  * [ ] Region defined in a provider to be a variable (set validation rule for the variable, so only EU region can be used)
  * [ ] Setup provider to assume Admin role in Dev account
  * [ ] Setup default Tag on a provider level: Terraform=true

### Configure Terraform

  * [ ] Terraform version to be used no less then 1.14.0

### Create a configuration for AWS VPC

  * [ ] VPC Configuration to be placed in 'vpc.tf' file
  * [ ] CIDR block to be defined as a variable (mandatory fields: description, type, default). OPTIONAL: regex validation for CIDR format 
  * [ ] VPC should contain 2 public, 2 private and 2 db subnets. Use Data block to fetch availability zones
  * [ ] IGW to be attached to a VPC
  * [ ] NAT GW to be attached to a VPC
  * [ ] Tags to be placed in 'locals', and contain 2 key-value pairs: Name, Environment
  * [ ] Route tables to have necessary routes for resources placed in Public and Private subnets to have access to the Internet (Private instances shouldn't be accessible FROM the Internet)
  * [ ] Add VPC ID and subnet CIDRs to outputs. Outputs to have description section
