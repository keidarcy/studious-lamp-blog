+++
title = 'Migrating Next.js Landing Page Projects from Vercel to AWS'
date = 2024-12-30T11:46:46+09:00
description = 'Migrating Next.js Landing Page Projects from Vercel to AWS, a decision that significantly improved performance and cost efficiency'
+++

## Introduction

My team manages multiple landing page projects, including around 100,000 pages. Vercel initially served us well, especially with its preview URL feature for rapid feedback and built-in Next.js support to start development quickly. However, many of you may have seen this famous tweet about the surprising Vercel cost.

![Screenshot 2024-12-30 at 14.40.48.png](https://storage.googleapis.com/zenn-user-upload/f33169715eb4-20241231.png)

My team experienced a similar moment: our monthly costs jumped from ~$2,000 to ~$3,500 due to a single server error happening in one day. This led us to migrate to AWS ECS, a decision that significantly **improved performance and cost efficiency.**

![Next.js Migration May 2 2024.png](https://storage.googleapis.com/zenn-user-upload/70eacf78c244-20241231.png)

## **What Did We Migrate?**

To help understand the changes, here is a simplified version of our system diagram before and after migration:

![d1.png](https://storage.googleapis.com/zenn-user-upload/0bac76004cba-20241231.png)

Before migration, we had multiple Vercel projects routed by a Lambda@Edge function, which added significant complexity to our infrastructure.

![2.png](https://storage.googleapis.com/zenn-user-upload/9b9a750b6370-20241231.png)

After migration, we used the same ECS cluster to host API applications and multiple landing page projects. Server-to-server communication benefited from AWS’s private network, improving performance.

## **What We Gained from Migration**

### **Cost Efficiency and Predictability**

Our Vercel costs (primarily from function invocations and durations) dropped from ~$2,000 to ~$500 after moving to ECS. CloudFront costs remained relatively unchanged, but the migration eliminated unexpected spikes, ensuring a better night’s sleep!

Additional savings were achieved by using:

- **ECS:** ARM architecture, ECS on EC2, and cost-saving plans.
- **CloudFront:** CloudFront Security Bundles.

To retain the preview URL feature, we implemented it using GitHub Actions and CloudFormation.

![Screenshot 2024-12-31 at 10.44.09.png](https://storage.googleapis.com/zenn-user-upload/01351346273a-20241231.png)

### **Simplified System Complexity**

Before migration, caching was a major pain point. With at least three layers - Redis for API responses, Vercel’s cache, and CloudFront—cache invalidation was often challenging. Vercel’s cache, in particular, required complete rebuilds and redeployments to clear.

After the migration:

- **Full control of cache invalidation:** Using [on-demand revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration#on-demand-revalidation-with-revalidatepath), we can easily invalidate page-level cache.
- **Simpler CloudFront Configuration:** No need for additional Lambda@Edge functions or to integrate Vercel with CloudFront.
- **Reduced Terraform Overhead:** Managing multiple Vercel projects became unnecessary.

### **Improved Performance**

Hosting on ECS yielded significant performance improvements, boosting our SEO rankings in Google Search Console:

![screenshot_2024-10-08_at_15.57.19_720.png](https://storage.googleapis.com/zenn-user-upload/2ecbcdb19103-20241231.png)

### **Enhanced Observability**

We already had a robust observability setup for ECS, including logs, metrics, and tracing. After the migration, landing page projects aligned seamlessly with this observability framework, providing consistent insights into our systems. This made troubleshooting and performance optimization much easier.

## **Conclusion**

This migration validated our decision and provided a stable foundation for future growth. For teams facing similar challenges with scaling and costs, carefully evaluating infrastructure needs can reveal that sometimes, traditional approaches offer the most innovative solutions.

Looking ahead, we plan to migrate our entire service, including landing page projects, to Kubernetes. This will further unify our infrastructure, improve scalability, and provide even greater flexibility in managing our workloads.