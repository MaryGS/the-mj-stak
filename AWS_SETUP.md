# AWS Infrastructure Setup for Hugo Blog

This guide walks you through setting up AWS S3 and CloudFront to host your Hugo static site.

## Prerequisites

- AWS Account with appropriate permissions
- AWS CLI installed and configured
- GitHub repository with the Hugo blog

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   GitHub        │────▶│   S3 Bucket     │────▶│   CloudFront    │
│   Actions       │     │   (Static Site) │     │   (CDN + HTTPS) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
                                                 ┌─────────────────┐
                                                 │     Users       │
                                                 └─────────────────┘
```

## Step 1: Create S3 Bucket

### Via AWS Console

1. Go to **S3** in AWS Console
2. Click **Create bucket**
3. Configure:
   - **Bucket name**: `your-blog-bucket-name` (must be globally unique)
   - **Region**: Choose your preferred region (e.g., `us-east-1`)
   - **Block Public Access**: Keep **all blocked** (CloudFront will access it)
   - Leave other settings as default
4. Click **Create bucket**

### Via AWS CLI

```bash
# Create the bucket
aws s3 mb s3://your-blog-bucket-name --region us-east-1

# Enable versioning (optional but recommended)
aws s3api put-bucket-versioning \
  --bucket your-blog-bucket-name \
  --versioning-configuration Status=Enabled
```

## Step 2: Create CloudFront Distribution

### Via AWS Console

1. Go to **CloudFront** in AWS Console
2. Click **Create distribution**
3. Configure Origin:
   - **Origin domain**: Select your S3 bucket from dropdown
   - **Origin access**: Select **Origin access control settings (recommended)**
   - Click **Create new OAC**
     - Name: `your-blog-oac`
     - Signing behavior: **Sign requests**
     - Click **Create**
4. Configure Default cache behavior:
   - **Viewer protocol policy**: **Redirect HTTP to HTTPS**
   - **Allowed HTTP methods**: **GET, HEAD**
   - **Cache policy**: **CachingOptimized**
5. Configure Settings:
   - **Default root object**: `index.html`
   - **Price class**: Choose based on your needs
   - **Standard logging**: Optional
6. Click **Create distribution**
7. **Important**: Copy the policy provided and update your S3 bucket policy (next step)

### Via AWS CLI

```bash
# Create Origin Access Control
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "your-blog-oac",
    "Description": "OAC for Hugo blog",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'

# Note the OAC ID from the output, then create the distribution
# (See AWS documentation for full CLI command)
```

## Step 3: Update S3 Bucket Policy

After creating the CloudFront distribution, update your S3 bucket policy to allow CloudFront access:

### Via AWS Console

1. Go to your S3 bucket
2. Click **Permissions** tab
3. Click **Edit** under **Bucket policy**
4. Add this policy (replace placeholders):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-blog-bucket-name/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
                }
            }
        }
    ]
}
```

### Via AWS CLI

```bash
aws s3api put-bucket-policy \
  --bucket your-blog-bucket-name \
  --policy file://bucket-policy.json
```

## Step 4: Configure Custom Error Pages (Optional)

For better 404 handling with Hugo:

1. Go to your CloudFront distribution
2. Click **Error pages** tab
3. Click **Create custom error response**
4. Configure:
   - **HTTP error code**: 403
   - **Customize error response**: Yes
   - **Response page path**: `/404.html`
   - **HTTP response code**: 404
5. Repeat for HTTP error code 404

## Step 5: Set Up GitHub OIDC Authentication (Recommended)

Using OIDC is more secure than storing AWS credentials as secrets.

### Create IAM Role for GitHub Actions

1. Go to **IAM** in AWS Console
2. Click **Identity providers** → **Add provider**
3. Configure:
   - **Provider type**: OpenID Connect
   - **Provider URL**: `https://token.actions.githubusercontent.com`
   - **Audience**: `sts.amazonaws.com`
4. Click **Add provider**

### Create IAM Role

1. Go to **IAM** → **Roles** → **Create role**
2. Select **Web identity**
3. Configure:
   - **Identity provider**: `token.actions.githubusercontent.com`
   - **Audience**: `sts.amazonaws.com`
4. Add condition:
   - **Key**: `token.actions.githubusercontent.com:sub`
   - **Condition**: `StringLike`
   - **Value**: `repo:YOUR_ORG/YOUR_REPO:*`
5. Click **Next**
6. Create and attach this inline policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3Deploy",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-blog-bucket-name",
                "arn:aws:s3:::your-blog-bucket-name/*"
            ]
        },
        {
            "Sid": "CloudFrontInvalidation",
            "Effect": "Allow",
            "Action": "cloudfront:CreateInvalidation",
            "Resource": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
    ]
}
```

7. Name the role: `github-actions-hugo-deploy`
8. Copy the **Role ARN**

## Step 6: Configure GitHub Secrets

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions**

### Required Secrets

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789:role/github-actions-hugo-deploy` | IAM role ARN for OIDC |
| `AWS_S3_BUCKET` | `your-blog-bucket-name` | S3 bucket name |
| `AWS_CLOUDFRONT_DISTRIBUTION_ID` | `E1234567890ABC` | CloudFront distribution ID |

### Optional Variables

Go to **Variables** tab:

| Variable Name | Value | Description |
|---------------|-------|-------------|
| `AWS_REGION` | `us-east-1` | AWS region |
| `SITE_URL` | `https://d1234.cloudfront.net` | Your site URL |

## Step 7: Test Deployment

1. Make a change to any file in the `site/` directory
2. Commit and push to `main` branch
3. Go to **Actions** tab in GitHub to monitor the workflow
4. Once complete, visit your CloudFront URL

## Alternative: Using IAM Access Keys

If you prefer not to use OIDC, you can use IAM access keys:

1. Create an IAM user with the same policy as above
2. Generate access keys
3. Add as GitHub secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
4. Modify `.github/workflows/deploy.yml`:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ vars.AWS_REGION || 'us-east-1' }}
```

## Custom Domain (Optional)

To use a custom domain:

1. **Request SSL Certificate**:
   - Go to **AWS Certificate Manager** (must be in `us-east-1` for CloudFront)
   - Request public certificate for your domain
   - Validate via DNS

2. **Update CloudFront**:
   - Edit distribution
   - Add **Alternate domain name (CNAME)**: `blog.yourdomain.com`
   - Select your SSL certificate
   - Save changes

3. **Update DNS**:
   - Create CNAME record pointing to CloudFront domain
   - Or use Route 53 alias record

4. **Update Hugo config**:
   ```toml
   baseURL = "https://blog.yourdomain.com/"
   ```

## Troubleshooting

### 403 Forbidden Errors

- Verify S3 bucket policy allows CloudFront
- Check OAC is correctly configured
- Ensure CloudFront distribution is deployed (Status: Enabled)

### Changes Not Appearing

- CloudFront caches content; invalidate cache after deployment
- Check GitHub Actions workflow completed successfully
- Verify S3 bucket has updated files

### HTTPS Issues

- Ensure CloudFront has "Redirect HTTP to HTTPS" enabled
- For custom domains, verify SSL certificate is issued and associated

## Cost Estimates

| Service | Estimated Monthly Cost |
|---------|----------------------|
| S3 Storage (1 GB) | ~$0.02 |
| S3 Requests | ~$0.01-0.05 |
| CloudFront (10 GB transfer) | ~$0.85 |
| **Total** | **~$1-2/month** |

*Costs vary by region and usage. First 12 months may qualify for AWS Free Tier.*

## Resources

- [Hugo Documentation](https://gohugo.io/documentation/)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

