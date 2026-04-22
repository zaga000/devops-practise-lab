# Task-5

## GitHub

[ ] Add [Terraform Docs])(https://github.com/terraform-docs/terraform-docs) to pre-commit hook for the module repository
[ ] Add validation for the PR pipeline if Terraform Docs are updated or not. If not, fail the pipeline
[ ] Add .tflint file and add tflint pre-commit hook. [Documentation](https://github.com/terraform-linters/tflint)
[ ] Add this validation to the pipeline for PR
----------------------------------------------
[ ] Add CHANGELOG.md file to the repository with Terraform module for VPC. File format:

```markdown
# CHANGELOG

## 0.0.2
* [Task-2] - Feature 2 added.

## 0.0.1
* [Task-1] - Feature 1 added. 
```

[ ] Add GitHub pipeline which will tag main branch in VPC Modules repository with semantic version and message from CHANGELOG.

----------------------------------------------
Add pull request template to the environment and module repos

pull request template example:

```markdown
Please ensure all items below are completed before submitting a PR:
 
- [ ] Pre-commit hooks were run locally
- [ ] Pull-request subject mentions task name and type of changes (feature/fix), if applicable
```
