# 🚀 Static Website Deployment to AWS S3 using GitHub Actions

This project demonstrates how to deploy a static website to **Amazon S3** and automate the deployment process using **GitHub Actions CI/CD**.

The goal of this project is to learn the basic AWS and DevOps workflow:

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
    ├── Configure AWS credentials
    └── Sync files to S3
    │
    ▼
Amazon S3
    │
    ▼
Static Website
