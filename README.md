# morex-site

Static website for Morex, hosted on AWS (S3 + CloudFront + Route 53) with HTTPS via ACM.

## How it deploys

Every push to `main` triggers [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml), which:

1. Authenticates to AWS via GitHub OIDC (assumes a scoped IAM role — no secrets stored).
2. Syncs the site files to the S3 bucket (`aws s3 sync --delete`).
3. Invalidates the CloudFront cache so changes go live immediately.

To update the site, edit the HTML and `git push`. That's it.

## Local preview

```bash
python3 -m http.server 8000
# open http://localhost:8000
```

## Infrastructure

| Component  | Purpose                                  |
|------------|------------------------------------------|
| S3         | Stores the static files (private bucket) |
| CloudFront | CDN + HTTPS, only path to the bucket     |
| ACM        | TLS certificate (us-east-1)              |
| Route 53   | DNS: apex + www redirect                 |
