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

All of the above is defined as one CloudFormation stack in
[`infra/cloudformation.yml`](infra/cloudformation.yml).

## Provisioning (one-time)

Deploy the stack in `us-east-1` (CloudFront requires its ACM cert there). It issues the
TLS certificate, builds the CloudFront distribution + private bucket, creates the GitHub
OIDC deploy role, and (optionally) the apex/`www` Route 53 records.

> **Pick the right hosted zone.** The account has more than one zone named
> `morex.network`; the live one is the one whose nameservers match the registrar's. As of
> this writing that is `Z094632629RI4KZOJOQKT`. Confirm with:
> `aws route53 get-hosted-zone --id <zoneId> --query DelegationSet.NameServers` and compare
> against `dig +short morex.network NS`.

The domain was already serving another app, so provisioning is done in two phases to avoid
downtime. **Phase A** builds everything with DNS left alone (`ManageApexDns=false`):

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name morex-site \
  --template-file infra/cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides HostedZoneId=Z094632629RI4KZOJOQKT \
    CreateOIDCProvider=true ManageApexDns=false
```

Read the outputs, store them as repo secrets, and push so the site lands in S3 / CloudFront:

```bash
aws cloudformation describe-stacks --region us-east-1 --stack-name morex-site \
  --query 'Stacks[0].Outputs'

gh secret set AWS_DEPLOY_ROLE_ARN        -b <DeployRoleArn>
gh secret set S3_BUCKET                  -b <BucketName>
gh secret set CLOUDFRONT_DISTRIBUTION_ID -b <DistributionId>
```

Once the site serves on the CloudFront URL, **Phase B** cuts the domain over by re-running
the deploy with `ManageApexDns=true` (this replaces the apex + `www` `A`/`AAAA` records with
CloudFront aliases; `MX`/DKIM email records are never touched).
