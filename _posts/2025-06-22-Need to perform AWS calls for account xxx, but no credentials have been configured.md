---
title: "How to resolve the dreadful 'Need to perform AWS calls for account xxx, but no credentials have been configured' error"
date: 2025-06-22
tags: [aws, cdk, debugging]
categories: [AWS]
description: "A breakdown of how to resolve 'Need to perform AWS calls for account xxx, but no credentials have been configured' error"
---

When working with AWS CDK in environments where OpenID Connect (OIDC) is used for role assumption—especially in CI/CD pipelines—developers often encounter cryptic permission-related errors. These can be frustrating, time-consuming, and opaque, even for experienced engineers. In this post, I’ll walk through a real-world debugging experience that took me nearly four hours to resolve, along with general strategies that can help others avoid similar pitfalls.

## The Problem: OIDC Role Has Insufficient Permissions

In many cases, the error manifests as a deployment failure due to an OIDC role not having enough permissions. You might see vague messages like:

> "AccessDenied: User is not authorized to perform: xyz on resource abc"

But there's no clear indication of *why* it's happening or *what* exactly is missing.

This typically happens because the CDK code performs a *lookup*—for example, fetching a KMS Key, a Route53 Hosted Zone, or an existing VPC—during the `cdk synth` or `cdk deploy` process. These lookups require the caller (in this case, the assumed OIDC role) to have explicit IAM permissions for the respective services and resources.

### Common CDK Lookup Triggers

Some of the typical constructs or functions that can trigger lookups:

- `HostedZone.fromLookup(...)`
- `Key.fromLookup(...)`
- `Vpc.fromLookup(...)`

If the role doesn’t have access to query these resources, CDK fails silently or with a vague message.

## Step-by-Step Debugging Guide

After hitting this issue myself, here’s what I recommend based on what finally worked:

### 1. Run CDK with Verbose Logging

Start with:

cdk synth --verbose

Or if you're deploying:

cdk deploy --verbose


Verbose logging can sometimes reveal additional clues around what resource is being accessed when the error is thrown.

> In my case, this step was partially useful. It hinted at a HostedZone lookup, but the error message lacked enough context to resolve it immediately.

### 2. Sign In with the AWS CLI and Run `cdk synth` Locally

The key insight came when I ran the same `cdk synth` locally after explicitly signing in using the AWS CLI:

```
aws sso login --profile your-profile-name
cdk synth
```

What this does is allow CDK to perform any required lookups (e.g., KMS Key ARNs or HostedZone IDs) using your credentials, and **cache the results into `cdk.context.json`**. This file then serves as a snapshot that your CI/CD pipeline or OIDC-based role can consume without needing additional permissions.

You can inspect `cdk.context.json` and confirm that things like Hosted Zone IDs or VPCs have been resolved.

### 3. Check and Clean Up Your `cdk.json`

Sometimes, CDK tags or configuration in `cdk.json` can also interfere with deploys—especially if they're applied globally or passed to stacks as environment values. If you're stuck, try temporarily removing any custom tags from `cdk.json` to isolate the issue.

Example:

```json
{
  "app": "npx ts-node bin/my-app.ts",
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": "true"
  },
  "tags": {
    "Project": "my-project",
    "Environment": "dev"
  }
}
```


What this does is allow CDK to perform any required lookups (e.g., KMS Key ARNs or HostedZone IDs) using your credentials, and **cache the results into `cdk.context.json`**. This file then serves as a snapshot that your CI/CD pipeline or OIDC-based role can consume without needing additional permissions. Not having profile and using default profile can also lead to errors so make sure you set the profile if you have one

You can inspect `cdk.context.json` and confirm that things like Hosted Zone IDs or VPCs have been resolved.

### 3. Check and Clean Up Your `cdk.json`

Sometimes, CDK tags or configuration in `cdk.json` can also interfere with deploys—especially if they're applied globally or passed to stacks as environment values. If you're stuck, try temporarily removing any custom tags from `cdk.json` to isolate the issue.

Example:

```json
{
  "app": "npx ts-node bin/my-app.ts",
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": "true"
  },
  "tags": {
    "Project": "my-project",
    "Environment": "dev"
  }
}
```

Try removing the tags section during debugging.

### 4. Re-run `cdk diff` and `cdk deploy` with Context Cached

Once your `cdk.context.json` is populated, try:

```
cdk diff
cdk deploy
```


This should avoid real-time lookups and instead use the cached values, sidestepping the permission issues that the OIDC role cannot handle directly.

## Lessons Learned

- **Always suspect a lookup** when you hit vague permission errors during CDK deploys.
- CDK’s logs are not always sufficient. You need to combine them with local synths and AWS CLI access to get the full picture.
- `cdk.context.json` is your friend. Populate it ahead of time if your CI/CD environment lacks the IAM privileges to resolve lookups.
- Be mindful of `cdk.json` tags and environment settings—they can interfere with deployments in subtle ways.

## Final Thoughts

Troubleshooting CDK issues in CI/CD with OIDC roles is not always straightforward. While CDK abstracts a lot of complexity, it also hides some of the details that are crucial for debugging. Understanding when and how lookups happen—and what roles are executing them—can save you hours of trial and error.

If you're hitting these issues, start local, synth with context, and deploy only after you've validated the lookup dependencies are resolved. It’s a bit of a dance, but one that’s unavoidable when security boundaries and automation intersect.

---

*If you’ve run into similar CDK lookup issues with OIDC roles or have alternative strategies, feel free to reach out or drop your thoughts in the comments.*



