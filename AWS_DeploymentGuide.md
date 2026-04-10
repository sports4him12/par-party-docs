# GolfSync — AWS Deployment Guide

Step-by-step checklist for deploying the Dev and Prod stacks using AWS CDK.

**Last updated:** 2026-04-10

---

## Prerequisites

### 1. AWS CLI configured
```bash
aws configure
# Enter: Access Key ID, Secret Access Key, default region (us-east-1), output format (json)

# Verify it's working
aws sts get-caller-identity
```

### 2. Bootstrap CDK (one-time per account/region)
```bash
cd golfsync-cdk
npx cdk bootstrap aws://YOUR_ACCOUNT_ID/us-east-1
```
Find your account ID with: `aws sts get-caller-identity --query Account --output text`

### 3. Docker running
CDK builds container images from source (`golfsync-api`, `golfsync-web`) and pushes them to ECR. Docker must be running on your machine at deploy time.

> **Note:** Camunda BPM is embedded in-process inside `golfsync-api` — there is no separate Camunda ECS service or container to build.

---

## Dev Stack (`GolfSyncDevStack`)

### Step 1 — Set required env vars
```bash
export GOLFSYNC_DEV_ALARM_EMAIL=support@golfsync.io
```

### Step 2 — Deploy
```bash
cd golfsync-cdk
npx cdk deploy GolfSyncDevStack
```
Note the `AppUrl` output — this is the ALB DNS name (e.g. `http://GolfSync-Alb-xxxx.us-east-1.elb.amazonaws.com`).

> **CORS note:** The CDK automatically sets `GOLFSYNC_CORS_ALLOWED_ORIGIN` to the ALB DNS name, so API calls from the browser work without any manual step.

### Step 2.5 — Run Liquibase migrations (required before the API can serve requests)

The API container has Liquibase disabled (`SPRING_LIQUIBASE_ENABLED=false`) to prevent startup races. A separate Liquibase ECS task must run first to create the database schema.

1. Get the subnet and security group IDs from the dev VPC:
```bash
# Public subnet IDs (Liquibase task runs in the public subnet in dev)
aws ec2 describe-subnets \
  --filters "Name=tag:aws-cdk:subnet-name,Values=public" \
            "Name=tag:aws-cdk:subnet-type,Values=Public" \
  --query "Subnets[*].SubnetId" --output text

# API security group ID
aws ec2 describe-security-groups \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=GolfSyncDevStack" \
            "Name=group-name,Values=*ApiSg*" \
  --query "SecurityGroups[0].GroupId" --output text
```

2. Copy the `LiquibaseMigrateCommand` from the CDK output, fill in the subnet and SG IDs, then run it:
```bash
# Example (replace <subnet-id> and <sg-id> with values from step 1)
# The dev Liquibase task has SPRING_LIQUIBASE_CONTEXTS=dev baked in by CDK,
# so migration 024 (seeding admin/test/test2 accounts) runs automatically in dev
# and is skipped in prod.
aws ecs run-task \
  --cluster golfsync-dev \
  --task-definition <LiquibaseTaskDefArn> \
  --launch-type EC2 \
  --network-configuration 'awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<sg-id>]}' \
  --count 1 \
  --started-by pre-deploy-liquibase
```

3. Wait for completion — re-run until `lastStatus` is `STOPPED` and `containers[0].exitCode` is `0`:
```bash
aws ecs describe-tasks --cluster golfsync-dev --tasks <task-arn> \
  --query "tasks[0].{status:lastStatus,exit:containers[0].exitCode}"
```

> **Run this before every deployment that includes database migrations.** If skipped, the API will fail its health check and ECS will not bring it up.

### Step 3 — Update `appBaseUrl`
Open `golfsync-cdk/bin/golfsync-cdk.ts` and update the dev stack config:
```ts
appBaseUrl: 'http://<ALB-DNS-NAME-FROM-OUTPUT>',
```
Then redeploy: `npx cdk deploy GolfSyncDevStack`

This URL is embedded in email invite links and the CAN-SPAM unsubscribe footer, so it must point to where the app is actually running.

### Step 4 — Populate Gmail SMTP credentials
Generate a Gmail App Password: Google Account → Security → 2-Step Verification → App passwords

```bash
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/gmail-smtp-credentials \
  --secret-string '{"username":"you@gmail.com","password":"xxxx xxxx xxxx xxxx"}'
```

> **Support email:** `support@golfsync.io` — set via `cfg.supportEmail` in `bin/golfsync-cdk.ts` and injected as `SUPPORT_EMAIL` into the API container.

### Step 5 — Populate optional secrets

These enable additional features. The app starts without them; features are silently disabled.

```bash
# Anthropic / OpenAI (AI navigation + booking assistant)
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/openai-api-key \
  --secret-string 'sk-...'

# Serper.dev (tournament discovery — free tier: 2,500 queries/month)
# Get key at https://serper.dev
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/serper-api-key \
  --secret-string 'your-serper-api-key'

# Stripe test keys (billing paused until June 1, 2026 — set now to test payment flows)
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-secret-key \
  --secret-string 'sk_test_...'

aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-publishable-key \
  --secret-string 'pk_test_...'

# Stripe webhook secret — from `stripe listen --forward-to localhost:8080/api/payments/webhook`
# or from Stripe Dashboard → Webhooks after registering the ALB endpoint
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-webhook-secret \
  --secret-string 'whsec_...'
```

> **Auto-generated — no action needed:**
> `golfsync-dev/db-credentials` and `golfsync-dev/jwt-secret` are generated by CDK automatically.

---

## Prod Stack (`GolfSyncProdStack`)

### Step 1 — Create a Route53 hosted zone for `golfsync.com`
If the domain is not already in Route53, create a hosted zone:
- AWS Console → Route53 → Hosted Zones → Create hosted zone → `golfsync.com`
- Update your domain registrar's nameservers to match the NS records Route53 provides.

Find the hosted zone ID:
```bash
aws route53 list-hosted-zones --query "HostedZones[?Name=='golfsync.com.'].Id" --output text
# Returns something like: /hostedzone/Z1234ABCDE5678
# The ID is the part after /hostedzone/
```

### Step 2 — Request an ACM certificate for HTTPS
```bash
# Request a certificate covering the apex and www subdomain
aws acm request-certificate \
  --domain-name golfsync.com \
  --subject-alternative-names '*.golfsync.com' \
  --validation-method DNS \
  --region us-east-1

# Get the certificate ARN
aws acm list-certificates \
  --query "CertificateSummaryList[?DomainName=='golfsync.com'].CertificateArn" \
  --output text \
  --region us-east-1
```

ACM will provide CNAME records for DNS validation:
- AWS Console → Certificate Manager → the new cert → "Create records in Route53"

Wait for the certificate status to show **Issued** before deploying (usually 1–5 minutes after DNS propagation).

### Step 3 — Set env vars and deploy
```bash
export GOLFSYNC_HOSTED_ZONE_ID=Z07602281XXQ30UW6PUD4
export GOLFSYNC_CERT_ARN=arn:aws:acm:us-east-1:897253013130:certificate/a3fb0028-f9e0-432c-91e7-545f60509187
export GOLFSYNC_PROD_ALARM_EMAIL=support@golfsync.io

cd golfsync-cdk
npx cdk deploy GolfSyncProdStack
```

This creates:
- Route53 A alias records: `golfsync.com` and `www.golfsync.com` → ALB
- ALB HTTPS listener on port 443 (TLS terminated using the ACM cert)
- ALB HTTP listener on port 80 → permanent redirect to HTTPS
- SES email identity for `golfsync.com` (outputs DNS verification records)

### Step 4 — Verify SES domain ownership
After deploy, CDK outputs CNAME/TXT records for SES domain verification:
- AWS Console → Route53 → Hosted Zones → `golfsync.com` → Create record
- Or: AWS Console → SES → Verified Identities → `golfsync.com` → view the records

Wait for SES to show the domain as **Verified** before expecting emails to send.

### Step 5 — Populate SES SMTP credentials
Generate SES SMTP credentials: AWS Console → SES → SMTP Settings → Create SMTP credentials.

```bash
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/ses-smtp-credentials \
  --secret-string '{"username":"AKIAIOSFODNN7EXAMPLE","password":"<derived-smtp-password>"}'
```

### Step 6 — Populate optional secrets
```bash
# Anthropic / OpenAI (AI navigation + booking assistant)
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/openai-api-key \
  --secret-string 'sk-...'

# Stripe live keys — activate billing before June 1, 2026
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/stripe-secret-key \
  --secret-string 'sk_live_...'

aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/stripe-publishable-key \
  --secret-string 'pk_live_...'
```

> **Billing activation checklist (June 1, 2026):**
> 1. Populate Stripe live keys above
> 2. Register the prod webhook endpoint in Stripe dashboard → `https://golfsync.com/api/payments/webhook`
> 3. Set `STRIPE_ENABLED=true` in the ECS task environment (CDK stack or Secrets Manager)
> 4. Send pre-billing notification email to all users

> **Auto-generated — no action needed:**
> `golfsync-prod/db-credentials` and `golfsync-prod/jwt-secret` are generated by CDK automatically.

### Step 7 — Set the admin UI password

Migration `001` creates the `admin` account automatically. Migration `024` (which sets the dev test passwords) is **skipped in prod** — so you must set the admin password manually after the first deployment.

1. Get the RDS endpoint and DB password:
```bash
# RDS endpoint
aws rds describe-db-instances \
  --query "DBInstances[?DBInstanceIdentifier=='golfsync-prod'].Endpoint.Address" \
  --output text

# Admin DB password (the golfsync_admin MySQL user)
aws secretsmanager get-secret-value \
  --secret-id golfsync-prod/db-credentials \
  --query SecretString --output text | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['password'])"
```

2. Generate a bcrypt hash for your chosen admin password:
```bash
# Requires Apache httpd tools (brew install httpd on Mac)
htpasswd -bnBC 12 "" 'YourStrongPassword!' | tr -d ':\n'
```

3. Connect to RDS and update the admin user (run from inside the VPC — use ECS Exec on the API task or a bastion):
```bash
mysql -h <rds-endpoint> -u golfsync_admin -p'<db-password>' golfsync \
  -e "UPDATE users SET password_hash='<bcrypt-hash>' WHERE username='admin';"
```

> **Security:** Change the admin password immediately after first login. The admin account's email is `admin@golfsync.io` (set in migration 001) — update it to a real monitored inbox.

### Step 8 — Request SES production access
By default, SES accounts are in sandbox mode and can only send to verified addresses. To send to any address:
- AWS Console → SES → Account dashboard → Request production access
- Explain use case: transactional and marketing email for a golf social app; includes CAN-SPAM opt-out mechanism

---

## Quick Reference — All Secrets

| Secret Name | Who sets it | Purpose |
|---|---|---|
| `golfsync-dev/db-credentials` | CDK (auto) | RDS MySQL password |
| `golfsync-dev/jwt-secret` | CDK (auto) | JWT signing key |
| `golfsync-dev/gmail-smtp-credentials` | You (Step 4) | Gmail app password for dev email |
| `golfsync-dev/openai-api-key` | You (Step 5) | AI assistant API key |
| `golfsync-dev/serper-api-key` | You (Step 5) | Serper.dev key for tournament discovery |
| `golfsync-dev/stripe-secret-key` | You (Step 5) | Stripe test secret key |
| `golfsync-dev/stripe-publishable-key` | You (Step 5) | Stripe test publishable key |
| `golfsync-dev/stripe-webhook-secret` | You (Step 5) | Stripe webhook signing secret |
| `golfsync-prod/db-credentials` | CDK (auto) | RDS MySQL password |
| `golfsync-prod/jwt-secret` | CDK (auto) | JWT signing key |
| `golfsync-prod/ses-smtp-credentials` | You (Step 5) | SES SMTP credentials for prod email |
| `golfsync-prod/openai-api-key` | You (Step 6) | AI assistant API key |
| `golfsync-prod/stripe-secret-key` | You (Step 6) | Stripe live secret key |
| `golfsync-prod/stripe-publishable-key` | You (Step 6) | Stripe live publishable key |

---

## CDK Environment Variable Reference

| Variable | Stack | Purpose |
|---|---|---|
| `GOLFSYNC_DEV_ALARM_EMAIL` | Dev | CloudWatch alarm notification email |
| `GOLFSYNC_PROD_ALARM_EMAIL` | Prod | CloudWatch alarm notification email |
| `GOLFSYNC_HOSTED_ZONE_ID` | Prod | Route53 hosted zone ID for `golfsync.com` |
| `GOLFSYNC_CERT_ARN` | Prod | ACM certificate ARN for HTTPS |

---

## Tearing Down

```bash
# Dev (safe to destroy — no deletion protection)
cd golfsync-cdk
npx cdk destroy GolfSyncDevStack

# Prod (deletion protection is ON on RDS — must disable first)
# AWS Console → RDS → Modify instance → uncheck deletion protection → apply immediately
npx cdk destroy GolfSyncProdStack
```

---

## ECS Exec (for debugging live containers)

The API container (which embeds Camunda in-process) can be accessed via ECS Exec:

```bash
# Dev
aws ecs execute-command --cluster golfsync-dev \
  --task <task-id> --interactive --command "/bin/sh"

# Prod
aws ecs execute-command --cluster golfsync-prod \
  --task <task-id> --interactive --command "/bin/sh"
```

Find the task ID: AWS Console → ECS → Clusters → `golfsync-dev` or `golfsync-prod` → Tasks
