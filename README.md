# Static Website Deployment to AWS S3 Using GitHub Actions CI/CD

## 1. Overview

This document explains how to deploy a static website to **Amazon S3** and automate future deployments using **GitHub Actions**.

The final deployment architecture will be:

```text
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ Push to main
    ▼
GitHub Actions
    │
    ├── Checkout source code
    │
    ├── Configure AWS credentials
    │
    └── Sync website files
    ▼
Amazon S3
    │
    ▼
Static Website
```

After the CI/CD pipeline is configured, the normal deployment process becomes:

```text
1. Make changes to website
2. git add .
3. git commit
4. git push origin main
5. GitHub Actions starts automatically
6. Website files are uploaded to S3
7. Updated website becomes available
```

---

# 2. Prerequisites

Before starting, make sure you have:

* An AWS account
* A GitHub account
* A static website
* Git installed on your local machine
* AWS CLI installed if you want to manage AWS from the command line
* Basic knowledge of Git

A static website may contain files such as:

```text
my-website/
├── index.html
├── about.html
├── contact.html
├── css/
│   └── style.css
├── js/
│   └── app.js
└── images/
    └── logo.png
```

---

# 3. AWS Infrastructure Setup

Before creating the GitHub Actions workflow, we need to prepare AWS.

The main AWS components required for this project are:

```text
AWS
│
├── IAM
│
└── S3
```

EC2 is optional.

---

# 4. EC2 / AMI — Optional

## 4.1 What is EC2?

**Amazon EC2 (Elastic Compute Cloud)** provides virtual servers in AWS.

For example:

```text
EC2 Instance
    │
    ├── Ubuntu
    ├── Nginx
    ├── PHP
    └── Application
```

EC2 is commonly used for dynamic applications such as Laravel, Node.js, Java, etc.

However, our current project is a **static website hosted on S3**, so an EC2 instance is not required.

---

## 4.2 What is an AMI?

AMI stands for:

> Amazon Machine Image

An AMI is a template used to create an EC2 instance.

For example:

```text
AMI
 │
 │ Launch
 ▼
EC2 Instance
```

An AMI can contain:

* Operating system
* Installed software
* Configuration
* Application setup

For example:

```text
Ubuntu AMI
    ↓
Launch
    ↓
EC2 Instance
```

---

## 4.3 When do we need an EC2 instance?

For this S3 static website project:

```text
EC2 → Not required
AMI → Not required
```

You would need EC2 if you were deploying something like:

```text
Laravel Application
       ↓
EC2
       ↓
Nginx
       ↓
PHP-FPM
       ↓
MySQL / Redis
```

Therefore, we will continue without EC2.

---

## 4.4 EC2 SSH Key Pair

If you later create an EC2 instance, AWS may ask you to create/select a **Key Pair**.

For example:

```text
my-server.pem
```

This is used to connect to the EC2 server through SSH.

Example:

```bash
ssh -i my-server.pem ubuntu@SERVER_IP
```

The `.pem` file is sensitive and must never be committed to GitHub.

For example, never do:

```text
git add my-server.pem
git push
```

Instead, keep private keys outside your Git repository.

---

# 5. Create an IAM User

IAM stands for:

> Identity and Access Management

IAM controls:

* Who can access AWS
* What they can access
* What actions they can perform

For this project, we need an IAM identity that GitHub Actions can use to upload files to S3.

For example:

```text
github-actions-s3-deployer
```

---

## 5.1 Create the IAM User

Go to:

```text
AWS Console
    ↓
IAM
    ↓
Users
    ↓
Create user
```

Use a meaningful name:

```text
github-actions-s3-deployer
```

Do not give this user unnecessary permissions.

The principle should be:

> Give only the permissions required for the deployment.

This is called the **Principle of Least Privilege**.

---

# 6. Give S3 Permissions to the IAM User

The GitHub Actions deployment needs to perform operations such as:

```text
List bucket
Upload files
Delete old files
```

A deployment using:

```bash
aws s3 sync
```

may require permissions such as:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
```

Ideally, these permissions should be restricted to the specific S3 bucket used by the website.

For example:

```text
my-static-website
```

rather than giving unrestricted access to all S3 buckets.

---

## 6.1 Example IAM Policy

An example bucket-specific policy could look like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::my-static-website"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-static-website/*"
    }
  ]
}
```

Replace:

```text
my-static-website
```

with your actual bucket name.

> This policy is specifically for deployment access. It does not give the IAM user full administrative access to AWS.

---

# 7. Create an S3 Bucket

Now create the S3 bucket that will store the website.

Go to:

```text
AWS Console
    ↓
S3
    ↓
Create bucket
```

Choose a unique bucket name.

Example:

```text
my-static-website-production
```

Bucket names must be globally unique.

---

## 7.1 Choose the AWS Region

Select an appropriate AWS region.

For example:

```text
US East (N. Virginia)
us-east-1
```

or another region closer to your expected users.

Remember the region because GitHub Actions will need it later.

For example:

```yaml
aws-region: us-east-1
```

---

# 8. Configure S3 Bucket for Static Website Hosting

If you want S3 itself to serve the website, enable static website hosting.

Open:

```text
S3
 ↓
Your Bucket
 ↓
Properties
 ↓
Static website hosting
```

Enable:

```text
Static website hosting
```

Set:

```text
Index document:
index.html
```

If your website has an error page:

```text
Error document:
error.html
```

For a simple website, `index.html` may be enough.

---

# 9. Upload the Website Manually First

Before building CI/CD, it is a good idea to verify that the S3 website works manually.

Upload your website:

```text
index.html
css/
js/
images/
```

to the S3 bucket.

For example:

```text
S3 Bucket
├── index.html
├── about.html
├── css/
├── js/
└── images/
```

Verify that the website can be accessed successfully.

This is an important troubleshooting principle:

> First make sure the infrastructure works manually. Then automate it.

If the website does not work manually, adding CI/CD makes troubleshooting more difficult.

---

# 10. S3 Public Access and Security

Static website hosting may require the website objects to be publicly readable when using the S3 website endpoint directly.

However, public S3 access is not always the best production architecture.

For a production website, a better architecture is often:

```text
User
 ↓
CloudFront
 ↓
S3
```

CloudFront can serve the website while the S3 bucket remains private.

For this learning project, we can first understand:

```text
GitHub Actions
      ↓
S3
      ↓
Static Website
```

Then later improve it to:

```text
GitHub Actions
      ↓
S3
      ↓
CloudFront
      ↓
User
```

---

# 11. Install and Configure AWS CLI

AWS CLI stands for:

> AWS Command Line Interface

It allows you to manage AWS resources from your terminal.

Check whether AWS CLI is installed:

```bash
aws --version
```

If it is installed, you should see something similar to:

```text
aws-cli/2.x.x
```

If the command is not found, install AWS CLI according to your operating system.

---

# 12. Configure AWS CLI

After installation, run:

```bash
aws configure
```

AWS CLI will ask for:

```text
AWS Access Key ID:
AWS Secret Access Key:
Default region name:
Default output format:
```

Example:

```text
AWS Access Key ID: AKIA...
AWS Secret Access Key: ********
Default region name: us-east-1
Default output format: json
```

The credentials should belong to the IAM identity created for AWS access.

Do not use your AWS root account credentials.

---

# 13. Verify AWS CLI Access

Run:

```bash
aws sts get-caller-identity
```

If authentication is successful, AWS returns information about the current identity.

You can also test S3:

```bash
aws s3 ls
```

Or check your specific bucket:

```bash
aws s3 ls s3://my-static-website-production
```

If these commands work, your AWS CLI configuration is working.

---

# 14. Test Manual S3 Deployment

Before creating GitHub Actions, test the exact deployment command locally.

From your website directory:

```bash
aws s3 sync ./ s3://my-static-website-production
```

This means:

```text
Local website
     ↓
aws s3 sync
     ↓
S3 bucket
```

You should see files being uploaded.

For example:

```text
upload: ./index.html to s3://my-static-website-production/index.html
upload: ./css/style.css to s3://my-static-website-production/css/style.css
```

Once this works, we know:

```text
AWS
 ↓
IAM
 ↓
S3
 ↓
AWS CLI
 ↓
Website
```

is working.

Now we can automate it.

---

# 15. Create a GitHub Repository

Now create a GitHub repository for the website.

For example:

```text
my-static-website
```

Your project should look like:

```text
my-static-website/
├── index.html
├── about.html
├── css/
├── js/
└── images/
```

Initialize Git:

```bash
git init
```

Add the files:

```bash
git add .
```

Create the first commit:

```bash
git commit -m "Initial website"
```

Create the main branch:

```bash
git branch -M main
```

Add the GitHub repository:

```bash
git remote add origin https://github.com/YOUR_USERNAME/my-static-website.git
```

Push:

```bash
git push -u origin main
```

Now your website source code is stored in GitHub.

---

# 16. Create GitHub Actions Secrets

Now we need to allow GitHub Actions to authenticate with AWS.

Go to:

```text
GitHub Repository
    ↓
Settings
    ↓
Secrets and variables
    ↓
Actions
    ↓
New repository secret
```

Create:

```text
AWS_ACCESS_KEY_ID
```

and set its value to the AWS access key created for the IAM deployment user.

Then create:

```text
AWS_SECRET_ACCESS_KEY
```

and set its value to the corresponding AWS secret access key.

You can also create:

```text
AWS_REGION
```

Example:

```text
AWS_REGION = us-east-1
```

And optionally:

```text
AWS_S3_BUCKET
```

Example:

```text
AWS_S3_BUCKET = my-static-website-production
```

---

# 17. Why GitHub Secrets Are Required

Never put credentials directly into your workflow.

Do NOT do this:

```yaml
aws-access-key-id: AKIA123456
aws-secret-access-key: abcdef123456
```

Instead:

```yaml
aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

The actual credentials remain stored securely in GitHub Secrets.

The workflow only references them.

---

# 18. Create the GitHub Actions Workflow

Create this directory:

```text
.github/
```

Inside it:

```text
.github/workflows/
```

Then create:

```text
deploy.yml
```

Your project becomes:

```text
my-static-website/
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── index.html
├── about.html
├── css/
├── js/
└── images/
```

---

# 19. Complete GitHub Actions Workflow

Use:

```yaml
name: Deploy to AWS S3

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

      # Step 1: Checkout source code
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Configure AWS credentials
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      # Step 3: Deploy website to S3
      - name: Sync files to S3
        run: |
          aws s3 sync ./ s3://${{ secrets.AWS_S3_BUCKET }} --delete
```

---

# 20. Explanation of the Workflow

## 20.1 `name`

```yaml
name: Deploy to AWS S3
```

This is the name displayed in GitHub Actions.

You will see:

```text
Deploy to AWS S3
```

inside:

```text
GitHub
 ↓
Actions
```

---

## 20.2 `on`

```yaml
on:
```

This defines **when the workflow should run**.

We are using:

```yaml
push:
```

which means the workflow runs when code is pushed.

---

## 20.3 `branches`

```yaml
branches:
  - main
```

This means the workflow only runs when code is pushed to:

```text
main
```

For example:

```bash
git push origin main
```

will trigger the workflow.

But:

```bash
git push origin development
```

will not trigger this workflow.

This is useful because the `main` branch can represent the production version of the website.

---

# 21. `jobs`

```yaml
jobs:
```

A workflow can contain multiple jobs.

For example:

```text
test
build
deploy
```

Our workflow currently has one job:

```yaml
jobs:
  deploy:
```

The job is responsible for deploying the website.

---

# 22. `runs-on`

```yaml
runs-on: ubuntu-latest
```

GitHub provides a temporary Ubuntu runner to execute the workflow.

Conceptually:

```text
GitHub
   ↓
Temporary Ubuntu machine
   ↓
Run deployment
   ↓
Machine removed
```

You don't need to create an EC2 instance for GitHub Actions.

---

# 23. Checkout Step

```yaml
- name: Checkout code
  uses: actions/checkout@v4
```

This downloads the repository source code onto the GitHub runner.

Before checkout:

```text
Ubuntu Runner
└── No project files
```

After checkout:

```text
Ubuntu Runner
├── index.html
├── about.html
├── css/
├── js/
└── images/
```

The next steps can now work with these files.

---

# 24. Configure AWS Credentials

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
```

This configures AWS authentication on the GitHub runner.

The credentials are provided through:

```yaml
with:
  aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
  aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  aws-region: ${{ secrets.AWS_REGION }}
```

The important part is:

```text
secrets.AWS_ACCESS_KEY_ID
```

GitHub retrieves the value stored under:

```text
AWS_ACCESS_KEY_ID
```

Similarly:

```text
secrets.AWS_SECRET_ACCESS_KEY
```

retrieves:

```text
AWS_SECRET_ACCESS_KEY
```

---

# 25. Deploy to S3

The final step is:

```yaml
- name: Sync files to S3
  run: |
    aws s3 sync ./ s3://${{ secrets.AWS_S3_BUCKET }} --delete
```

`run` means:

> Execute this command on the GitHub runner.

The command:

```bash
aws s3 sync
```

synchronizes files between the runner and S3.

The source:

```text
./
```

means:

> Current directory.

The destination:

```text
s3://${{ secrets.AWS_S3_BUCKET }}
```

means:

> The S3 bucket whose name is stored in GitHub Secrets.

---

# 26. Understanding `--delete`

The `--delete` option removes files from S3 that no longer exist in the GitHub repository.

Example:

GitHub:

```text
index.html
about.html
```

S3:

```text
index.html
about.html
old.html
```

After:

```bash
aws s3 sync ./ s3://bucket --delete
```

S3 becomes:

```text
index.html
about.html
```

`old.html` is removed.

This keeps the S3 bucket synchronized with the repository.

---

# 27. Complete CI/CD Flow

At this point, the complete process is:

```text
                 Developer
                     │
                     │ git push
                     ▼
              GitHub Repository
                     │
                     │ push to main
                     ▼
              GitHub Actions
                     │
                     ▼
              Ubuntu Runner
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     Checkout Code        AWS Credentials
          │                     │
          └──────────┬──────────┘
                     ▼
                 AWS CLI
                     │
                     ▼
                S3 Bucket
                     │
                     ▼
              Static Website
```

---

# 28. Test the CI/CD Pipeline

Change your website.

For example:

```html
<h1>Welcome to My Website</h1>
```

Change it to:

```html
<h1>Welcome to My DevOps Website</h1>
```

Then run:

```bash
git add .
```

Commit:

```bash
git commit -m "Update website homepage"
```

Push:

```bash
git push origin main
```

GitHub Actions should automatically start.

Go to:

```text
GitHub Repository
    ↓
Actions
```

You should see:

```text
Deploy to AWS S3
       ↓
    Running
       ↓
    Success
```

If successful:

```text
GitHub
   ↓
GitHub Actions
   ↓
S3
```

has automatically deployed the latest version.

---

# 29. Troubleshooting

## Workflow does not start

Check that the code was pushed to:

```text
main
```

The workflow contains:

```yaml
on:
  push:
    branches:
      - main
```

---

## AWS authentication fails

Check:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
```

in:

```text
GitHub
 ↓
Settings
 ↓
Secrets and variables
 ↓
Actions
```

Also verify that the IAM credentials are active.

---

## S3 AccessDenied error

The IAM identity probably does not have the required S3 permissions.

Check the IAM policy.

The deployment generally needs permissions such as:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
```

---

## Website doesn't update

Check:

1. GitHub Actions completed successfully.
2. Files exist in S3.
3. S3 website configuration is correct.
4. Browser cache is not showing an old version.
5. If CloudFront is later added, check CloudFront caching.

---

# 30. Important Security Rules

Never commit AWS credentials.

Never commit:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Never commit:

```text
*.pem
```

Never put credentials directly into:

```yaml
deploy.yml
```

Use:

```text
GitHub Secrets
```

instead.

Also avoid using the AWS root account for application deployment.

Use a dedicated IAM identity with only the required permissions.

---

# 31. Recommended Production Improvement: GitHub OIDC

The workflow above uses:

```text
AWS Access Key
+
AWS Secret Access Key
```

This is easy to understand and is suitable for learning the basic CI/CD process.

However, for a production environment, a better architecture is:

```text
GitHub Actions
      │
      ▼
GitHub OIDC
      │
      ▼
AWS IAM Role
      │
      ▼
Temporary AWS Credentials
      │
      ▼
S3
```

This eliminates the need for long-lived AWS access keys stored in GitHub Secrets.

After understanding the basic pipeline, the next step should be to replace:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

with:

```text
GitHub OIDC
AWS IAM Role
```

---

# 32. Recommended Final Architecture

For learning:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ▼
AWS IAM
    │
    ▼
S3
    │
    ▼
Static Website
```

For a more production-oriented architecture:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Test
    ├── Build
    └── Deploy
          │
          ▼
     GitHub OIDC
          │
          ▼
      AWS IAM Role
          │
          ▼
         S3
          │
          ▼
      CloudFront
          │
          ▼
         Users
```

---

# 33. Final Step-by-Step Checklist

Use this checklist while building the project:

### AWS Setup

```text
[ ] Create AWS account
[ ] Understand EC2/AMI (optional for this project)
[ ] Create IAM deployment user
[ ] Create S3 deployment policy
[ ] Attach policy to IAM user
[ ] Create S3 bucket
[ ] Select bucket region
[ ] Configure static website hosting
[ ] Configure required S3 access
[ ] Upload website manually
[ ] Verify website works
```

### AWS CLI

```text
[ ] Install AWS CLI
[ ] Run aws configure
[ ] Configure IAM credentials
[ ] Run aws sts get-caller-identity
[ ] Run aws s3 ls
[ ] Test aws s3 sync manually
```

### GitHub

```text
[ ] Create GitHub repository
[ ] Push website source code
[ ] Create .github/workflows directory
[ ] Create deploy.yml
```

### GitHub Secrets

```text
[ ] AWS_ACCESS_KEY_ID
[ ] AWS_SECRET_ACCESS_KEY
[ ] AWS_REGION
[ ] AWS_S3_BUCKET
```

### GitHub Actions

```text
[ ] Configure push trigger
[ ] Configure main branch
[ ] Configure Ubuntu runner
[ ] Checkout repository
[ ] Configure AWS credentials
[ ] Run aws s3 sync
[ ] Test deployment
[ ] Verify website
```

### Next DevOps Improvements

```text
[ ] Add CI tests
[ ] Add build process
[ ] Replace AWS keys with GitHub OIDC
[ ] Add CloudFront
[ ] Add custom domain
[ ] Add HTTPS
[ ] Learn Terraform
[ ] Infrastructure as Code
```

---

# 34. Key Concepts You Should Understand

After completing this project, you should be able to explain these concepts:

| Concept                   | Purpose                                      |
| ------------------------- | -------------------------------------------- |
| Git                       | Version control                              |
| GitHub                    | Source code repository                       |
| GitHub Actions            | CI/CD automation                             |
| Runner                    | Machine that executes workflow               |
| IAM                       | AWS access control                           |
| IAM User                  | AWS identity                                 |
| IAM Role                  | AWS identity used by services/workflows      |
| Access Key                | Programmatic AWS authentication              |
| Secret Key                | Private part of AWS credentials              |
| S3                        | Object storage                               |
| S3 Static Website Hosting | Hosts static websites                        |
| AWS CLI                   | Manage AWS from terminal                     |
| `aws s3 sync`             | Synchronize local files with S3              |
| `--delete`                | Remove obsolete S3 files                     |
| OIDC                      | Secure authentication between GitHub and AWS |
| CloudFront                | CDN for serving website globally             |
| AMI                       | Template for creating EC2 instances          |
| EC2                       | Virtual server in AWS                        |

---

# Conclusion

The fundamental idea of this project is simple:

```text
Manual Deployment:

Developer
   ↓
Upload files manually
   ↓
S3
```

After CI/CD:

```text
Automated Deployment:

Developer
   ↓
git push
   ↓
GitHub
   ↓
GitHub Actions
   ↓
AWS Authentication
   ↓
S3
   ↓
Website Updated
```

The important DevOps principle is that **you first build and verify the infrastructure manually, then automate the repeatable deployment process**.

For this particular static website project, EC2/AMI is not required. S3 is the actual hosting service, IAM provides controlled AWS access, AWS CLI lets you test deployment manually, and GitHub Actions automates the deployment.
