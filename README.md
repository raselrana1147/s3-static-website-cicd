# Deploying a Static Website to AWS S3 with GitHub Actions (CI/CD)

A complete, production-style guide for automatically deploying a static website to Amazon S3 whenever code is pushed to GitHub.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Configure the S3 Bucket](#step-1--configure-the-s3-bucket)
5. [Step 2 — Create an IAM User with Least-Privilege Access](#step-2--create-an-iam-user-with-least-privilege-access)
6. [Step 3 — Store Credentials as GitHub Secrets](#step-3--store-credentials-as-github-secrets)
7. [Step 4 — Create the GitHub Actions Workflow](#step-4--create-the-github-actions-workflow)
8. [Step 5 — Deploy and Verify](#step-5--deploy-and-verify)
9. [Optional: CloudFront Cache Invalidation](#optional-cloudfront-cache-invalidation)
10. [Security Best Practices](#security-best-practices)
11. [Troubleshooting](#troubleshooting)

---

## Overview

This guide sets up a CI/CD pipeline where:

> **Push to `main` branch → GitHub Actions triggers → Files sync to S3 automatically**

No manual uploads, no local AWS CLI commands — every merge to `main` becomes a live deployment.

---

## Architecture

```
┌─────────────┐      push      ┌──────────────────┐      sync      ┌─────────────┐
│  Developer  │ ─────────────▶ │  GitHub Actions   │ ─────────────▶ │   S3 Bucket  │
│  (git push) │                │     Workflow       │                │ (static site)│
└─────────────┘                └──────────────────┘                └─────────────┘
                                                                            │
                                                                            ▼
                                                                    ┌─────────────┐
                                                                    │   End Users  │
                                                                    └─────────────┘
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS Account | With permissions to create IAM users/policies |
| S3 Bucket | Already created and configured for static website hosting |
| GitHub Repository | Contains your website source files |
| AWS CLI (optional) | Useful for local testing before automating |

---

## Step 1 — Configure the S3 Bucket

1. Create S3 bucket Properties **Any Region**
2. Open **S3 → your bucket → Properties**
3. Scroll to **Static website hosting** → **Edit**
4. Enable it, set:
   - **Index document:** `index.html` (starting point)
   - **Error document:** `error.html` (optional but recommended)
4. Save, and note the **Bucket website endpoint** shown — this is your live URL
5. If you get any 403 Forbidden error **Add Bucket policy**, under **Permission → Bucket Policy → Edit* Bucket Policy allow public read access (skip this if using CloudFront with Origin Access Control instead):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::Bucket-Name/*"
            ]
        }
    ]
}
```

Also confirm **Block Public Access** settings are not blocking bucket policies, if you intend the site to be public.

---

## Step 2 — Create an IAM User with Least-Privilege Access

Avoid using root credentials or broad `AmazonS3FullAccess`. Scope permissions to exactly what the pipeline needs.

1. **IAM → Users → Create user**
   - Name: `github-actions-deployer`
   - Access type: Programmatic access (no console login needed)

2. Attach an **inline policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3DeploySync",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ]
    }
  ]
}
```

3. Go to **Security credentials → Create access key**
   - Use case: *Application running outside AWS*
4. Copy the **Access Key ID** and **Secret Access Key** — the secret is shown only once

> 🔐 **Recommended upgrade:** Once this pipeline works, migrate to **OpenID Connect (OIDC)**, which lets GitHub Actions assume an IAM role without storing any long-lived AWS keys at all. See [Security Best Practices](#security-best-practices).

---

## Step 3 — Store Credentials as GitHub Secrets

In your repository: **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Example Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | `AKIAxxxxxxxxxxxxxxxx` |
| `AWS_SECRET_ACCESS_KEY` | `wJalrxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `AWS_REGION` | `ap-southeast-1` |
| `S3_BUCKET_NAME` | `my-static-site-bucket` |

Never commit these values directly into your repository or workflow file.

---

## Step 4 — Create the GitHub Actions Workflow

Create the file: `.github/workflows/deploy.yml`

```yaml
name: Deploy Static Site to S3

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Sync files to S3
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} \
            --delete \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --exclude "README.md"
```

### Notes on the workflow
- `--delete` removes files from S3 that no longer exist in the repo (keeps bucket in sync)
- `--exclude` prevents repo metadata (`.git`, `.github`, README) from being uploaded to the live site
- If your site's files live in a subfolder (e.g. a `dist/` or `build/` output), change the sync source:
  ```yaml
  run: aws s3 sync ./dist s3://${{ secrets.S3_BUCKET_NAME }} --delete
  ```

---

## Step 5 — Deploy and Verify

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add S3 deployment workflow"
git push origin main
```

1. Go to your repo's **Actions** tab
2. Watch the `Deploy Static Site to S3` workflow run
3. On success, open your **S3 static website endpoint** and confirm the changes are live

---

## Optional: CloudFront Cache Invalidation

If a CloudFront distribution sits in front of the bucket (recommended for HTTPS + caching + custom domain), add this step so users see updates immediately instead of waiting for the cache to expire:

```yaml
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

Add `CLOUDFRONT_DISTRIBUTION_ID` as an additional GitHub secret, and give the IAM user/role `cloudfront:CreateInvalidation` permission.

---

## Security Best Practices

| Practice | Why it matters |
|---|---|
| Use least-privilege IAM policies | Limits blast radius if credentials leak |
| Migrate to OIDC instead of static keys | Removes long-lived secrets entirely |
| Never commit secrets to the repo | Use GitHub Secrets exclusively |
| Restrict workflow triggers to `main` | Prevents accidental deploys from feature branches |
| Enable branch protection on `main` | Requires PR review before code reaches production |
| Rotate IAM access keys periodically | Reduces risk from stale credentials |

### OIDC Setup (Recommended Next Step)

Instead of storing AWS keys as GitHub secrets, GitHub can authenticate directly with AWS using a trust relationship:

1. Create an **IAM OIDC Identity Provider** in AWS for `token.actions.githubusercontent.com`
2. Create an **IAM Role** with a trust policy scoped to your GitHub repo/branch
3. Replace the `configure-aws-credentials` step with:

```yaml
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_ACCOUNT_ID:role/github-actions-role
          aws-region: ${{ secrets.AWS_REGION }}
```

4. Add `permissions: id-token: write` at the top of the workflow

This eliminates the need to store `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` entirely.

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `Access Denied` during sync | IAM policy missing permissions or wrong bucket ARN | Recheck the IAM policy resource ARNs |
| Workflow doesn't trigger | Push was to a different branch | Confirm `branches: [main]` matches your default branch |
| Site not updating after deploy | CloudFront caching old files | Add the cache invalidation step |
| `NoSuchBucket` error | Bucket name typo or wrong region | Verify `S3_BUCKET_NAME` and `AWS_REGION` secrets |
| 403 Forbidden on website URL | Bucket policy not public / Block Public Access enabled | Review Step 1 bucket policy settings |

---

**Maintained as part of personal DevOps practice documentation.**
