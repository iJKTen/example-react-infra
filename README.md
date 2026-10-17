# React SPA Infrastructure

CloudFormation template that deploys a React single-page application to AWS with S3 hosting, CloudFront CDN, CI/CD pipeline, Route 53 DNS, security headers, and client-side routing support.

## Prerequisites

- **Connection stack** deployed exporting `Example-GitHubConnectionArn` (series post 2)
- **Certificate stack** deployed in `us-east-1` exporting `Example-CertificateArn` (series post 3)
- **Git Sync role** from series post 4 with the permission additions from posts 7 through 9, plus post 17 for the artifact bucket cleanup Lambda

## Parameters

| Parameter | Description |
|---|---|
| `ProjectName` | Prefix used for naming all resources (e.g. `Example`) |
| `DomainName` | Fully qualified domain name (e.g. `app.example.net`) |
| `GitHubRepository` | GitHub repository in `owner/repo` format |
| `GitHubBranch` | Branch to track for CI/CD pipeline (default: `main`) |
| `HostedZoneId` | Route 53 hosted zone ID for the domain |
| `BuildOutputDirectory` | Build output directory for the SPA framework (default: `dist`) |

## What Gets Created

- S3 bucket for site files (private, versioned, 7-day noncurrent cleanup)
- CloudFront distribution with OAC, HTTP/2+3, TLS 1.2, and Brotli compression
- CloudFront Function (`cloudfront-js-2.0`, viewer-request) rewriting non-asset paths to `/index.html` for client-side routing
- S3 bucket policy scoped to the specific CloudFront distribution
- Route 53 A and AAAA alias records pointing the domain to CloudFront
- S3 artifact bucket for pipeline intermediates
- Lambda-backed custom resource that empties the artifact bucket on stack deletion
- CodePipeline with Source, Build, and Deploy stages
- CodeBuild build project (Node.js 20, `npm run build`)
- CodeBuild deploy project with two-tier S3 caching and CloudFront invalidation
- IAM roles for CodePipeline, CodeBuild, the deploy project, and the cleanup Lambda
- CloudFront response headers policy (HSTS, CSP with `connect-src 'self'`, X-Frame-Options, X-Content-Type-Options)
- Stack outputs for the distribution ID, distribution domain, bucket name, and site URL

## Deploy

The repo includes two files. `infra.yaml` is the CloudFormation template. `deployment-file.yaml` is the deployment file that Sync from Git reads to find the template and resolve parameter values.

1. Update the placeholder values in `deployment-file.yaml` with your project name, domain, repository, and hosted zone ID.
2. Push the repo to GitHub.
3. Follow the Sync from Git process in series post 16 to create the stack.

## Architecture

Uses two-tier caching. Hashed assets (JS, CSS, images) get `max-age=31536000,public,immutable` because new builds produce new filenames. The single `index.html` gets `max-age=0,no-cache,no-store,must-revalidate` so browsers always fetch the latest version. A CloudFront Function at viewer-request rewrites any non-asset path to `/index.html`, so React Router handles client-side routing without relying on S3 error responses. If your app calls an external API, add its origin to the CSP's `connect-src` directive.
