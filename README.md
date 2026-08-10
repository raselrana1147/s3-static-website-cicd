# DevOps Practice Notes — Static Website CI/CD on AWS S3

Personal reference documentation: deploying a static website to AWS S3 with an automated CI/CD pipeline using GitHub Actions.

---

## 📑 Table of Contents
1. [Launch an AWS Machine (EC2 + Key Pair)](#1-launch-an-aws-machine-ec2--key-pair)
2. [Create IAM User & Permissions](#2-create-iam-user--permissions)
3. [Create S3 Bucket](#3-create-s3-bucket)
4. [Configure AWS CLI](#4-configure-aws-cli)
5. [Configure Static Website Hosting on S3](#5-configure-static-website-hosting-on-s3)
6. [Create GitHub Repository](#6-create-github-repository)
7. [Set AWS Credentials as GitHub Secrets](#7-set-aws-credentials-as-github-secrets)
8. [Write the GitHub Actions Workflow (Explained)](#8-write-the-github-actions-workflow-explained)
9. [Troubleshooting](#9-troubleshooting)
10. [Next Steps](#10-next-steps)

---

## 1. Launch an AWS Machine (EC2 + Key Pair)

> This step is **optional** — only needed if you want a Linux server (e.g. to test builds, host tools, or practice server administration alongside S3). If you're only deploying static files to S3, you can skip straight to Step 2.

### 1.1 Create a Key Pair (for SSH access)
1. AWS Console → **EC2 → Key Pairs → Create key pair**
2. Name: `devops-practice-key`
3. Type: `RSA`, Format: `.pem` (for Linux/Mac) or `.ppk` (for PuTTY on Windows)
4. Click **Create** — the private key file downloads automatically. **Save it securely; AWS will not let you download it again.**
5. Set correct permissions locally so SSH accepts it:
   ```bash
   chmod 400 devops-practice-key.pem
   ```

### 1.2 Launch the EC2 Instance (AMI)
1. AWS Console → **EC2 → Launch Instance**
2. **Name:** `devops-practice-server`
3. **AMI (Amazon Machine Image):** choose `Amazon Linux 2023` or `Ubuntu 22.04 LTS` (free-tier eligible)
4. **Instance type:** `t2.micro` (free-tier eligible)
5. **Key pair:** select the key pair created in Step 1.1
6. **Network settings:** allow inbound `SSH (22)` from your IP, and `HTTP (80)` / `HTTPS (443)` if the instance will serve web traffic
7. Click **Launch Instance**

### 1.3 Connect to the Instance
```bash
ssh -i devops-practice-key.pem ec2-user@<EC2_PUBLIC_IP>
```

---

## 2. Create IAM User & Permissions

This IAM user is what **GitHub Actions** will use to authenticate with AWS and upload files to S3. Never use your AWS root account for this.

### 2.1 Create the User
1. AWS Console → **IAM → Users → Create user**
2. Username: `github-actions-deployer`
3. **Do not** enable AWS Management Console access (this user only needs programmatic/API access)

### 2.2 Attach a Scoped Permission Policy
Rather than attaching a broad policy like `AmazonS3FullAccess`, create an inline policy scoped to only your bucket — this follows the **principle of least privilege**, a core DevOps/security practice.

**IAM → Users → github-actions-deployer → Add permissions → Create inline policy → JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3DeployAccess",
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
Replace `YOUR_BUCKET_NAME` with your actual bucket name. Name the policy `S3DeployPolicy` and save.

### 2.3 Generate an Access Key
1. Go to the user → **Security credentials** tab → **Create access key**
2. Use case: **"Application running outside AWS"**
3. Copy the **Access Key ID** and **Secret Access Key** — the secret is shown only once. Store both temporarily; they'll go into GitHub Secrets in Step 7.

---

## 3. Create S3 Bucket

1. AWS Console → **S3 → Create bucket**
2. **Bucket name:** must be globally unique, e.g. `my-devops-practice-site-2026`
3. **Region:** choose one close to you (e.g. `ap-southeast-1`)
4. **Block Public Access settings:** **uncheck** "Block all public access" (a static website bucket must be publicly readable) — acknowledge the warning checkbox
5. Leave other settings as default → **Create bucket**

### 3.1 Add a Bucket Policy (Public Read Access)
Go to the bucket → **Permissions → Bucket Policy** and add:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

> This makes every object in the bucket publicly readable — correct for a public static website, but never do this on a bucket holding private/sensitive data.

---

## 4. Configure AWS CLI

Installing and configuring the AWS CLI lets you manage AWS from your terminal — useful for testing uploads manually before automating them in GitHub Actions.

### 4.1 Install
```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Verify
aws --version
```

### 4.2 Configure Credentials
```bash
aws configure
```
You'll be prompted for:
```
AWS Access Key ID [None]: <your access key from Step 2.3>
AWS Secret Access Key [None]: <your secret key from Step 2.3>
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

This saves credentials to `~/.aws/credentials` and config to `~/.aws/config`. Test it:
```bash
aws s3 ls
```
If it lists your bucket(s) without error, the CLI is correctly configured.

---

## 5. Configure Static Website Hosting on S3

1. Go to your bucket → **Properties** tab → scroll to **Static website hosting** → **Edit**
2. Select **Enable**
3. **Hosting type:** "Host a static website"
4. **Index document:** `index.html`
5. **Error document:** `error.html` (optional but recommended)
6. Save — AWS gives you a **bucket website endpoint URL**, e.g.:
   ```
   http://YOUR_BUCKET_NAME.s3-website-ap-southeast-1.amazonaws.com
   ```
7. Upload a test `index.html` and open the endpoint URL to confirm it loads

---

## 6. Create GitHub Repository

1. Go to [github.com](https://github.com) → **New repository**
2. Name it, e.g. `s3-static-site-cicd`
3. Add your website files (`index.html`, `style.css`, etc.) to the repo, or push an existing local project:
   ```bash
   git init
   git add .
   git commit -m "Initial website files"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git push -u origin main
   ```

---

## 7. Set AWS Credentials as GitHub Secrets

GitHub Actions needs the IAM user's credentials to authenticate with AWS — but they must **never** be hardcoded in the workflow file. Store them as encrypted repository secrets instead.

**Where:** Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value | Source |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Access key | Step 2.3 |
| `AWS_SECRET_ACCESS_KEY` | Secret key | Step 2.3 |
| `AWS_REGION` | e.g. `ap-southeast-1` | Step 3 |
| `S3_BUCKET_NAME` | Your bucket name | Step 3 |

These are encrypted at rest by GitHub and only exposed to workflow runs as environment variables — never visible in logs (GitHub automatically masks them).

---

## 8. Write the GitHub Actions Workflow (Explained)

Create the file `.github/workflows/deploy.yml` in your repo:

```yaml
name: Deploy to S3

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
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
            --exclude ".github/*"
```

### Line-by-line explanation

| Section | What it does |
|---|---|
| `name: Deploy to S3` | Display name of the workflow, shown in the GitHub Actions tab |
| `on: push: branches: [main]` | Trigger condition — this workflow runs automatically every time code is pushed to `main` |
| `jobs: deploy:` | Defines a job named `deploy` (you can have multiple jobs; this workflow has one) |
| `runs-on: ubuntu-latest` | GitHub spins up a fresh Ubuntu virtual machine to run the job's steps |
| **Step 1** `actions/checkout@v4` | Official GitHub action that clones your repo's code onto the runner, so later steps can access the files |
| **Step 2** `aws-actions/configure-aws-credentials@v4` | Official AWS action that reads the three secrets and configures the AWS CLI on the runner, so subsequent `aws` commands are authenticated |
| **Step 3** `aws s3 sync . s3://...` | The actual deployment command — see below |

### Breaking down the `aws s3 sync` command
```bash
aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} --delete --exclude ".git/*" --exclude ".github/*"
```
- `aws s3 sync . s3://BUCKET` — uploads only **new or changed** files from the current directory (`.`) to the bucket (much faster than re-uploading everything every time)
- `--delete` — removes files from the S3 bucket that no longer exist in the repo, keeping the bucket an exact mirror of your repo
- `--exclude ".git/*"` — don't upload git's internal metadata folder
- `--exclude ".github/*"` — don't upload the workflow files themselves to the website bucket

> If your site's files live in a subfolder (e.g. a build output folder like `dist/` or `build/`), change `.` to that path: `aws s3 sync ./dist s3://...`

### Verifying it works
1. Commit and push the workflow file:
   ```bash
   git add .github/workflows/deploy.yml
   git commit -m "Add CI/CD workflow for S3 deployment"
   git push origin main
   ```
2. Go to the repo's **Actions** tab — you'll see the workflow running (yellow), then passing (green) or failing (red)
3. Visit your S3 website endpoint URL to confirm the update appeared

---

## 9. Troubleshooting

| Problem | Likely Cause |
|---|---|
| `Access Denied` during sync | IAM policy doesn't include the correct bucket ARN, or bucket name typo in secrets |
| Workflow doesn't trigger | Pushed to a branch other than `main`, or workflow file isn't in `.github/workflows/` |
| Website shows 403 | Bucket policy not applied, or "Block Public Access" still enabled |
| Website shows old content | Browser cache — hard refresh, or add CloudFront + cache invalidation |

_(add real issues here as you hit them)_

---

## 10. Next Steps

- [ ] Replace IAM access keys with **OIDC** (GitHub ↔ AWS federated auth — no long-lived secrets stored at all)
- [ ] Add **CloudFront** in front of the S3 bucket for HTTPS + CDN caching, with a cache-invalidation step in the workflow
- [ ] Add a **staging** branch/environment that deploys to a separate test bucket before merging to `main`
- [ ] Explore containerized deployment: Docker + ECS/EKS
- [ ] Learn **Terraform** to provision the IAM user, S3 bucket, and bucket policy as code instead of manual console clicks

---

## 📌 Project 2: _(next project goes here)_
