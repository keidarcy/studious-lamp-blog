+++
title = 'Integrate E1s to Improve Daily Operations with Security'
description = 'How we improved our daily operations by integrating e1s with ECS Exec with simplified operations, enhanced security through IAM roles workflow.'
date = 2024-12-31T09:21:51+09:00
+++

## Introduction

Managing infrastructure security and operational efficiency is critical for modern tech teams. In my organization, we implemented a feature allowing developers to access databases via ORM methods, which required duplicating deployments for developer access. Previously, we relied on dedicated EC2 instances and specific pipelines to facilitate this process. However, this approach presented several challenges:

- **Managing SSH keys** for server access.
- **Limited scalability** and significant operational overhead.
- **Complex CI/CD pipelines** tied to EC2 instances.
- **High AWS costs** due to persistent EC2 usage.
- **Security risks** from less granular access control and limited audit capabilities.

To address these issues, I integrated [e1s](https://github.com/keidarcy/e1s)(a CLI tool to manage ECS resources in the terminal) with the ECS Exec, replacing the EC2-based workflow for database access. This transition simplified operations, reduced costs, and enhanced security.

This blog outlines the transformation, comparing the old and new workflows, and highlights the benefits of the integration.

## Old vs. New Workflow

### Old Workflow: EC2-Based Operations

**Access via Dedicated EC2 Instances**

- Team members used SSH keys to log into specific EC2 servers.
- Operational tasks were performed manually on these instances.

![old-workflow.png](https://storage.googleapis.com/zenn-user-upload/cbe42adfc429-20241231.png)

**New Workflow: ECS-Based Operations with e1s**

- Team members use [e1s](https://github.com/keidarcy/e1s) to connect directly to ECS containers based on IAM roles granted through [OneLogin](https://github.com/onelogin/onelogin-python-aws-assume-role), our company-wide identity provider.
- ECS Exec enables seamless container access without intermediary servers.

![new-workflow.png](https://storage.googleapis.com/zenn-user-upload/c733932fd992-20241231.png)

## Benefits Overview

### **Reduced Complexity**

- Eliminates the need for SSH keys and dedicated EC2 servers, reducing manual operations related to key management.
- Simplifies CI workflows by removing EC2 deployment requirements.

### **Enhanced Security**

- No longer requires managing SSH keys, eliminating risks from forgotten or misplaced keys.
- Isolates production secrets to specific ECS containers.
- Enforces Role-Based Access Control (RBAC) through OneLogin-granted IAM roles.
- Logs ECS Exec output to S3 for improved auditing and traceability.

### **Cost Efficiency**

- Significantly reduces AWS expenses by removing persistent EC2 instances.
- Lowers CI costs by eliminating EC2-specific deployment pipelines.

## Conclusion

Migrating to [e1s](https://github.com/keidarcy/e1s) has transformed our operations, offering a leaner, more secure, and cost-efficient workflow. I hope [e1s](https://github.com/keidarcy/e1s) becomes a part of your toolkit, simplifying your operations just as it has for us!
