# GolfSync — Prod Go-Live Checklist

**Billing start date:** June 1, 2026  
**Soft-launch target:** May 2026 (invite-only, free trial)

Work through this top to bottom. Every item must be checked before real users are on the platform.

---

## 1. Local Tooling

- [ ] AWS CLI v2 installed (`aws --version`)
- [ ] Node.js + npx installed (`node --version`)
- [ ] Docker running (`docker ps`)
- [ ] CDK installed (`npx cdk --version`)

---

## 2. AWS Accounts & IAM Identity Center

- [ ] Management account hosts IAM Identity Center only — no workload resources deployed there
- [ ] Dev workload account created
- [ ] Prod workload account created
- [ ] IAM Identity Center user created and activation email accepted
- [ ] User assigned to dev account with `AdministratorAccess` permission set
- [ ] User assigned to prod account with `AdministratorAccess` permission set
- [ ] SSO profiles configured locally:
  ```bash
  aws configure sso --profile golfsync-dev
  aws configure sso --profile golfsync-prod
  ```
- [ ] Account IDs set in `~/.zshrc`:
  ```bash
  export GOLFSYNC_DEV_ACCOUNT_ID=<dev-account-id>
  export GOLFSYNC_PROD_ACCOUNT_ID=<prod-account-id>
  ```
- [ ] CDK bootstrapped in dev account:
  ```bash
  npx cdk bootstrap aws://$GOLFSYNC_DEV_ACCOUNT_ID/us-east-1 --profile golfsync-dev
  ```
- [ ] CDK bootstrapped in prod account:
  ```bash
  npx cdk bootstrap aws://$GOLFSYNC_PROD_ACCOUNT_ID/us-east-1 --profile golfsync-prod
  ```

---

## 3. DNS & SSL (Prod only)

- [ ] `golfsync.com` domain registered at a registrar
- [ ] Route53 hosted zone created for `golfsync.com`:
  ```bash
  aws route53 create-hosted-zone --name golfsync.com --caller-reference $(date +%s) --profile golfsync-prod
  ```
- [ ] Domain registrar nameservers updated to match the 4 NS records Route53 provides
- [ ] ACM certificate requested (must be in `us-east-1`):
  ```bash
  aws acm request-certificate \
    --domain-name golfsync.com \
    --subject-alternative-names '*.golfsync.com' \
    --validation-method DNS \
    --region us-east-1 \
    --profile golfsync-prod
  ```
- [ ] ACM DNS validation CNAME records added to Route53 (AWS Console → ACM → "Create records in Route53")
- [ ] ACM certificate status = **Issued**
- [ ] `GOLFSYNC_HOSTED_ZONE_ID` and `GOLFSYNC_CERT_ARN` env vars set:
  ```bash
  export GOLFSYNC_HOSTED_ZONE_ID=Z1234ABCDE5678
  export GOLFSYNC_CERT_ARN=arn:aws:acm:us-east-1:...
  ```

---

## 4. CDK Deploy

- [ ] `aws sso login --profile golfsync-prod` run (active session)
- [ ] `GOLFSYNC_PROD_ALARM_EMAIL` set:
  ```bash
  export GOLFSYNC_PROD_ALARM_EMAIL=support@golfsync.io
  ```
- [ ] Prod stack deployed:
  ```bash
  cd golfsync-cdk
  npx cdk deploy GolfSyncProdStack --profile golfsync-prod
  ```
- [ ] Note the CDK outputs:
  - `AppUrl` — the ALB DNS name
  - `LiquibaseMigrateCommand` — the ECS task run command

---

## 5. Database Migrations

- [ ] Liquibase migration ECS task run (see `LiquibaseMigrateCommand` output):
  ```bash
  aws ecs run-task \
    --cluster golfsync-prod \
    --task-definition <LiquibaseTaskDefArn> \
    --launch-type FARGATE \
    --network-configuration 'awsvpcConfiguration={subnets=[<private-subnet-id>],securityGroups=[<api-sg-id>],assignPublicIp=DISABLED}' \
    --count 1 \
    --started-by pre-deploy-liquibase \
    --profile golfsync-prod
  ```
- [ ] Migration task exits with code `0`:
  ```bash
  aws ecs describe-tasks --cluster golfsync-prod --tasks <task-arn> \
    --query "tasks[0].{status:lastStatus,exit:containers[0].exitCode}" \
    --profile golfsync-prod
  ```
- [ ] API service health check passes (ECS shows task as RUNNING)

---

## 6. Secrets Manager — Prod

All of these are created by CDK with placeholder values. Replace them before traffic hits the app.

### Required (app broken without these)

- [ ] SES SMTP credentials:
  ```bash
  # Generate: AWS Console → SES → SMTP Settings → Create SMTP credentials
  aws secretsmanager put-secret-value \
    --secret-id golfsync-prod/ses-smtp-credentials \
    --secret-string '{"username":"AKIAIOSFODNN7EXAMPLE","password":"<derived-smtp-password>"}' \
    --profile golfsync-prod
  ```

### Required for features

- [ ] OpenAI API key (AI assistant):
  ```bash
  aws secretsmanager put-secret-value \
    --secret-id golfsync-prod/openai-api-key \
    --secret-string 'sk-...' \
    --profile golfsync-prod
  ```

- [ ] Stripe live keys (billing — activate before June 1):
  ```bash
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

### Optional (features silently disabled if missing)

- [ ] Serper API key (tournament discovery):
  ```bash
  aws secretsmanager put-secret-value \
    --secret-id golfsync-prod/serper-api-key \
    --secret-string 'your-serper-key' \
    --profile golfsync-prod
  ```

- [ ] Google CSE key + CX (tournament discovery):
  ```bash
  aws secretsmanager put-secret-value \
    --secret-id golfsync-prod/google-cse-api-key \
    --secret-string 'AIza...' \
    --profile golfsync-prod

  aws secretsmanager put-secret-value \
    --secret-id golfsync-prod/google-cse-cx \
    --secret-string '<cx-id>' \
    --profile golfsync-prod
  ```

### Auto-generated — no action needed

- `golfsync-prod/db-credentials` — random 32-char password, created by CDK
- `golfsync-prod/jwt-secret` — random 64-char key, created by CDK

---

## 7. Email — Amazon SES

- [ ] SES domain DNS verification records added to Route53 (CDK outputs these after deploy)
- [ ] SES domain status = **Verified** (AWS Console → SES → Verified Identities)
- [ ] SES production access requested (removes sandbox restriction):
  - AWS Console → SES → Account dashboard → Request production access
  - Use case: transactional + marketing email for a golf app; CAN-SPAM opt-out implemented
- [ ] SES production access **approved** by AWS (usually 24 hours)
- [ ] Test email sent and received:
  ```bash
  aws ses send-email \
    --from noreply@golfsync.com \
    --to your-inbox@example.com \
    --subject "GolfSync SES test" \
    --text "SES is working." \
    --region us-east-1 \
    --profile golfsync-prod
  ```

---

## 8. Admin Account

- [ ] Admin password set (migration 024 is dev-only — prod has no preset password):
  1. Generate bcrypt hash: `htpasswd -bnBC 12 "" 'YourStrongPassword!' | tr -d ':\n'`
  2. Connect to RDS via SSM port-forward (see `AWS_DeploymentGuide.md` § Step 7)
  3. `UPDATE users SET password_hash='<hash>' WHERE username='admin';`
- [ ] Admin email updated from `admin@golfsync.io` to a real monitored inbox
- [ ] Admin login verified at `https://golfsync.com/login`
- [ ] Admin portal verified at `https://golfsync.com/admin`

---

## 9. Stripe Billing Setup

- [ ] Stripe account in live mode (not test mode)
- [ ] Monthly price created in Stripe Dashboard: **$9.99/month recurring**
- [ ] Annual price created in Stripe Dashboard: **$79.99/year recurring**
- [ ] `STRIPE_MONTHLY_PRICE_ID` and `STRIPE_ANNUAL_PRICE_ID` set in prod ECS task environment
- [ ] Webhook endpoint registered in Stripe Dashboard:
  - URL: `https://golfsync.com/api/stripe/webhook`
  - Events: `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`
- [ ] Webhook signing secret (`whsec_...`) stored in `golfsync-prod/stripe-webhook-secret`
- [ ] `STRIPE_ENABLED=true` set in prod ECS task (update CDK or set directly before June 1)
- [ ] End-to-end payment test with a Stripe test card passed

---

## 10. Monitoring

- [ ] CloudWatch error alarm SNS email subscription confirmed (check inbox after CDK deploy — AWS sends a confirmation link)
- [ ] CloudWatch dashboard visible in AWS Console (`/golfsync-prod/api` log group)
- [ ] RDS CloudWatch metrics visible (CPU, connections, storage)

---

## 11. Security Verification

- [ ] `https://golfsync.com` loads (HTTPS enforced)
- [ ] `http://golfsync.com` redirects to `https://golfsync.com`
- [ ] `www.golfsync.com` resolves correctly
- [ ] Auth cookie is `HttpOnly; Secure; SameSite=Strict` (check browser DevTools → Application → Cookies)
- [ ] `COOKIE_SECURE=true` confirmed (CDK sets this automatically for prod)
- [ ] Admin password is not `GolfSync1!` (dev seed password — never reaches prod, but verify)

---

## 12. Smoke Tests

Run these manually after deploy, before announcing to any users:

- [ ] Register a new account — confirmation email received
- [ ] Log in with new account
- [ ] Password reset flow — reset email received, link works
- [ ] Create a round
- [ ] Invite a friend (by username)
- [ ] Membership page loads at `/membership`
- [ ] Support form at `/support#contact` submits and confirmation email received
- [ ] `/terms` and `/privacy` pages load
- [ ] Admin portal: list users, view rounds, view analytics

---

## 13. Billing Activation (June 1, 2026)

Do not do this before June 1. Come back to this section on billing day.

- [ ] Pre-billing email sent to all users at least 7 days before June 1
- [ ] Stripe live keys confirmed in Secrets Manager
- [ ] Stripe webhook confirmed active
- [ ] `STRIPE_ENABLED=true` in prod ECS task
- [ ] `STRIPE_MONTHLY_PRICE_ID` and `STRIPE_ANNUAL_PRICE_ID` confirmed
- [ ] Redeploy API task to pick up new env vars
- [ ] End-to-end subscription test with a real card (then cancel/refund)

---

## 14. Open Business Decisions (block soft launch or June 1)

- [ ] **GolfNow API** — negotiate partnership or reposition as "round management" and remove tee-time promise. Currently mocked. Blocking real marketing.
- [ ] **3-step onboarding flow** — already built; verify it appears for new registrations
- [ ] **support@golfsync.io inbox** — confirm it is monitored before users can contact you

---

## Quick Reference — Prod Secret IDs

| Secret ID | Who sets it | Status at CDK deploy |
|---|---|---|
| `golfsync-prod/db-credentials` | CDK (auto) | Ready |
| `golfsync-prod/jwt-secret` | CDK (auto) | Ready |
| `golfsync-prod/ses-smtp-credentials` | You (§6) | Placeholder — must replace |
| `golfsync-prod/openai-api-key` | You (§6) | Placeholder — must replace |
| `golfsync-prod/stripe-secret-key` | You (§9) | Placeholder — must replace |
| `golfsync-prod/stripe-publishable-key` | You (§9) | Placeholder — must replace |
| `golfsync-prod/stripe-webhook-secret` | You (§9) | Placeholder — must replace |
| `golfsync-prod/serper-api-key` | You (§6) | Placeholder — optional |
| `golfsync-prod/google-cse-api-key` | You (§6) | Placeholder — optional |
| `golfsync-prod/google-cse-cx` | You (§6) | Placeholder — optional |
