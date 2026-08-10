# AWS S3 Static Website CI/CD with GitHub Actions

This documentation provides a complete, step-by-step guide to hosting a static website on Amazon S3 and automating the deployment process (CI/CD) using GitHub Actions. 

Since S3 static hosting is serverless, no virtual machines (EC2/AMIs) are required.

---

## Phase 1: AWS Infrastructure Setup

### 1. Create and Configure the S3 Bucket
First, we need a place to store the website files and serve them to the internet.
1. Log into the AWS Management Console and navigate to **S3**.
2. Click **Create bucket**.
3. **Bucket name:** Enter a globally unique name (e.g., `my-awesome-static-site-123`).
4. **Region:** Choose the region closest to your users.
5. **Object Ownership:** Leave as *ACLs disabled*.
6. **Block Public Access settings:** **Uncheck** "Block all public access" and acknowledge the warning. (Your website needs to be public for people to see it).
7. Click **Create bucket**.

### 2. Enable Static Website Hosting
Now, tell S3 to act as a web server.
1. Click on your newly created bucket and go to the **Properties** tab.
2. Scroll to the very bottom to **Static website hosting** and click **Edit**.
3. Select **Enable**.
4. **Index document:** Type `index.html`.
5. **Error document:** Type `error.html` (or `index.html` if using a Single Page Application like React/Vue).
6. Click **Save changes**. 
*(Note your newly generated **Bucket website endpoint** at the bottom of the Properties tab—this is your website URL).*

### 3. Apply a Public Bucket Policy
Allow the internet to read your website files.
1. Go to the **Permissions** tab of your bucket.
2. Scroll to **Bucket policy** and click **Edit**.
3. Paste the following JSON (replace `YOUR-BUCKET-NAME` with your actual bucket name):
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "PublicReadGetObject",
               "Effect": "Allow",
               "Principal": "*",
               "Action": "s3:GetObject",
               "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
           }
       ]
   }
