# GolfSync — AWS Deployment Guide

Step-by-step checklist for deploying the Dev and Prod stacks using AWS CDK.

**Last updated:** 2026-04-10

---

## Prerequisites

### 1. AWS CLI configured — IAM Identity Center (SSO)

GolfSync uses IAM Identity Center with separate workload accounts for Dev and Prod.
The management (root) account hosts IAM Identity Center only — no workload resources are deployed there.

**One-time SSO profile setup:**
```bash
# Dev workload account profile
aws configure sso --profile golfsync-dev
# SSO session name:  golfsync
# SSO start URL:     https://<your-org>.awsapps.com/start
# SSO region:        us-east-1
# Account:           <dev-account-id>      (select from the list)
# Permission set:    AdministratorAccess
# CLI default region: us-east-1
# CLI output format:  json

# Prod workload account profile
aws configure sso --profile golfsync-prod
# (same SSO session name/start URL — select the prod account this time)
```

**Daily workflow — log in before any CDK or AWS CLI command:**
```bash
aws sso login --profile golfsync-dev   # opens browser, grants 8-hour credentials
aws sso login --profile golfsync-prod  # separate session for prod
```

**Verify you're in the right account:**
```bash
aws sts get-caller-identity --profile golfsync-dev
aws sts get-caller-identity --profile golfsync-prod
```

**One-time: set account IDs and alarm email in `~/.zshrc`:**
```bash
# Paste this block, filling in your real account IDs
export GOLFSYNC_DEV_ACCOUNT_ID=<dev-account-id>
export GOLFSYNC_PROD_ACCOUNT_ID=<prod-account-id>
export GOLFSYNC_DEV_ALARM_EMAIL=support@golfsync.io
export GOLFSYNC_PROD_ALARM_EMAIL=support@golfsync.io
```

Or run this script to auto-populate account IDs from your SSO sessions:
```bash
aws sso login --profile golfsync-dev
aws sso login --profile golfsync-prod

DEV_ACCOUNT=$(aws sts get-caller-identity --profile golfsync-dev --query Account --output text)
PROD_ACCOUNT=$(aws sts get-caller-identity --profile golfsync-prod --query Account --output text)

grep -q "GOLFSYNC_DEV_ACCOUNT_ID"  ~/.zshrc || echo "\nexport GOLFSYNC_DEV_ACCOUNT_ID=$DEV_ACCOUNT"  >> ~/.zshrc
grep -q "GOLFSYNC_PROD_ACCOUNT_ID" ~/.zshrc || echo "export GOLFSYNC_PROD_ACCOUNT_ID=$PROD_ACCOUNT" >> ~/.zshrc
grep -q "GOLFSYNC_DEV_ALARM_EMAIL" ~/.zshrc || echo "export GOLFSYNC_DEV_ALARM_EMAIL=support@golfsync.io" >> ~/.zshrc
grep -q "GOLFSYNC_PROD_ALARM_EMAIL" ~/.zshrc || echo "export GOLFSYNC_PROD_ALARM_EMAIL=support@golfsync.io" >> ~/.zshrc
source ~/.zshrc
```

> **Why account IDs matter:** With `GOLFSYNC_DEV_ACCOUNT_ID` set, running `cdk deploy GolfSyncDevStack --profile golfsync-prod` will fail — CDK detects the account mismatch before touching anything.

### 2. Bootstrap CDK (one-time per workload account)

Each workload account needs CDK bootstrapped once. Do **not** bootstrap the management account.

```bash
cd golfsync-cdk

# Dev account
npx cdk bootstrap aws://$GOLFSYNC_DEV_ACCOUNT_ID/us-east-1 --profile golfsync-dev

# Prod account
npx cdk bootstrap aws://$GOLFSYNC_PROD_ACCOUNT_ID/us-east-1 --profile golfsync-prod
```

### 3. Docker running
CDK builds container images from source (`golfsync-api`, `golfsync-web`) and pushes them to ECR. Docker must be running on your machine at deploy time.

> **Note:** Camunda BPM is embedded in-process inside `golfsync-api` — there is no separate Camunda ECS service or container to build.

---

## Dev Stack (`GolfSyncDevStack`)

### Step 1 — Deploy
```bash
cd golfsync-cdk
npx cdk deploy GolfSyncDevStack --profile golfsync-dev
```
Note the `AppUrl` output — this is the ALB DNS name (e.g. `http://GolfSync-Alb-xxxx.us-east-1.elb.amazonaws.com`).

> **CORS:** CDK automatically sets `GOLFSYNC_CORS_ALLOWED_ORIGIN` to the ALB DNS name — no manual step needed.

### Step 2 — Run Liquibase migrations

The API container has Liquibase disabled (`SPRING_LIQUIBASE_ENABLED=false`) to prevent startup races. A separate Liquibase ECS task must run first to create the database schema.

> **Run this before every deployment that includes database migrations.** If skipped, the API will fail its health check and ECS will not bring it up.

1. Get the subnet and security group IDs from the dev VPC:
```bash
# Public subnet ID (Liquibase task runs in the public subnet in dev)
aws ec2 describe-subnets \
  --filters "Name=tag:aws-cdk:subnet-name,Values=public" \
            "Name=tag:aws-cdk:subnet-type,Values=Public" \
  --query "Subnets[0].SubnetId" --output text \
  --profile golfsync-dev

# API security group ID
aws ec2 describe-security-groups \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=GolfSyncDevStack" \
            "Name=group-name,Values=*ApiSg*" \
  --query "SecurityGroups[0].GroupId" --output text \
  --profile golfsync-dev
```

2. Copy the `LiquibaseMigrateCommand` from the CDK output, fill in the IDs, then run it:
```bash
# The dev task has SPRING_LIQUIBASE_CONTEXTS=dev baked in — seeds admin/test/test2 accounts.
aws ecs run-task \
  --cluster golfsync-dev \
  --task-definition <LiquibaseTaskDefArn> \
  --launch-type EC2 \
  --network-configuration 'awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<sg-id>]}' \
  --count 1 \
  --started-by pre-deploy-liquibase \
  --profile golfsync-dev
```

3. Wait for exit code 0:
```bash
aws ecs describe-tasks --cluster golfsync-dev --tasks <task-arn> \
  --query "tasks[0].{status:lastStatus,exit:containers[0].exitCode}" \
  --profile golfsync-dev
```

### Step 3 — Update `appBaseUrl`
Open `golfsync-cdk/bin/golfsync-cdk.ts` and update the dev stack config:
```ts
appBaseUrl: 'http://<ALB-DNS-NAME-FROM-OUTPUT>',
```
Then redeploy: `npx cdk deploy GolfSyncDevStack --profile golfsync-dev`

This URL is embedded in email invite links and the CAN-SPAM unsubscribe footer.

### Step 4 — Populate secrets

All secrets are created by CDK at deploy time with placeholder values. Replace them now.

```bash
# Gmail SMTP (required — emails fail without this)
# Generate app password: Google Account -> Security -> 2-Step Verification -> App passwords
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/gmail-smtp-credentials \
  --secret-string '{"username":"you@gmail.com","password":"xxxx xxxx xxxx xxxx"}' \
  --profile golfsync-dev

# OpenAI (optional — AI assistant disabled if missing)
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/openai-api-key \
  --secret-string 'sk-...' \
  --profile golfsync-dev

# Serper.dev (optional — tournament discovery disabled if missing)
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/serper-api-key \
  --secret-string 'your-serper-api-key' \
  --profile golfsync-dev

# Stripe test keys (optional — set now to test payment flows before June 1)
aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-secret-key \
  --secret-string 'sk_test_...' \
  --profile golfsync-dev

aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-publishable-key \
  --secret-string 'pk_test_...' \
  --profile golfsync-dev

aws secretsmanager put-secret-value \
  --secret-id golfsync-dev/stripe-webhook-secret \
  --secret-string 'whsec_...' \
  --profile golfsync-dev
```

> **Auto-generated — no action needed:**
> `golfsync-dev/db-credentials` and `golfsync-dev/jwt-secret` are generated by CDK automatically.

### Dev accounts after first deploy

Migration `024` runs automatically in dev (seeded via `SPRING_LIQUIBASE_CONTEXTS=dev`):

| Username | Password | Role |
|---|---|---|
| `admin` | `GolfSync1!` | ADMIN |
| `test` | `Test1234!` | USER |
| `test2` | `Test1234!` | USER |

---

## Prod Stack (`GolfSyncProdStack`)

### Step 1 — Create a Route53 hosted zone for `golfsync.com`
If the domain is not already in Route53, create a hosted zone:
- AWS Console -> Route53 -> Hosted Zones -> Create hosted zone -> `golfsync.com`
- Update your domain registrar's nameservers to match the NS records Route53 provides.

Find the hosted zone ID:
```bash
aws route53 list-hosted-zones \
  --query "HostedZones[?Name=='golfsync.com.'].Id" --output text \
  --profile golfsync-prod
# Returns: /hostedzone/Z1234ABCDE5678  -- the ID is the part after /hostedzone/
```

### Step 2 — Request an ACM certificate for HTTPS
```bash
# Must be in us-east-1 for ALB
aws acm request-certificate \
  --domain-name golfsync.com \
  --subject-alternative-names '*.golfsync.com' \
  --validation-method DNS \
  --region us-east-1 \
  --profile golfsync-prod

# Get the certificate ARN
aws acm list-certificates \
  --query "CertificateSummaryList[?DomainName=='golfsync.com'].CertificateArn" \
  --output text \
  --region us-east-1 \
  --profile golfsync-prod
```

AWS Console -> Certificate Manager -> the new cert -> "Create records in Route53"

Wait for the certificate status to show **Issued** before deploying (usually 1-5 minutes after DNS propagation).

### Step 3 — Set env vars and deploy
```bash
export GOLFSYNC_HOSTED_ZONE_ID=<your-hosted-zone-id>
export GOLFSYNC_CERT_ARN=<your-acm-cert-arn>

cd golfsync-cdk
npx cdk deploy GolfSyncProdStack --profile golfsync-prod
```

This creates:
- Route53 A alias records: `golfsync.com` and `www.golfsync.com` to ALB
- ALB HTTPS listener on port 443 (TLS terminated using the ACM cert)
- ALB HTTP listener on port 80 with permanent redirect to HTTPS
- SES email identity for `golfsync.com` (outputs DNS verification records)

### Step 4 — Run Liquibase migrations

```bash
# Private subnet ID (prod tasks run in private subnets)
aws ec2 describe-subnets \
  --filters "Name=tag:aws-cdk:subnet-name,Values=private" \
            "Name=tag:aws-cdk:subnet-type,Values=Private" \
  --query "Subnets[0].SubnetId" --output text \
  --profile golfsync-prod

# API security group ID
aws ec2 describe-security-groups \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=GolfSyncProdStack" \
            "Name=group-name,Values=*ApiSg*" \
  --query "SecurityGroups[0].GroupId" --output text \
  --profile golfsync-prod

# Run the task (no --contexts flag — migration 024 is skipped in prod)
aws ecs run-task \
  --cluster golfsync-prod \
  --task-definition <LiquibaseTaskDefArn> \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[<private-subnet-id>],securityGroups=[<sg-id>],assignPublicIp=DISABLED}' \
  --count 1 \
  --started-by pre-deploy-liquibase \
  --profile golfsync-prod

# Wait for exit code 0
aws ecs describe-tasks --cluster golfsync-prod --tasks <task-arn> \
  --query "tasks[0].{status:lastStatus,exit:containers[0].exitCode}" \
  --profile golfsync-prod
```

### Step 5 — Verify SES domain ownership
After deploy, CDK outputs CNAME/TXT records for SES domain verification:
- AWS Console -> Route53 -> Hosted Zones -> `golfsync.com` -> Create record
- Or: AWS Console -> SES -> Verified Identities -> `golfsync.com` -> view the records

Wait for SES to show the domain as **Verified** before expecting emails to send.

### Step 6 — Populate secrets

```bash
# SES SMTP (required — emails fail without this)
# Generate: AWS Console -> SES -> SMTP Settings -> Create SMTP credentials
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/ses-smtp-credentials \
  --secret-string '{"username":"AKIAIOSFODNN7EXAMPLE","password":"<derived-smtp-password>"}' \
  --profile golfsync-prod

# OpenAI (optional — AI assistant disabled if missing)
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/openai-api-key \
  --secret-string 'sk-...' \
  --profile golfsync-prod

# Serper.dev (optional — tournament discovery disabled if missing)
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/serper-api-key \
  --secret-string 'your-serper-key' \
  --profile golfsync-prod

# Stripe live keys (activate before June 1, 2026)
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/stripe-secret-key \
  --secret-string 'sk_live_...' \
  --profile golfsync-prod

aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/stripe-publishable-key \
  --secret-string 'pk_live_...' \
  --profile golfsync-prod

aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/stripe-webhook-secret \
  --secret-string 'whsec_...' \
  --profile golfsync-prod
```

> **Billing activation checklist (June 1, 2026):**
> 1. Stripe live keys in Secrets Manager (above)
> 2. Register webhook in Stripe Dashboard: `https://golfsync.com/api/stripe/webhook`
> 3. Set `STRIPE_ENABLED=true` in the prod ECS task environment and redeploy
> 4. Send pre-billing notification email to all users

> **Auto-generated — no action needed:**
> `golfsync-prod/db-credentials` and `golfsync-prod/jwt-secret` are generated by CDK automatically.

### Step 7 — Set the admin UI password

Migration `001` creates the `admin` account. Migration `024` is dev-only — no preset password exists in prod. You must set it manually.

**Option A — Forgot password flow (simplest, requires SES to be working):**
Go to `https://golfsync.com/login` -> Forgot Password -> enter `admin@golfsync.io`.

**Option B — SSM port forwarding to RDS:**

1. Get the RDS endpoint and DB password:
```bash
RDS_HOST=$(aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'prod')].Endpoint.Address" \
  --output text --profile golfsync-prod)

DB_PASS=$(aws secretsmanager get-secret-value \
  --secret-id golfsync-prod/db-credentials \
  --query SecretString --output text --profile golfsync-prod \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['password'])")
```

2. Launch a temporary SSM-enabled bastion in the prod VPC:
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=GolfSyncProdStack" \
  --query "Vpcs[0].VpcId" --output text --profile golfsync-prod)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:aws-cdk:subnet-type,Values=Private" \
  --query "Subnets[0].SubnetId" --output text --profile golfsync-prod)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --instance-type t3.nano \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name=AmazonSSMRoleForInstancesQuickSetup \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=golfsync-prod-bastion-temp}]' \
  --query "Instances[0].InstanceId" --output text --profile golfsync-prod)
```

3. Open the SSM tunnel (keep this terminal open):
```bash
aws ssm start-session \
  --target $INSTANCE_ID \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters "{\"host\":[\"$RDS_HOST\"],\"portNumber\":[\"3306\"],\"localPortNumber\":[\"3307\"]}" \
  --profile golfsync-prod
```

4. In a second terminal — generate hash and update:
```bash
# Generate bcrypt hash (brew install httpd if needed)
htpasswd -bnBC 12 "" 'YourStrongPassword!' | tr -d ':\n'

# Connect and update
mysql -h 127.0.0.1 -P 3307 -u golfsync_admin -p"$DB_PASS" golfsync \
  -e "UPDATE users SET password_hash='<bcrypt-hash>', email='you@example.com' WHERE username='admin';"
```

5. Terminate the bastion immediately:
```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --profile golfsync-prod
```

### Step 8 — Request SES production access
By default, SES accounts are in sandbox mode and can only send to verified addresses. To send to any address:
- AWS Console -> SES -> Account dashboard -> Request production access
- Explain use case: transactional and marketing email for a golf social app; includes CAN-SPAM opt-out mechanism

---

## Quick Reference — All Secrets

| Secret ID | Who sets it | Required? | Purpose |
|---|---|---|---|
| `golfsync-dev/db-credentials` | CDK (auto) | Auto | RDS MySQL password |
| `golfsync-dev/jwt-secret` | CDK (auto) | Auto | JWT signing key |
| `golfsync-dev/gmail-smtp-credentials` | You (Step 4) | Required | Gmail app password for dev email |
| `golfsync-dev/openai-api-key` | You (Step 4) | Optional | AI assistant |
| `golfsync-dev/serper-api-key` | You (Step 4) | Optional | Tournament discovery |
| `golfsync-dev/stripe-secret-key` | You (Step 4) | Optional | Stripe test key |
| `golfsync-dev/stripe-publishable-key` | You (Step 4) | Optional | Stripe test key |
| `golfsync-dev/stripe-webhook-secret` | You (Step 4) | Optional | Stripe webhook |
| `golfsync-prod/db-credentials` | CDK (auto) | Auto | RDS MySQL password |
| `golfsync-prod/jwt-secret` | CDK (auto) | Auto | JWT signing key |
| `golfsync-prod/ses-smtp-credentials` | You (Step 6) | Required | SES SMTP for prod email |
| `golfsync-prod/openai-api-key` | You (Step 6) | Optional | AI assistant |
| `golfsync-prod/serper-api-key` | You (Step 6) | Optional | Tournament discovery |
| `golfsync-prod/stripe-secret-key` | You (Step 6) | June 1 | Stripe live key |
| `golfsync-prod/stripe-publishable-key` | You (Step 6) | June 1 | Stripe live key |
| `golfsync-prod/stripe-webhook-secret` | You (Step 6) | June 1 | Stripe webhook |

---

## CDK Environment Variable Reference

These must be set in your shell (`~/.zshrc`) before running any CDK command.

| Variable | Stack | Purpose |
|---|---|---|
| `GOLFSYNC_DEV_ACCOUNT_ID` | Dev | Pins GolfSyncDevStack to the dev workload account |
| `GOLFSYNC_PROD_ACCOUNT_ID` | Prod | Pins GolfSyncProdStack to the prod workload account |
| `GOLFSYNC_DEV_ALARM_EMAIL` | Dev | CloudWatch alarm notification email |
| `GOLFSYNC_PROD_ALARM_EMAIL` | Prod | CloudWatch alarm notification email |
| `GOLFSYNC_HOSTED_ZONE_ID` | Prod | Route53 hosted zone ID for `golfsync.com` |
| `GOLFSYNC_CERT_ARN` | Prod | ACM certificate ARN for HTTPS |

---

## Tearing Down

```bash
# Dev (safe to destroy — no deletion protection)
cd golfsync-cdk
npx cdk destroy GolfSyncDevStack --profile golfsync-dev

# Prod (deletion protection is ON on RDS — must disable first)
# AWS Console -> RDS -> Modify instance -> uncheck deletion protection -> apply immediately
npx cdk destroy GolfSyncProdStack --profile golfsync-prod
```

---

## ECS Exec (for debugging live containers)

```bash
# Find the task ID first
aws ecs list-tasks --cluster golfsync-dev --profile golfsync-dev
aws ecs list-tasks --cluster golfsync-prod --profile golfsync-prod

# Shell into the API container
aws ecs execute-command --cluster golfsync-dev \
  --task <task-id> --interactive --command "/bin/sh" \
  --profile golfsync-dev

aws ecs execute-command --cluster golfsync-prod \
  --task <task-id> --interactive --command "/bin/sh" \
  --profile golfsync-prod
```

> **Note:** The API container is a JDK image — no MySQL client is available. Use SSM port forwarding (Prod Step 7) to connect to RDS directly.
