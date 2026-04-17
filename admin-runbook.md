# Golf Sync — Admin & Troubleshooting Runbook

**Audience:** Admin operators responding to user issues or system alerts  
**Last updated:** 2026-04-17

---

## Quick Reference

| What you need | Where to look |
|---|---|
| All API errors | CloudWatch → `/golfsync-prod/api` |
| DB migration output | CloudWatch → `/golfsync-prod/liquibase` |
| CloudWatch alarms | CloudWatch Alarms → `golfsync-prod-*` (8 critical alarms) |
| Uptime monitoring | CloudWatch Alarms → `golfsync-prod-uptime-down` |
| Operations dashboard | CloudWatch Dashboards → `golfsync-prod` |
| Admin dashboard | `https://golfsync.io/admin` (ADMIN role required) |
| Tournament management | `https://golfsync.io/admin` → Tournaments tab (ADMIN or TOURNAMENT_SUPPORT) |
| Health check | `GET /health` → 200 means API is up |

---

## 1. CloudWatch Log Groups

All logs are structured JSON and shipped via the AWS Logs driver on the ECS task.

| Log Group | What's in it | Retention |
|---|---|---|
| `/golfsync-prod/api` | All Spring Boot output — requests, errors, Stripe events, email sends, AI assistant | 90 days |
| `/golfsync-dev/api` | Same for dev stack | 30 days |
| `/golfsync-prod/liquibase` | Liquibase migration runs — schema change history | 90 days |
| `/golfsync-dev/liquibase` | Same for dev stack | 30 days |

**Log stream naming:** `api/<container-id>` within each log group. Each ECS task start creates a new stream.

### Navigating to logs

1. AWS Console → **CloudWatch** → **Log groups**
2. Open `/golfsync-prod/api`
3. Click **Search log group** (top right)
4. Set time range to the incident window
5. Enter a filter pattern (see below)

### Useful CloudWatch Insights queries

```sql
-- All ERROR-level events in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Stripe webhook events
fields @timestamp, @message
| filter @message like /checkout.session/ or @message like /invoice./ or @message like /subscription/
| sort @timestamp desc

-- Payment failures
fields @timestamp, @message
| filter @message like /invoice.payment_failed/ or @message like /payment failed/
| sort @timestamp desc

-- Email send failures
fields @timestamp, @message
| filter @message like /Failed to send/ or @message like /MailException/
| sort @timestamp desc

-- Rate limit hits
fields @timestamp, @message
| filter @message like /Rate limit/
| sort @timestamp desc

-- Content moderation blocks
fields @timestamp, @message
| filter @message like /Message blocked/ or @message like /moderation/
| sort @timestamp desc

-- Tournament discovery
fields @timestamp, @message
| filter @message like /Discovered/ or @message like /tournament discovery/
| sort @timestamp desc

-- Auth events (login failures, lockouts)
fields @timestamp, @message
| filter @message like /AUTH /
| sort @timestamp desc

-- Audit trail (admin actions)
fields @timestamp, @message
| filter @message like /AUDIT/
| sort @timestamp desc
```

---

## 2. CloudWatch Alarms (Prod Only)

Prod has 8 critical alarms. Dev has none (dev is only up during deploys).

| Alarm | Trigger | What to do |
|---|---|---|
| `golfsync-prod-api-error-rate` | 5+ ERROR logs in 5 min | Check `/golfsync-prod/api` for errors |
| `golfsync-prod-api-cpu-critical` | CPU > 85% for 10 min | May need more ECS tasks or larger task |
| `golfsync-prod-api-memory-critical` | Memory > 90% for 10 min | OOM risk — check for memory leaks |
| `golfsync-prod-api-tasks-critical` | Running tasks < 2 | Deploy failure or crash loop — check ECS |
| `golfsync-prod-rds-cpu-critical` | RDS CPU > 85% for 10 min | Slow queries — check RDS Performance Insights |
| `golfsync-prod-rds-storage-critical` | Free storage < 2 GB | Expand storage immediately in RDS Console |
| `golfsync-prod-alb-5xx-critical` | 5xx rate > 5% | Service degraded — check API logs |
| `golfsync-prod-uptime-down` | /health unreachable 2+ min | Site may be down — check ECS + ALB |

All alarms route to SNS → email.

---

## 3. Issue: User Cannot Log In

### Steps (all via Admin Dashboard)

1. **Admin Dashboard → Users tab** → search/scroll to find the user by email
2. Check the row for: `role`, `banned` status, `membershipTier`, `trialExpiresAt`
3. If **banned**: check **User Reports** tab for the report that triggered it. Unban via the Users tab toggle.
4. If **password expired**: passwords expire every 90 days. Direct the user to `/change-password`.
5. If **account locked**: after 5 failed login attempts, accounts lock for 15 minutes. This resolves automatically — no admin action needed.

### Checking logs (if needed)

```sql
-- CloudWatch Insights → /golfsync-prod/api
fields @timestamp, @message
| filter @message like /AUTH/ and @message like /locked\|failed\|banned/
| sort @timestamp desc
```

Note: Auth logs use userId only (no email addresses logged for privacy).

---

## 4. Issue: User Can't Access Features (Trial / Membership)

### Error codes (HTTP 403)

| Code | Meaning |
|---|---|
| `TRIAL_EXPIRED` | 30-day trial lapsed, no paid membership |
| `MEMBERSHIP_REQUIRED` | No trial set (unusual legacy path) |
| `FORBIDDEN` | Account banned or insufficient role |

### Steps

1. **Admin Dashboard → Users tab** → find the user → check `membershipTier`, `trialExpiresAt`, `membershipExpiresAt`
2. If **paid but no access**: Stripe Dashboard → Customers → search by email → verify subscription is active
3. If **Stripe webhook failed**: Stripe Dashboard → Developers → Webhooks → check for failed deliveries → re-send manually
4. To **extend trial manually** (DB access required):
   ```sql
   UPDATE users SET trial_expires_at = '2026-06-30 23:59:59' WHERE email = 'user@example.com';
   ```
5. To **manually activate membership** (DB access required):
   ```sql
   UPDATE users SET membership_tier = 'MONTHLY', membership_expires_at = DATE_ADD(NOW(), INTERVAL 1 MONTH) WHERE email = 'user@example.com';
   ```

---

## 5. Issue: User Not Receiving Emails

### Types of emails sent

| Email Type | Trigger | Affected by opt-out? |
|---|---|---|
| Welcome email | Registration | No |
| Friend request / accepted | Friend actions | No |
| Poll invite | Added to availability poll | No |
| Round invite | Invited to a round | No |
| Tournament reminder | Scheduled daily 8 AM | Yes |
| Trial drip (Day 3, 7) | Scheduled daily 9 AM | Yes |
| Dunning (Day 0, 3, 7) | Payment failure | No |
| Renewal reminder | 7 days before renewal | No |
| Account deletion | Self-deletion | No |

### Steps

1. **Admin Dashboard → Users tab** → find user → check `emailMarketingOptOut`. If opted out, marketing/drip emails are silently skipped (CAN-SPAM compliance).
2. **Check CloudWatch for send failures**:
   ```sql
   fields @timestamp, @message
   | filter @message like /Failed to send/ or @message like /MailException/
   | sort @timestamp desc
   ```
3. **Check SES**: AWS Console → SES → Account dashboard → Sending statistics. If daily quota is hit, emails drop.

---

## 6. Issue: Payment / Billing Problems

### Steps

1. **Stripe Dashboard → Customers** → search by user email → check subscription status and payment history
2. **Stripe Dashboard → Developers → Webhooks** → check delivery status for the relevant event
3. **Re-send failed webhooks** from the Stripe Dashboard directly
4. Key events to look for:
   - `checkout.session.completed` — subscription provisioned
   - `invoice.paid` — renewal extended
   - `invoice.payment_failed` — payment failed (dunning emails sent)
   - `customer.subscription.deleted` — access revoked

---

## 7. Issue: API Returning 5xx Errors

### Steps (all via AWS Console)

1. **CloudWatch Dashboard → `golfsync-prod`** — check CPU, memory, error rate, task count
2. **EC2 → Target Groups** → find API target group → check for unhealthy targets
3. **ECS → Clusters → `golfsync-prod`** → check running vs desired task count
4. **CloudWatch → `/golfsync-prod/api`** → search for exceptions
5. **RDS** → check instance status, CPU, connections, free storage

---

## 8. Rate Limiting & WAF

### Rate limits

| Endpoint | Limit | Scope |
|---|---|---|
| Auth (login, register) | 10/min | Per IP |
| Change password | 5/min | Per user |
| AI assistant | 5/min | Per user |
| Messages | 10/min | Per user |
| User search | 30/min | Per IP |
| Tournaments | 30/min | Per IP |
| All other mutations (POST/PUT/PATCH/DELETE) | 10/min | Per user |

### WAF Geo-Restriction

Prod has an AWS WAF that blocks all traffic from outside the United States. If a user reports they can't access the site while traveling internationally, they need a US VPN. Check WAF metrics in CloudWatch → `golfsync-prod-waf`.

---

## 9. Content Moderation

Messages are moderated by three layers:

1. **URL pattern filter** — blocks phishing, adult, malware link patterns
2. **OpenAI Moderation API** — detects hate, violence, sexual content (free)
3. **Keyword blocklist fallback** — catches worst content if OpenAI is unavailable

Blocked messages return HTTP 400 to the sender. Users see: "Your message was flagged for inappropriate content." The messages page shows a safety notice banner.

---

## 10. Admin Dashboard — What's Where

Access at `https://golfsync.io/admin`

| Role | Access |
|---|---|
| **ADMIN** | Full panel — all tabs |
| **TOURNAMENT_SUPPORT** | Tournaments + Reports tabs only. Sees "Tournaments" link in dashboard nav. |

### Tabs

| Tab | Who | What you can do |
|---|---|---|
| **Analytics** | ADMIN | Signups/day, conversions, churn, attribution, cancellation reasons |
| **Users** | ADMIN | User list, ban/unban, role assignment, delete, impersonate |
| **Rounds** | ADMIN | All rounds, update status |
| **User Reports** | ADMIN | Reports about users — approve (ban) or deny |
| **Promo Codes** | ADMIN | Create/deactivate promo codes |
| **Tournaments** | ADMIN + TS | State visibility toggle, curated tournaments, Serper refresh, featured tournaments, tournament reports |
| **Courses** | ADMIN | Featured courses management |
| **Feedback** | ADMIN | View user feedback |
| **Polls** | ADMIN | Create/manage site-wide polls |

### Tournament Management

**State Visibility** — toggle which states users can see. Green = visible, red = hidden.

**Curated Tournaments:**
- **State selector + Refresh from Google** — pick a state and refresh to discover new tournaments
- **Show/Hide Removed** — toggle to see deactivated tournaments
- **Edit** any row — sets `adminLocked=true` so Serper never overwrites your edits
- **Remove** — soft-deletes (hides from users)

**Key:** The weekly Serper refresh (Mondays 3 AM UTC) skips any row with `adminLocked=true`.

### Roles

| Role | Description |
|---|---|
| `USER` | Standard user — default for all registrations |
| `TOURNAMENT_SUPPORT` | Tournament management only via admin panel |
| `ADMIN` | Full admin access |

Change roles: Admin Dashboard → Users tab → role dropdown → select new role (immediate).

---

## 11. Deployment

### Prod deploy checklist

1. Ensure all repos on `main`
2. Run Cypress: `gh workflow run cypress.yml --repo sports4him12/golfsync-web -f branch=release`
3. Wait for Cypress to pass
4. Run: `./scripts/deploy.sh prod`

### Deploy script flags

```bash
./scripts/deploy.sh prod                    # full deploy with all gates
./scripts/deploy.sh prod --skip-cypress     # bypass Cypress gate
./scripts/deploy.sh prod --skip-ci-check    # bypass CI gate
./scripts/deploy.sh prod --skip-liquibase   # skip DB migrations
./scripts/deploy.sh prod --yes              # skip confirmation prompts
```

### Dev environment

Dev scales to 0 when not in use. Scale up/deploy/scale down via ECS Console or the deploy script.

---

## 12. Liquibase Migration Failures

1. **Check logs:** CloudWatch → `/golfsync-prod/liquibase` → most recent stream
2. **Lock stuck?** `DELETE FROM DATABASECHANGELOGLOCK WHERE ID = 1;`
3. **Re-run:** Use the `LiquibaseMigrateCommand` from CloudFormation Console → Stack outputs

---

## 13. Production Rollback

### Fast rollback (~2 min, via AWS Console)

1. ECS → Clusters → `golfsync-prod` → API service → Update
2. Change task definition revision to N-1 → Force new deployment
3. Repeat for Web service
4. Wait for stabilization

### Full rollback (via git tags)

```bash
git tag -l 'prod-*' | sort -r | head -20
git checkout prod-2026-04-16-1840
git submodule update --init --recursive
./scripts/deploy.sh prod --skip-cypress
```

**DB caveat:** Liquibase migrations are not auto-reversible. Additive changes are safe; destructive changes need a manual rollback script.

---

## 14. Escalation Checklist

1. Get user's **email** and **approximate time** of issue
2. **Admin Dashboard → Users tab** — check membership, ban, role
3. **CloudWatch → `/golfsync-prod/api`** — search by userId
4. **Stripe Dashboard** — if payment-related
5. **SES Console** — if email delivery suspect
6. **CloudWatch Dashboard → `golfsync-prod`** — system health

**Support email:** `support@golfsync.io`

---

## 15. Key Infrastructure

| Resource | Where |
|---|---|
| ALB | EC2 → Load Balancers |
| ECS Cluster | ECS → `golfsync-prod` |
| RDS | RDS → `golfsync-prod` instance |
| WAF | WAF → `GeoRestrictWaf` (US-only) |
| Route53 Health Check | Route53 → Health Checks |
| SES | SES → `golfsync.io` domain |
| Secrets Manager | Secrets Manager → `golfsync-prod/*` |
| CloudWatch Dashboard | CloudWatch → Dashboards → `golfsync-prod` |
