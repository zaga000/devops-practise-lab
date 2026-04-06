# Task 3

## GitHub

- [ ] Install pre-commit binary on your local machine
- [ ] Implement .pre-commit-config.yaml in 'environments' repository
  - [ ] Add 'terraform_fmt' hook
  - [ ] Add 'terraform_validate' hook
- [ ] Investigate pre-commit command running on-demand and then build it into commit flow so it will be run automatically 

## Terraform

- [ ] Inspect terraform `taint` command. Can be useful for partially created Kubernetes resources
- [ ] Inspect terraform `import` command. Create a one more VPC manually, and import it to the state
- [ ] Create terraform code for imported VPC, so terraform will not delete and change it
----------------------------------------------
- [ ] Inspect terraform `workspace` command.
- [ ] Create new workspaces 'dev1' and 'dev2'
- [ ] Redesign a code to include workspace name in the `Name` tag of VPC resources
- [ ] Inspect terraform `state` command with options available
- [ ] Inspect `TF_LOG` and `TF_LOG_PATH` environment variables
