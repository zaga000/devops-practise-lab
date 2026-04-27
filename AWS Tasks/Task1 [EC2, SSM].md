# Associate Cloud Engineer - EC2 & CloudWatch Comprehensive Lab

## Overview
This comprehensive practical lab is designed to reinforce theoretical knowledge of EC2 and CloudWatch at the AWS Certified Associate Cloud Engineer level. The lab covers all key aspects according to the provided lectures.

## Prerequisites
- AWS Account with appropriate permissions
- AWS CLI installed and configured
- Basic Linux command knowledge
- Terraform (optional, for IaC part)

---

## Part 1: EC2 Instance Management Fundamentals

### Task 1.1: EC2 Instance Launch & Configuration
**Objective:** Create and configure EC2 instances with different types and configurations

**Steps:**
1. **Launch EC2 Instance:**
   - Create an EC2 instance of type t3.micro in the default VPC
   - Use Amazon Linux 2023 AMI
   - Configure Security Group for SSH (port 22) and HTTP (port 80)
   - Use a key pair for access
   - Add User Data script to install Apache web server

2. **Instance Type Modification:**
   - Change instance type from t3.micro to t3.small
   - Verify that the instance works correctly after the change
   - Save changes and restart the instance

3. **Verification:**
   - Connect to the instance via SSH
   - Check Apache web server status
   - Verify public IP and accessibility through browser

### Task 1.2: EC2 Placement Groups
**Objective:** Configure placement groups for network performance optimization

**Steps:**
1. **Create Cluster Placement Group:**
   - Create a cluster placement group
   - Launch 2 EC2 instances (t3.medium) in this placement group
   - Verify that instances are in the same availability zone

2. **Create Spread Placement Group:**
   - Create a spread placement group
   - Launch 3 instances (t3.micro) in different availability zones
   - Verify instance distribution

---

## Part 2: AMI Management & Image Builder

### Task 2.1: AMI Creation & Management
**Objective:** Create and manage AMIs for different use cases

**Steps:**
1. **Create AMI from Running Instance:**
   - Take one of the previously created instances
   - Install additional software (nginx, git, htop)
   - Create AMI from this instance
   - Use "No Reboot" option and compare results

2. **AMI Migration:**
   - Create a new instance from the created AMI in another availability zone
   - Verify that all installed software is available
   - Create snapshot of root volume

3. **EC2 Image Builder:**
   - Create an Image Builder pipeline
   - Configure automatic AMI creation with basic setup
   - Set up schedule for automatic AMI updates

---

## Part 3: Systems Manager (SSM) Integration

### Task 3.1: SSM Agent & Resource Management
**Objective:** Configure SSM for centralized instance management

**Steps:**
1. **SSM Agent Installation:**
   - Check SSM agent status on all instances
   - Install/update SSM agent if needed
   - Verify connectivity to SSM endpoints

2. **Resource Groups & Tags:**
   - Create tags for all instances (Environment, Project, Owner)
   - Create SSM Resource Group based on tags
   - Verify that instances appear in resource group

### Task 3.2: SSM Documents & Run Command
**Objective:** Use SSM for operations automation

**Steps:**
1. **Create Custom SSM Document:**
   - Create SSM document for software installation
   - Add commands for system diagnostics
   - Test document on one instance

2. **Run Command Execution:**
   - Execute command on all instances in resource group
   - Check execution results
   - Create scheduled command for automatic operations

### Task 3.3: SSM Parameter Store
**Objective:** Centralize configuration and secrets

**Steps:**
1. **Create Parameters:**
   - Create String parameters for application configuration
   - Create SecureString parameters for passwords
   - Configure parameter versioning

2. **Integration with Applications:**
   - Configure parameter reading from EC2 instances
   - Create IAM role for Parameter Store access
   - Test parameter access

### Task 3.4: SSM Session Manager & Fleet Manager
**Objective:** Secure access to instances without SSH keys

**Steps:**
1. **Session Manager Setup:**
   - Configure Session Manager for all instances
   - Create IAM policy for Session Manager access
   - Test shell access through Session Manager

2. **Fleet Manager:**
   - Configure Fleet Manager for visual management
   - Check ability to view logs and files
   - Test remote desktop functionality

---

## Part 4: CloudWatch Monitoring & Alerting

### Task 4.1: CloudWatch Metrics & Custom Metrics
**Objective:** Set up comprehensive EC2 instance monitoring

**Steps:**
1. **Basic Metrics Setup:**
   - Check standard metrics for all instances
   - Create CloudWatch Dashboards for visualization
   - Set up alarms for CPU utilization > 80%

2. **Custom Metrics:**
   - Install Unified CloudWatch Agent on instances
   - Configure memory metrics collection
   - Add custom metrics for disk space utilization

3. **Log Collection:**
   - Configure CloudWatch Logs agent
   - Create log groups for Apache access and error logs
   - Set up log rotation and retention policies

### Task 4.2: CloudWatch Alarms & Notifications
**Objective:** Create problem notification system

**Steps:**
1. **Create Alarms:**
   - CPU Utilization > 80% for 5 minutes
   - Memory Utilization > 90% for 3 minutes
   - Disk Space < 10% free
   - Status Check failures

2. **SNS Integration:**
   - Create SNS topic for notifications
   - Subscribe email address to notifications
   - Test alarm notifications

3. **Auto Scaling Integration:**
   - Create Auto Scaling Group with EC2 instances
   - Configure scaling policies based on CloudWatch alarms
   - Test automatic scaling

---

## Part 5: Advanced EC2 Features

### Task 5.1: EC2 Hibernate
**Objective:** Use hibernate for instance state preservation

**Steps:**
1. **Enable Hibernate:**
   - Create instance with hibernate support
   - Install applications and create files in memory
   - Test hibernate functionality

2. **State Preservation:**
   - Run application in instance
   - Perform hibernate operation
   - Verify state preservation after resume

### Task 5.2: Instance Scheduler
**Objective:** Automate instance startup and shutdown

**Steps:**
1. **Setup Instance Scheduler:**
   - Deploy Instance Scheduler solution
   - Configure schedule for development instances
   - Create different schedules for different environments

2. **Cost Optimization:**
   - Configure automatic shutdown of non-production instances
   - Create exceptions for critical instances
   - Monitor cost savings

---

## Part 6: Troubleshooting & Maintenance

### Task 6.1: EC2 Troubleshooting
**Objective:** Develop diagnostic and problem-solving skills

**Steps:**
1. **SSH Issues:**
   - Simulate SSH connection problems
   - Use EC2 Instance Connect for diagnostics
   - Check Security Groups and NACLs

2. **Status Checks:**
   - Monitor System and Instance Status Checks
   - Create alarms for status check failures
   - Develop troubleshooting procedures

### Task 6.2: SSM Patch Manager
**Objective:** Automate system updates

**Steps:**
1. **Patch Manager Setup:**
   - Create patch baseline for Amazon Linux
   - Configure maintenance windows
   - Test patching process

2. **Compliance Reporting:**
   - Check patch compliance status
   - Create reports for audit
   - Configure automated remediation

---

## Part 7: Cleanup & Cost Management

### Task 7.1: Resource Cleanup
**Objective:** Properly delete created resources

**Steps:**
1. **Instance Termination:**
   - Stop and delete all EC2 instances
   - Delete created AMIs and snapshots
   - Clean up Security Groups and key pairs

2. **SSM Resources:**
   - Delete SSM documents and parameters
   - Clean up resource groups
   - Remove Fleet Manager configurations

3. **CloudWatch Resources:**
   - Delete alarms and dashboards
   - Clean up log groups
   - Remove custom metrics

### Task 7.2: Cost Analysis
**Objective:** Analyze costs and optimization

**Steps:**
1. **Cost Review:**
   - Check AWS Cost Explorer
   - Analyze costs by service
   - Identify optimization opportunities

2. **Best Practices Documentation:**
   - Create best practices document
   - Describe lessons learned
   - Formulate production recommendations

---

## Evaluation Criteria

### Technical Skills (40%)
- Proper creation and configuration of EC2 instances
- Effective use of AMI and Image Builder
- Deep understanding of SSM functionalities
- Comprehensive CloudWatch monitoring setup

### Problem Solving (30%)
- Ability to diagnose EC2 problems
- Effective use of troubleshooting tools
- Configuration optimization for performance

### Best Practices (20%)
- Following security best practices
- Proper use of tagging and resource management
- Effective cost management

### Documentation & Communication (10%)
- Clear documentation of created configurations
- Proper description of processes and procedures
- Effective communication of technical solutions

---

## Success Metrics

### Functional Requirements
- [ ] All EC2 instances created and working correctly
- [ ] AMI creation and Image Builder pipeline working
- [ ] SSM functionalities fully configured
- [ ] CloudWatch monitoring and alarms working
- [ ] All resources successfully cleaned up after completion

### Performance Requirements
- [ ] Instances respond within < 5 seconds
- [ ] Alarms trigger within < 2 minutes
- [ ] Patching completes within maintenance windows
- [ ] Cost optimization achieved > 20% savings

### Security Requirements
- [ ] All instances use IAM roles
- [ ] SSH access through Session Manager without keys
- [ ] All sensitive data in Parameter Store encrypted
- [ ] Network access properly restricted

---

## Additional Resources

### Documentation Links
- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/)
- [Systems Manager User Guide](https://docs.aws.amazon.com/systems-manager/)

### AWS CLI Commands Reference
```bash
# EC2 Commands
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro
aws ec2 create-image --instance-id i-xxx --name "My-AMI"
aws ec2 create-placement-group --name my-cluster --strategy cluster

# SSM Commands
aws ssm send-command --document-name "AWS-RunShellScript" --targets "Key=tag:Environment,Values=Dev"
aws ssm put-parameter --name "/myapp/config" --value "production" --type String
aws ssm start-session --target i-xxx

# CloudWatch Commands
aws cloudwatch put-metric-alarm --alarm-name "CPU-High" --metric-name CPUUtilization
aws logs create-log-group --log-group-name "/myapp/apache"
```

### Troubleshooting Checklist
- [ ] Security Groups properly configured
- [ ] IAM permissions sufficient
- [ ] SSM agent running and connected
- [ ] CloudWatch agent properly configured
- [ ] Network connectivity working
- [ ] Resource limits not exceeded

---

## Conclusion

This comprehensive lab provides practical reinforcement of all key EC2 and CloudWatch concepts for the Associate Cloud Engineer level. Successful completion of the lab will demonstrate readiness to work with AWS infrastructure in real project environments.

**Estimated Time:** 8-12 hours
**Difficulty:** Intermediate to Advanced
**AWS Services Covered:** EC2, CloudWatch, Systems Manager, SNS, IAM, Lambda
