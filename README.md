# DevOps Learning Notes

Personal reference documentation for my DevOps practice on AWS.

---

## 📌 Project 1: Static Website on S3 with GitHub Actions CI/CD

**Goal:** Auto-deploy a static website to S3 whenever code is pushed to GitHub.

### Architecture
```
GitHub repo (push to main) → GitHub Actions workflow → AWS S3 bucket (static website hosting)
```

### Prerequisites
- AWS account
- S3 bucket already created and configured for static website hosting
- GitHub repository with your website files

---

### Step 1: Create an IAM user for GitHub Actions

1. AWS Console → **IAM → Users → Create user**
2. Name it something like `github-actions-deployer`
3. Attach an **inline policy** scoped only to your bucket (don't use full S3 access):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
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

4. Go to the user → **Security credentials → Create access key** → choose "Application running outside AWS"
5. Save the **Access Key ID** and **Secret Access Key** somewhere safe (you won't see the secret again)

> ⚠️ This is the "quick start" method using long-lived keys. The more secure, production-grade approach is **OIDC** (GitHub authenticates to AWS without any stored secret keys). Worth learning next — see "Next Steps" below.

---

### Step 2: Add secrets to the GitHub repo

Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | from Step 1 |
| `AWS_SECRET_ACCESS_KEY` | from Step 1 |
| `AWS_REGION` | e.g. `ap-southeast-1` |
| `S3_BUCKET_NAME` | your bucket name |

---

### Step 3: Create the workflow file

Create `.github/workflows/deploy.yml` in the repo:

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

> If website files live in a subfolder (e.g. `dist/`, `public/`, `build/`), change the sync source: `aws s3 sync ./dist s3://...`

---

### Step 4: Push and verify

```bash
git add .github/workflows/deploy.yml
git commit -m "Add S3 deploy workflow"
git push origin main
```

Check the **Actions** tab in GitHub for the run status. Then check the S3 website endpoint to confirm the update.

---

### Step 5 (optional): CloudFront cache invalidation

If a CloudFront distribution sits in front of the bucket, add this step after the sync so changes show up immediately instead of waiting for cache expiry:

```yaml
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

---

### Troubleshooting notes
_(fill in as I hit issues)_

- 
- 

---

## Next Steps / Things to Learn
- [ ] Switch from IAM access keys to OIDC (keyless GitHub → AWS auth)
- [ ] Add a CloudFront distribution in front of S3
- [ ] Add a staging branch/environment before production deploy
- [ ] Explore Docker + ECS/EKS deployment pipeline
- [ ] Learn Terraform/CloudFormation for infra-as-code instead of manual console setup

---

## 📌 Project 2: _(next project goes here)_
