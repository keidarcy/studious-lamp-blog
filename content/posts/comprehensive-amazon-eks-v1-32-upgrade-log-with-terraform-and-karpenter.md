+++
title = 'A Comprehensive Amazon EKS v1.32 Upgrade Log with Terraform and Karpenter'
description = 'Detailed guide on upgrading Amazon EKS to v1.32 using Terraform and Karpenter, focusing on zero-downtime strategy and lessons learned'
date = 2025-04-29T09:21:51+09:00
+++

## Context

In our organization, we encountered increasing complexity with our applications and realized that we could leverage the benefits of Kubernetes over Amazon ECS. We already had a managed EKS cluster and had deployed some internal tools on Kubernetes, appreciating the platform's convenience. As a result, we decided to migrate all of our running applications to Kubernetes.

We are currently in the process of migrating about 100 ECS services to Kubernetes. A portion of our production traffic (around 20%) is already running on Kubernetes, and several internal tools are also being handled by the cluster. Before routing more critical traffic to Kubernetes, I wanted to ensure that the upgrade to EKS v1.32 would cause minimal disruption to our production services.

In the past, I had managed internal tools within the Kubernetes cluster and had upgraded Kubernetes versions a few times without careful planning. This time, I aimed for a comprehensive strategy to avoid any service interruptions. I scheduled a one-hour maintenance window for safety, intending to achieve zero downtime.

My main references for the upgrade were the [EKS Cluster Upgrades](https://eksworkshop.com/docs/fundamentals/cluster-upgrades/) guide from the [EKS Workshop](https://eksworkshop.com/) and the [Best Practices for Cluster Upgrades](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html).

## Before the Upgrade

### Infrasture overview

- All AWS resources are managed with Terraform
- Atlantis runs to execute plans and apply changes.
- All applications are deployed via ArgoCD
- Worker nodes, except for Karpenter nodes provisioned by Karpenter.
- One managed node group for Karpenter,

### Upgrade checklist

- [Kubernetes v1.32 changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md)
- [EKS v1.32 release notes](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html#kubernetes-1-32)
- [Karpenter Compatibility Matrix](https://karpenter.sh/v1.4/upgrading/compatibility/)
- EKS Upgrade Insights: This feature helps you check upgrade compatibility and provides recommendations.
    - [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
    - [kube-no-trouble](https://github.com/doitintl/kube-no-trouble) to scan for deprecated APIs
    - Addon compatibility check with the following script:

```bash
for ADDON in $(aws eks list-addons --cluster-name $CLUSTER_NAME --query 'addons' --output text); do
 echo "### $ADDON"
 aws eks describe-addon-versions --addon-name $ADDON --query 'addons[0].addonVersions[*].addonVersion' --kubernetes-version 1.32 --output text
 echo ""
done
```

## Upgrade Steps

I started with a staging environment to test the upgrade process before applying changes to production. During the upgrade, I continuously monitored several services.

### Karpenter Version

Since v1.32 does not support the current version of Karpenter we were using, I upgraded Karpenter via ArgoCD before upgrading the control plane. This process was smooth, completing in about a minute, and no nodes were recreated during this step.

### Control Plane Upgrade

I applied the following change through Atlantis:

```hcl
resource "aws_eks_cluster" "cluster" {
  name                      = "eks-cluster-name"
  version                   = "1.32"  # Updated from 1.31 to 1.32
  ...
}
```

However, I noticed that Atlantis became unresponsive during the upgrade, and while the AWS console showed the upgrade completed after about 10 minutes, the Terraform apply logs were lost.

Interestingly, I realized that Karpenter detected the version upgrade even before EKS reported it as finished and immediately began replacing nodes with the new version.

Here is the production environment control plane upgrade Atlantis Terraform output.

![Screenshot 2025-04-29 at 19.43.11.png](https://storage.googleapis.com/zenn-user-upload/f125e40e8b29-20250429.png)


### Karpenter Managed Nodes Update

After checking the events, I saw that Karpenter detected the version drift in under a minute. Over 30 nodes were replaced, which took about 20 minutes.

![Screenshot 2025-04-29 at 19.52.59.png](https://storage.googleapis.com/zenn-user-upload/bff484695b94-20250429.png)

One important lesson I learned here is that the `spec.disruption.budgets[].nodes` value in the NodePool controls the speed of node replacement. The default value is 10%, meaning a gradual update will occur with 10% voluntary disruption. If `nodes` is set to `0`, Karpenter will prevent any nodes from being considered for voluntary disruption, giving us more control over when worker nodes are updated.

Another common issue I encountered was deployment replicas conflicting with the PodDisruptionBudget (PDB) when both the replica count and PDB were set to 1.

### Managed Node Group Update

The managed node group, which is primarily used for Karpenter, is also managed by Terraform. I updated it as follows:

```
resource "aws_eks_node_group" "example" {
  cluster_name    = "eks-cluster-name"
  node_group_name = "default"
  version         = "1.32"
  ...
}
```

The update was performed as a rolling update. I monitored pod cordon and eviction processes, followed by the launching of new pods. The update took around 10 minutes.

![Screenshot 2025-04-29 at 18.21.54.png](https://storage.googleapis.com/zenn-user-upload/8a1247607279-20250429.png)

### Addon Update

Using the previous script, I discovered that all addons needed to be updated. I updated them to the latest version via Terraform:

```
resource "aws_eks_addon" "example" {
  cluster_name                = aws_eks_cluster.example.name
  addon_name                  = "kube-proxy"
  addon_version               = "v1.32.0-eksbuild.2"
  ...
}
```

This step took approximately 4 minutes.

![Screenshot 2025-04-29 at 18.23.15.png](https://storage.googleapis.com/zenn-user-upload/1843f4693361-20250429.png)

### Upgrade Production Environment

After gaining insights from the staging environment upgrade, I proceeded with the production upgrade at a slower pace. The upgrade order was: `control plane` ⇒ `managed node group` ⇒ `Karpenter managed nodes` ⇒ `addon`.

By slowing down the pace of Karpenter’s node replacement, the entire upgrade took around 1 hour.

### Monitoring and Observability

Throughout the upgrade, I actively monitored application metrics, pod lifecycle events, and Karpenter activity to ensure minimal service disruption.

One important observation: services **without high availability configurations** (i.e., single replica deployments or missing PDBs) experienced **brief downtime of approximately one minute** during node replacement or pod eviction. These gaps in redundancy made otherwise smooth infrastructure changes visible at the application layer.

![Screenshot 2025-04-29 at 17.56.40.png](https://storage.googleapis.com/zenn-user-upload/8aa9b5b61ae6-20250429.png)

The **entire upgrade process**, from the initial control plane version bump to node group rollouts and addon updates, **took about 2 hours**, including 2 environments pre-checks and post-upgrade validation.

These are events during upgrade.

![Screenshot 2025-04-29 at 17.51.19.png](https://storage.googleapis.com/zenn-user-upload/44b5675cf3f4-20250429.png)

## Errors Encountered and Learnings

Thanks to this upgrade, I found our cluster internals and the behavior of cluster components during disruptive events. A few notable issues surfaced that helped shape our future upgrade and architecture strategy:

### FailedAttachVolume

```
1 FailedAttachVolume: Multi-Attach error for volume "<REDACTED>" Volume is already exclusively attached to one node and can't be attached to another
```

This error typically occurs when a pod is rescheduled to a new node while its associated EBS volume is still attached to the previous node.

### DisruptionBlocked

```
**DisruptionBlocked**: PDB "<REDACTED>" prevents pod evictions
```

This is caused when a PodDisruptionBudget (PDB) blocks voluntary disruptions, often due to having only one pod replica. This reinforced the need to ensure that production workloads are highly available — ideally with at least 2 replicas — and that PDB configurations are reviewed prior to upgrades.

### Managed node group update

Since Kubernetes v1.32 is the last version that supports Amazon Linux 2 (AL2) AMIs, and our managed node group still uses AL2, I’ll need to upgrade the AMI type before the next version. I’m also considering moving Karpenter to run on Fargate, since it’s the only component that requires a managed node group. This could reduce maintenance and make future upgrades easier.

## Reflections and Next Steps

This upgrade helped our team deepen our understanding of EKS, Terraform, and the operational nuances of Kubernetes upgrades. A few key takeaways:

- **Karpenter reacts immediately** to version changes and may start replacing nodes before the control plane officially reports the upgrade as complete. It's important to sequence upgrades carefully and use `spec.disruption.budgets.nodes` to control rollout speed.
- **High availability matters**. We saw brief (~1 minute) downtime for non-HA services, even though the infrastructure behaved as expected. Going forward, we’ll define clear availability goals for every service.
- **PodDisruptionBudget configuration** can silently block upgrades if misaligned with replica counts. We’ll revisit our PDB and replica settings to ensure smooth disruptions.
- **Control plane logging** will be enabled to gain visibility into transient issues and make future upgrades easier to debug.
- **Maintenance windows may be unnecessary** for critical services if they are properly configured for high availability. This upgrade showed us that a one-hour window wasn’t strictly needed, and we can aim for zero-downtime in future upgrades.+++