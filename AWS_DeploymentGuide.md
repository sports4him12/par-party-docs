# GolfSync — AWS Deployment Guide

Step-by-step checklist for deploying the Dev and Prod stacks using AWS CDK.

**Last updated:** 2026-04-19

---

## Prerequisites

### 1. AWS CLI configured — IAM Identity Center (SSO)

GolfSync uses IAM Identity Center with separate workload accounts for Dev and Prod.
The management (root) account hosts IAM Identity Center only — no workload resources are deployed there.

**One-time SSO profile setup:** ✅ Done (`golfsync-dev` and `golfsync-prod` profiles configured)
```bash
# Dev workload account profile
aws configure sso --profile golfsync-dev
# SSO session name:  golfsync
# SSO start URL:     https://<your-org>.awsapps.com/start
# SSO region:        us-east-1
# Account:           161457898848  (dev)
# Permission set:    AdministratorAccess
# CLI default region: us-east-1
# CLI output format:  json

# Prod workload account profile
aws configure sso --profile golfsync-prod
# (same SSO session name/start URL — select the prod account this time)
# Account:           805865757850  (prod)
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

**One-time: set account IDs and alarm email in `~/.zshrc`:** ✅ Done
```bash
# Already set in ~/.zshrc:
export GOLFSYNC_DEV_ACCOUNT_ID=161457898848
export GOLFSYNC_PROD_ACCOUNT_ID=805865757850
export GOLFSYNC_DEV_ALARM_EMAIL=support@golfsync.io
export GOLFSYNC_PROD_ALARM_EMAIL=support@golfsync.io
```

> **Why account IDs matter:** With `GOLFSYNC_DEV_ACCOUNT_ID` set, running `cdk deploy GolfSyncDevStack --profile golfsync-prod` will fail — CDK detects the account mismatch before touching anything.

### 2. Bootstrap CDK (one-time per workload account)

Each workload account needs CDK bootstrapped once. Do **not** bootstrap the management account.

✅ **Dev account bootstrapped** (`161457898848`)

```bash
cd golfsync-cdk

# Dev account — already done
npx cdk bootstrap aws://$GOLFSYNC_DEV_ACCOUNT_ID/us-east-1 --profile golfsync-dev

# Prod account — run before first prod deploy
npx cdk bootstrap aws://$GOLFSYNC_PROD_ACCOUNT_ID/us-east-1 --profile golfsync-prod
```

### 3. Docker running
CDK builds container images from source (`golfsync-api`, `golfsync-web`) and pushes them to ECR. Docker must be running on your machine at deploy time.

> **Note:** Camunda BPM is embedded in-process inside `golfsync-api` — there is no separate Camunda ECS service or container to build.

---

## Dev Stack (`GolfSyncDevStack`) ✅ Deployed

**Live URLs:**
- CloudFront (HTTPS): `https://dehi6vc3s5x1l.cloudfront.net` ← use this
- ALB (HTTP only): `http://GolfSy-Alb16-Ms3amAWFxLgj-225436661.us-east-1.elb.amazonaws.com`

> Always use the CloudFront URL. The ALB is plain HTTP and browser APIs (geolocation, `crypto.randomUUID`, secure cookies) require HTTPS.

### Step 1 — Deploy ✅ Done

```bash
scripts/deploy.sh dev
```

The script handles: SSO check → `cdk diff` review → Liquibase prompt → deploy → ECS stabilization watch.

CDK outputs the `DevCdnUrl` (CloudFront HTTPS URL) and `AppUrl` (plain ALB). Use `DevCdnUrl`.

> **CORS:** CDK automatically sets `GOLFSYNC_CORS_ALLOWED_ORIGIN` to the CloudFront URL — no manual step needed.

### Step 2 — Liquibase migrations ✅ Done (runs inline in dev)

Dev runs Liquibase automatically inside the API container on startup (`SPRING_LIQUIBASE_ENABLED=true`).
The `SPRING_LIQUIBASE_CONTEXTS=dev` flag seeds admin/test/test2 accounts (migration `024`).

**For subsequent deploys that include DB schema changes**, use the deploy script (it will prompt):
```bash
scripts/deploy.sh dev
# Answer "y" to the Liquibase prompt to run the separate migration task before deploying
```

If you need to run the Liquibase task manually:
```bash
# Get subnet and security group IDs
aws ec2 describe-subnets \
  --filters "Name=tag:aws-cdk:subnet-name,Values=public" "Name=tag:aws-cdk:subnet-type,Values=Public" \
  --query "Subnets[0].SubnetId" --output text --profile golfsync-dev

aws ec2 describe-security-groups \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=GolfSyncDevStack" \
            "Name=group-name,Values=*ApiSg*" \
  --query "SecurityGroups[0].GroupId" --output text --profile golfsync-dev

# Run migration task (use LiquibaseMigrateCommand from CDK outputs)
aws ecs run-task \
  --cluster golfsync-dev \
  --task-definition <LiquibaseTaskDefArn> \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[<public-subnet-id>],securityGroups=[<sg-id>],assignPublicIp=ENABLED}' \
  --count 1 \
  --started-by pre-deploy-liquibase \
  --profile golfsync-dev

# Wait for exit code 0
aws ecs describe-tasks --cluster golfsync-dev --tasks <task-arn> \
  --query "tasks[0].{status:lastStatus,exit:containers[0].exitCode}" \
  --profile golfsync-dev
```

### Step 3 — Update `appBaseUrl` ✅ Done

Already set in `golfsync-cdk/bin/golfsync-cdk.ts`:
```ts
appBaseUrl: 'https://dehi6vc3s5x1l.cloudfront.net',
```

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

## Prod Stack (`GolfSyncProdStack`) — Deployed

### Step 1 — Route53 hosted zone ✅

Hosted zone ID: `Z0171145TFPVNPO0FH1Z` (hardcoded in `golfsync-cdk/bin/golfsync-cdk.ts`)

### Step 2 — ACM certificate ✅

Certificate ARN: `arn:aws:acm:us-east-1:805865757850:certificate/7710d209-daf1-4b9a-bcca-591d5cf2a005`
(hardcoded in `golfsync-cdk/bin/golfsync-cdk.ts` — no env var needed at deploy time)

### Step 3 — Deploy ✅

Regular prod deploys go through the standard wrapper — all branches required on `main`,
Liquibase runs post-deploy, ECS stabilisation is watched, and the script tags the release
`prod-YYYY-MM-DD-HHmm` on every submodule.

```bash
./scripts/deploy.sh prod
```

> **First-deploy-only:** The original bootstrap used `--skip-liquibase` because the stack
> didn't exist yet and the Liquibase task ARN wasn't readable from outputs. That situation
> has passed — normal prod deploys do NOT need the flag.

This creates:
- Route53 A alias records: `golfsync.com` and `www.golfsync.com` → ALB
- ALB HTTPS listener on port 443 (TLS terminated using the ACM cert)
- ALB HTTP listener on port 80 with permanent redirect to HTTPS
- CloudFront distribution in front of the ALB
- SES email identity for `golfsync.com` (outputs DNS verification records)

### Step 4 — Run Liquibase migrations

Run after Step 3 completes and the stack outputs are available.

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

# Run migration task (use LiquibaseMigrateCommand from CDK outputs — no dev contexts flag)
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

After Liquibase succeeds, force a fresh API deployment so the service picks up the schema:
```bash
aws ecs update-service --cluster golfsync-prod \
  --service <ApiServiceName> --force-new-deployment \
  --profile golfsync-prod
```

### Step 5 — Verify SES domain ownership

After deploy, CDK outputs CNAME/TXT records for SES domain verification:
- AWS Console → Route53 → Hosted Zones → `golfsync.com` → Create record
- Or: AWS Console → SES → Verified Identities → `golfsync.com` → view the records

Wait for SES to show the domain as **Verified** before expecting emails to send.

### Step 6 — Populate secrets

```bash
# SES SMTP (required — emails fail without this)
# Generate: AWS Console -> SES -> SMTP Settings -> Create SMTP credentials
aws secretsmanager put-secret-value \
  --secret-id golfsync-prod/ses-smtp-credentials \
  --secret-string '{"username":"AKIAIOSFODNN7EXAMPLE","password":"<derived-smtp-password>"}' \
  --profile golfsync-prod

# OpenAI (required — AI assistant disabled if missing)
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
> 3. Send pre-billing notification email to all users

> **Auto-generated — no action needed:**
> `golfsync-prod/db-credentials` and `golfsync-prod/jwt-secret` are generated by CDK automatically.

### Step 7 — Set the admin password

Migration `001` creates the `admin` account. Migration `024` is dev-only — no preset password exists in prod.

**Option A — Forgot password flow (simplest, requires SES to be working):**
Go to `https://golfsync.com/login` → Forgot Password → enter `admin@golfsync.io`.

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
- AWS Console → SES → Account dashboard → Request production access
- Explain use case: transactional and marketing email for a golf social app; includes CAN-SPAM opt-out mechanism

---

## Infrastructure Additions (April 2026)

The core Dev + Prod stacks above capture the baseline. These additions have shipped since:

### Beta stack (`GolfSyncBetaStack`)

Isolated environment for testing feature branches (e.g. League Management) without touching dev or prod.

- Runs in the **dev AWS account** (same profile: `golfsync-dev`), so no extra cost vs. the baseline dev account.
- Scale to zero when not in use: `./scripts/deploy.sh beta --shutdown` sets ECS desired-count to 0 on both api and web services.
- Fully tear down: `./scripts/deploy.sh beta --destroy` — skips the termination-protection prompt and drops the stack entirely.
- Deploy branch gate: beta deploys require `golfsync-api` and `golfsync-web` to be checked out on the `LeagueMockUp` branch; CDK can stay on `main`.
- No Route53 or custom domain — accessed via the ALB URL.

### WAF US-only geo-restriction (prod ALB)

Prod ALB now sits behind an AWS WAF Web ACL that blocks all non-US traffic. This keeps the geographic scope consistent with the business focus and cuts bot/scraper load from outside the US.

- Managed by CDK — no manual WAF configuration.
- To temporarily allow a non-US visitor (e.g. for demo), add an exception rule in the AWS Console; revert after.

### Route53 uptime health check + alarm

Prod has a Route53 HTTPS health check against `/healthz` on golfsync.io, with a CloudWatch alarm wired to the existing prod SNS topic. If the check fails for 2 consecutive minutes, the alarm email (`GOLFSYNC_PROD_ALARM_EMAIL`) fires.

### RDS MySQL 8.4 LTS upgrade flag

CDK now sets `allowMajorVersionUpgrade=true` on the prod RDS instance, letting us move from MySQL 8.0 → 8.4.8 LTS without recreating the instance. **Must upgrade before MySQL 8.0 EOL on 2026-07-31.** See `golfsync-cdk/lib/golfsync-cdk-stack.ts` for the `engineVersion` pin.

### `golfsync-ops` IAM user

Read-only IAM user for CloudWatch + ECS console access (log tailing, service status, task descriptions) without granting full AWS console access. Console sign-in URL and username are exposed as CDK stack outputs on prod.

### Admin signup notification email

When a new account is created in prod, the API sends a notification to the operator. Gated by the `ADMIN_SIGNUP_NOTIFICATION_EMAIL` env var — unset (dev, beta) = silent. Wired via the CDK `adminSignupNotificationEmail` field on `EnvironmentConfig`; current prod value is `ryanrpick@golfsync.io`.

### Universal-link manifest routes (web)

The web app serves two route handlers at `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` so iOS and Android can verify Golf Sync as a handler for `https://golfsync.io/invite/*` URLs.

Both routes require env vars on the prod web task — until set they return 404 cleanly, so deep links gracefully fall back to the custom scheme (`golfsync://invite/<token>`) or mobile Safari:

| Env var | Purpose |
|---|---|
| `APPLE_TEAM_ID` | 10-character Apple Developer Team ID; goes into AASA `appID` |
| `ANDROID_CERT_SHA256` | SHA-256 cert fingerprints (comma-separated) for the upload key and Play App Signing key |

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
| `golfsync-prod/openai-api-key` | You (Step 6) | Required | AI assistant |
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

> `GOLFSYNC_HOSTED_ZONE_ID` and `GOLFSYNC_CERT_ARN` are no longer needed as env vars —
> both are hardcoded in `golfsync-cdk/bin/golfsync-cdk.ts`.

---

## Tearing Down

```bash
# Dev (safe to destroy — no deletion protection)
cd golfsync-cdk
npx cdk destroy GolfSyncDevStack --profile golfsync-dev

# Prod (deletion protection is ON on RDS — must disable first)
# AWS Console -> RDS -> Modify instance -> uncheck deletion protection -> apply immediately
# Prod stack also has CloudFormation termination protection — disable in Console first
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
