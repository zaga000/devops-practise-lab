# Task-4

## GitHub

- [ ] Setup CI/CD pipeline in GitHub actions which will:
    - Run pre-commit hook with 'fmt' and 'validate' tools
    - In GitHub setup this pipeline as mandatory to succeed for PR to be merged

- [ ] Setup another pipeline with actions:
    - Terraform fmt
    - Terraform validate
    - Terraform plan
    - TErrafrom apply (with manual confirmation)
    - This pipeline to be triggered after PR is merged

- [ ] Setup one more pipeline with will:
    - Run terraform destroy
    - Pipeline to be triggered manually
