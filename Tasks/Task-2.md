# Task 2

## GitHub

- [x] Create repository `terraform-aws-vpc`
- [x] `main` branch to be used as a default one

## Terraform

### Modules repo

- [x] Move all the code related to VPC from 'environments' repository to `terraform-aws-vpc`
- [x] Replace subnet CIDRs definition with a function 'cidrsubnet'. Numer of subnets has to be the same
- [x] Inspect usage of **heredocs** for variables descriptions for multiline definitions and use them in the module
- [x] Add a mandatory tag from a module `Module="VPC"`
- [x] Tag main branch in the repository with `v0.0.1`

### Environments repo

- [x] Call the VPC Module from the Dev environment configuration, with a version definition
- [x] Add provisioner to the configuration which will call AWS API via aws cli and output the list of VPCs in the account to a file 'vpcs.txt'
- [x] Add another provisioner will output all subnet CIDRs to a file 'subnets.txt' by using 'self' object
- [x] Add another provisioner which will delete a local file 'vpcs.txt' when the VPC is destroyed
- [x] Deploy an S3 bucket from a public module (with the bare minimum configuration)

Optional:

- [x] Call a module one more time, but define VPC Configuration in a format below and read it from a local file `vpc.json`:

```json
{
  "vpcs": [
    {
      "vpc_name": "my-vpc-1",
      "vpc_attributes": {
        "cidr_block": "10.0.0.0/16",
        "private_subnets_count": 2,
        "public_subnets_count": 2,
        "db_subnets_count": 2
      }
    },
    {
      "vpc_name": "my-vpc-2",
      "vpc_attributes": {
        "cidr_block": "10.1.0.0/16",
        "private_subnets_count": 2,
        "public_subnets_count": 2,
        "db_subnets_count": 2
      }
    }
  ]
}
```

- [x] Create a variable `vpcs` with a type of `map(list(object({})))` and define all the fields required to be presented in the object following the JSON above
- [x] Define the value for `vpcs` variable in the *.tfvars file according to the JSON above
- [x] Try to omit one of the fields, i.e. `cidr_block` and check how Terraform will react on it
- [x] Make this filed optional in the variable definition like in the [example here](https://spacelift.io/blog/terraform-optional-variable)
- [x] Add an output with all VPC Private Subnets CIDRs
- [x] Try to define '0' for public subnets and run `terraform plan/apply` to see how Terraform will react on it
