# ModelVerse

ModelVerse is a serverless, mobile-first Progressive Web Application (PWA) that provides a unified playground for experimenting with AWS Bedrock AI models on-the-go. Zero server costs - you only pay for what you use on AWS.

## Features

- 🚀 Mobile-first design optimized for on-the-go AI experimentation
- 💻 Installable as a PWA on any device
- 🔄 Quick switching between different AWS Bedrock models
- 💰 No server costs - pay only for AWS token usage
- 📱 100% client-side application
- 💾 Local data storage using IndexedDB
- 🔒 Secure local credential storage
- 🌐 Works offline (except for API calls)
- 🎯 Enterprise-grade AI models in your pocket

## Technical Stack

- 🔧 Pure client-side JavaScript/TypeScript
- 📦 IndexedDB for local storage
- 🔐 Local credential encryption
- 🌐 AWS SDK for JavaScript
- 📱 Progressive Web App (PWA)

## Prerequisites

- AWS Account with Bedrock access
- AWS Access Key ID and Secret Access Key
- Modern web browser that supports:
  - IndexedDB
  - PWA installation
  - Service Workers

# ModelVerse Distribution Guide

This guide explains how to host the ModelVerse distribution folder (`/dist`) using different hosting services.

## 1. AWS S3 + CloudFront
### 1. Create S3 Bucket
```bash
aws s3 mb s3://your-modelverse-bucket --region your-region
```

### 2. Keep S3 Bucket Private
- Do NOT enable static website hosting
- Do NOT add public bucket policy

### 3. Create CloudFront Origin Access Identity (OAI)
```bash
# Create OAI
aws cloudfront create-cloud-front-origin-access-identity \
    --cloud-front-origin-access-identity-config \
    CallerReference="modelverse-oai",Comment="ModelVerse OAI"
```

### 4. Configure S3 Bucket Policy
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
            "Resource": "arn:aws:s3:::your-modelverse-bucket/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
                }
            }
        }
    ]
}
```

### 5. Create CloudFront Distribution
```bash
# Create distribution using AWS CLI
aws cloudfront create-distribution \
    --origin-domain-name your-modelverse-bucket.s3.amazonaws.com \
    --default-root-object index.html
```

Key CloudFront Settings:
```yaml
Distribution Settings:
  Origin:
    - Domain Name: your-modelverse-bucket.s3.amazonaws.com
    - Origin Path: (empty or /prod)
    - S3 Origin Config:
      - Origin Access Identity: Use OAI created above
  
  Behaviors:
    - Path Pattern: Default (*)
    - Viewer Protocol Policy: Redirect HTTP to HTTPS
    - Allowed HTTP Methods: GET, HEAD, OPTIONS
    - Cache Policy: CachingOptimized
    - Origin Request Policy: CORS-S3Origin
    
  Error Pages:
    - HTTP Error Code: 403
    - Response Page Path: /index.html
    - Response Code: 200
    
  Functions:
    - Viewer Response: Add security headers
```

### 6. Upload Distribution Files
```bash
aws s3 sync dist/ s3://your-modelverse-bucket
```

### 7. Security Headers Function
Create a CloudFront Function to add security headers:

```javascript
function handler(event) {
    var response = event.response;
    var headers = response.headers;

    headers['strict-transport-security'] = { 
        value: 'max-age=31536000; includeSubdomains; preload'
    };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    headers['x-xss-protection'] = { value: '1; mode=block' };
    headers['referrer-policy'] = { value: 'strict-origin-when-cross-origin' };
    headers['content-security-policy'] = {
        value: "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';"
    };

    return response;
}
```

### 8. Deployment Script
```bash
#!/bin/bash

# Variables
BUCKET="your-modelverse-bucket"
DISTRIBUTION_ID="your-distribution-id"

# Build application (if needed)
npm run build

# Sync with S3
aws s3 sync dist/ s3://$BUCKET --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
    --distribution-id $DISTRIBUTION_ID \
    --paths "/*"
```

### 9. Custom Domain Setup (Optional)
```bash
# Request SSL Certificate in ACM
aws acm request-certificate \
    --domain-name app.modelverse.com \
    --validation-method DNS \
    --region us-east-1

# Add certificate to CloudFront distribution
aws cloudfront update-distribution \
    --id $DISTRIBUTION_ID \
    --viewer-certificate "ACMCertificateArn=arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/YOUR_CERT_ID,SSLSupportMethod=sni-only"
```

### 10. Monitoring Setup
```bash
# Create CloudWatch Dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "ModelVerse-Distribution" \
    --dashboard-body file://dashboard.json
```

Example `dashboard.json`:
```json
{
    "widgets": [
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/CloudFront", "Requests", "DistributionId", "YOUR_DISTRIBUTION_ID"]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-east-1",
                "title": "Total Requests"
            }
        }
    ]
}
```

## 2. GitHub Pages

### Prerequisites
- GitHub account
- Git installed locally

### Setup Steps

1. **Create GitHub Repository**
- Create a new repository on GitHub
- Name it `modelverse` or `your-username.github.io`

2. **Configure GitHub Pages**
```bash
# Create a new branch for GitHub Pages
git checkout -b gh-pages

# Copy dist contents to root
cp -r dist/* .

# Add and commit files
git add .
git commit -m "Deploy to GitHub Pages"

# Push to GitHub
git push origin gh-pages
```

3. **Enable GitHub Pages**
- Go to repository settings
- Navigate to "Pages"
- Select `gh-pages` branch as source
- Save changes

### Deploy Updates
```bash
# Update dist files
cp -r dist/* .
git add .
git commit -m "Update deployment"
git push origin gh-pages
```

## 3. Firebase Hosting

### Prerequisites
- Firebase account
- Firebase CLI installed

### Setup Steps

1. **Initialize Firebase**
```bash
firebase login
firebase init hosting
```

2. **Configure firebase.json**
```json
{
  "hosting": {
    "public": "dist",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

3. **Deploy**
```bash
firebase deploy --only hosting
```

## Important Notes

- Always ensure HTTPS is enabled
- Configure proper cache headers
- Set up proper redirects for SPA
- Update DNS records if using custom domain

## Custom Domain Setup

### CloudFront
1. Add alternate domain name in CloudFront
2. Upload SSL certificate (ACM)
3. Update DNS with CNAME record

### GitHub Pages
1. Add custom domain in repository settings
2. Create CNAME file in repository
3. Update DNS settings

### Firebase
1. Add custom domain in Firebase Console
2. Verify domain ownership
3. Update DNS records