+++
title = 'How We Improved Developer Experience and Reduced SRE Toil by Migrating to TypeScript-Based Infrastructure'
description = 'How we empowered developers to manage cloud infrastructure by migrating from Terraform to CDKTF, reducing SRE toil through TypeScript and automated PR workflows.'
date = 2024-12-17T09:21:51+09:00
+++

![](https://storage.googleapis.com/zenn-user-upload/efd42572ae73-20241217.png)

## The Challenge

While managing infrastructure resources on service providers like AWS and GCP for multiple services, our engineering team frequently needed simple infrastructure changes - creating S3 buckets, modifying CloudFront behaviors, or updating IAM permissions. While we used Terraform, developers weren't comfortable with HCL (HashiCorp Configuration Language), creating redundant or repetitive work for SRE team.

After evaluating several Infrastructure as Code (IaC) solutions including Pulumi, AWS CDK, AWS CloudFormation, we chose [CDK for Terraform](https://developer.hashicorp.com/terraform/cdktf) (CDKTF) because it offered:

- Multi-cloud provider support
- Terraform's mature toolchain
- TypeScript integration for developer familiarity


## Implementation Journey

### Migration Process

Our SRE team completed the migration from HCL to CDKTF in approximately 6 months. Key milestones included:

1. Converting existing infrastructure to TypeScript-based CDKTF
2. Implementing Atlantis for pull request automation
3. Establishing security controls and review processes
4. Setting up automated drift detection

This transformation delivered immediate benefits:

- Reduced SRE team [toil](https://sre.google/sre-book/eliminating-toil/)
- Improved focus on scalability and observability
- Better infrastructure understanding across teams
- Increased overall productivity

### Setup

We chose [Bun](https://bun.sh/) as our package manager and [Biome](https://biomejs.dev/) as our linter for their superior CI installation speed and performance. The project setup is straightforward - our `cdktf.json` configuration simply specifies:

```json
{
  "app": "bun run main.ts"
}

```

For teams looking to start with a similar setup, I've created a [template repository](https://github.com/keidarcy/cdktf-bun-init) demonstrating the complete basic configuration.

### **Multi-Environment Management**

Our infrastructure spans multiple AWS accounts, GCP projects, and Datadog organizations. To avoid complexity in the sample code, I will only consider AWS in the following examples. We define different stacks for each account, using environment variables to determine which stack to initialize in the CDKTF main file. We use environment-specific stacks to:

- Isolate resources and prevent cross-impact
- Enable parallel work across teams
- Maintain separate state files in S3

Here's our main CDKTF file:

```tsx
const app = new App()

const isShared = NODE_ENV === 'shared-staging' || NODE_ENV === 'shared-prod'
const isProductA = NODE_ENV === 'product-a-prod' || NODE_ENV === 'product-a-staging'
const isProductB = NODE_ENV === 'product-b-prod' || NODE_ENV === 'product-b-staging'

if (isProductA) {
  new ProductAStack(app, `product-a-${NODE_ENV}`)
}

if (isShared) {
  new SharedStack(app, `shared-${NODE_ENV}`)
}

if (isProductB) {
  new ProductBStack(app, `product-b-${NODE_ENV}`)
}

app.synth()
```

### Atlantis Integration

In Atlantis, we define pre-workflow hooks as those that handle different environments. While this approach might seem brute force, Bun's performance makes it highly efficient:

```yaml
pre_workflow_hooks:
  - run: |
      bun install --production &&
      NODE_ENV=product-a-staging cdktf synth --output ./product-a-staging &&
      # Additional environments
```

Our `atlantis.yaml` in the repository root defines projects:

```yaml
version: 3
projects:
  - name: product-a-staging
    dir: staging/stacks/product-a-staging
    # Additional project definitions...
```

After set up, `Atlantis` listens to GitHub webhooks and responds to PR. When a team member comments like `atlantis plan -p product-a-staging`, `Atlantis` executes a Terraform plan for the product-a staging environment. Similarly, `atlantis apply -p product-a-staging` applies the previously generated plan to the corresponding environment.

![](https://storage.googleapis.com/zenn-user-upload/1ccafc7d4fdf-20241217.png)

This automation ensures our master branch accurately reflects the deployed infrastructure state. While we primarily manage infrastructure through code, we maintain manual console access for incident response and staging environment testing. To keep infrastructure code synchronized with the actual state, we implemented daily drift detection that identifies and reports any discrepancies between code and deployed resources.

### **Security and Automation**

**Access Control**

We implemented a balanced security approach:

- All organization members can contribute code
- Anyone can trigger **`atlantis plan`**
- Only SRE team members can execute **`atlantis apply`**
- Changes must be applied before merging to master

**Drift Detection**

We automated infrastructure drift detection using GitHub Actions:

```yaml
name: Drift Detection Check
run-name: drift-detection
on:
    schedule:
      - cron: "00 00 * * 0-5"

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Plan product-a-staging
        if: github.event.schedule == '00 00 * * 0-5'
        env:
          NODE_ENV: product-a-staging
          STACK: product-a-staging
        run: |
          gh pr comment $PR_NUMBER --body "atlantis plan -d $NODE_ENV/stacks/$STACK"
```

This system:

- Creates one PR and runs plans for all different stacks daily
- Collects results and sends Slack notifications with plan links
- Runs before work hours to allow immediate attention to any differences
- Maintains infrastructure consistency


![](https://storage.googleapis.com/zenn-user-upload/3c10e7446b51-20241217.png)


### Resource Tagging

We've implemented a sophisticated tagging system to organize and track resources:

```tsx
export class TagsAddingAspect implements IAspect {
  constructor(private tagsToAdd: Record<string, Record<string, string>>) {}

  visit(node: IConstruct): void {
    if (isTaggableConstruct(node)) {
      for (const [key, value] of Object.entries(this.tagsToAdd)) {
        if (node.node.path.includes(key)) {
          const currentTags = node.tagsInput || {}
          node.tags = { ...value, ...currentTags }
        }
      }
    }
  }
}

type TaggableConstruct = IConstruct & {
  tags?: { [key: string]: string }
  tagsInput?: { [key: string]: string }
}

function isTaggableConstruct(x: IConstruct): x is TaggableConstruct {
  const isTaggable = 'tags' in x && 'tagsInput' in x
  const isNotDataSource = !(x instanceof TerraformDataSource)
  return isTaggable && isNotDataSource
}
```

In stack file:

```jsx
Aspects.of(this).add(
  new TagsAddingAspect({
    atlantis: {
      app_name: APP_NAME.ATLANTIS,
      area_name: AREA_NAME.COMPANY_SHARED,
    },
  })
)
```

This system ensures consistent tagging across resources with minimal code duplication. We add `app_name` and `area_name` to all resources as cost allocation tags and an indicator to understand resource scope.

![](https://storage.googleapis.com/zenn-user-upload/f1a46b68f43c-20241217.png)

## **Challenges and** Future Improvements

### Continuous Reconciliation

Inspired by ArgoCD's approach to Kubernetes resource management, we're exploring [Burrito](https://github.com/padok-team/burrito) to bring similar continuous state reconciliation to our Terraform ecosystem, potentially eliminating the need for scheduled drift detection.

### Stack Optimization

As our stacks grow, we face challenges with:

- Increasing Terraform command execution time
- Resource coupling
- Atlantis locks blocking development

We're planning to implement more granular stack separation to address these issues.

### Community Considerations

The CDKTF community, while growing, isn't as active as the traditional Terraform or AWS CDK communities. Additionally, OpenTofu's uncertain commitment to support creates a consideration point for long-term planning. We're actively monitoring these aspects to ensure our infrastructure strategy remains sustainable. Note: While we've confirmed CDKTF is currently OpenTofu compatible, there's no official [guarantee](https://github.com/opentofu/opentofu/issues/1335).

### Terraform Module Support

One significant advantage of CDKTF is its Terraform module support, and TypeScript provides strong type-checking. However, when using Terraform Modules, any [map](https://developer.hashicorp.com/terraform/language/expressions/types#maps-objects) type HCL variables become `any` in TypeScript, which compromises type safety. Due to this incomplete type support, we remain conservative in our use of Terraform modules.

### Developer Team Involvement

While CDKTF migration has significantly increased developer contributions to our infrastructure repository, the SRE team aims to foster even more involvement. We continue looking for ways to encourage broader participation from our development teams.

## **Conclusion**

Our migration to CDKTF has significantly improved our infrastructure management:

- Increased developer participation in infrastructure changes
- Enhanced security through automated workflows
- Better resource organization and tracking
- Reduced operational overhead

While challenges remain, particularly around performance and community support, the benefits have justified our decision. We continue to iterate on our implementation, focusing on automation, security, and developer experience.
