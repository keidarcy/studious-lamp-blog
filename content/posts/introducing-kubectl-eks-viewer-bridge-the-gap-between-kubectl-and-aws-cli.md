+++
title = 'Introducing kubectl-eks-viewer Bridge the Gap between kubectl and AWS CLI'
description = 'kubectl-eks-viewer - A kubectl plugin to seamlessly bridge kubectl and AWS CLI for EKS resource management'
date = 2025-02-01T09:21:51+09:00
+++

As a developer working with Amazon EKS (Elastic Kubernetes Service), I often found myself juggling between different tools like `eksctl`, `aws-cli`, and `kubectl` to manage and view EKS resources. While `kubectl` provides a great interface for Kubernetes resources, accessing AWS-specific EKS resources required switching to other tools, breaking the workflow and reducing productivity.

The most common task was simply checking the status of EKS-related AWS resources - What are the desired count of my nodegroups? What addons are installed? What pod identity associations exist? Getting this information required multiple commands across different tools. I wanted a single command that could show me everything at once, following the familiar kubectl patterns I use every day.

## What is kubectl-eks-viewer?

https://github.com/keidarcy/kubectl-eks-viewer

`kubectl-eks-viewer` is a kubectl plugin that brings AWS EKS resources into your kubectl workflow. With a single command, you can view all your EKS-specific resources like nodegroups, pod identity associations, and addons. It follows kubectl conventions, so you can use familiar flags like `--context` to switch clusters or `--output=jsonpath` to extract specific information - just like you do with kubectl.

## Sample Usage

- Basic Usage

```bash
$ kubectl eks-viewer
=== cluster ===
NAME   VERSION   STATUS   PLATFORM VERSION   AUTH MODE
<REDACTED>

=== access-entries ===
ACCESS ENTRY PRINCIPAL ARN    KUBERNETES GROUPS    ACCESS POLICIES
<REDACTED>

=== nodegroups ===
NAME      STATUS   INSTANCE TYPE   DESIRED SIZE   MIN SIZE   MAX SIZE   VERSION   AMI TYPE     CAPACITY TYPE
<REDACTED>

=== fargate-profiles ===
NAME   SELECTOR NAMESPACE   SELECTOR LABELS   POD EXECUTION ROLE ARN     SUBNETS     STATUS
<REDACTED>

=== pod-identity-associations ===
ARN    AMESPACE   SERVICE ACCOUNT NAME   IAM ROLE ARN      OWNER ARN
<REDACTED>

=== insights ===
NAME       CATEGORY    STATUS
<REDACTED>
```

- Get pod-identity-associations resource only with different context

```bash
kubectl eks-viewer pod-identity-associations --context <context-name>
```

- Get access-entries only with jsonpath expression

```bash
kubectl eks-viewer access-entries -o=jsonpath="{.items.access-entries[0].AccessEntryArn}"
```

## Feedback

Would love to hear your feedback! Let me know if you find it useful or if there are any features you'd like to see. If you have any feature requests or bug reports, please submit them through GitHub [Issues](https://github.com/keidarcy/kubectl-eks-viewer/issues).