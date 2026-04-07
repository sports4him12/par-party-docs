# GolfSync — Lean Canvas

---

## 1. Problem

**Top 3 problems golfers face today:**

1. **Organizing a group round is friction-heavy** — coordinating schedules, finding available tee times, and confirming players requires back-and-forth across text messages, phone calls, and multiple booking sites.
2. **No single place to manage your golf social life** — tee time booking, friend networks, direct messaging, and tournament discovery exist in separate, disconnected tools.
3. **Tournament discovery is manual and unreliable** — finding local amateur tournaments requires scouring club websites and bulletin boards; listings go stale and are hard to verify.

**Existing Alternatives:**
- GolfNow / TeeOff — tee time booking only, no social or tournament layer
- Group chats (iMessage, WhatsApp) — unstructured coordination with no booking integration
- Club websites / USGA / local flyers — fragmented tournament information with no reminders or reporting

---

## 2. Customer Segments

**Primary: Social Golfers (US Market)**
- Golfers of any skill level who play with friends regularly (handicap tracking optional)
- Value convenience of organized group bookings over solo rounds
- Tech-comfortable — comfortable using AI-assisted tools for recommendations
- Age range: broad (any golfer with a smartphone)

**Secondary: Amateur Tournament Players**
- Golfers who actively seek local tournaments (Stroke Play, Scramble, Match Play, etc.)
- Want advance notice (reminders 1–30 days out) and reliable, community-verified listings

**Early Adopters:**
- Golfers who are already coordinating rounds via group texts but want a better way
- The person in the friend group who organizes every outing ("the planner")

---

## 3. Unique Value Proposition

**Single, clear, compelling message:**

> **"Book your next round, invite your crew, and never miss a tournament — all in one place."**

**High-level concept:** GolfSync is the social layer that golf has always been missing — combining AI-assisted tee time search, a friend network, direct messaging, and local tournament discovery into one purpose-built platform.

**Differentiator:** AI-native booking experience (GPT-4o assistant can suggest courses, find available times, and help coordinate friends through natural conversation) combined with a community-moderated tournament board.

---

## 4. Solution

**Top 3 features:**

1. **AI-Assisted Round Booking**
   - Chat with GPT-4o assistant to describe what you want ("4 players, this Saturday, Richmond area")
   - Assistant surfaces matching courses and available tee times
   - Select a tee time → pre-fills round form → invite friends by username → confirm booking
   - Structured BPMN workflow guides the round through: Collect Details → Confirm → Await Invitees → Booked

2. **Friend Network & Direct Messaging**
   - Search users by username/name; send friend requests
   - Invite non-users by email (signup link auto-sent)
   - Message friends directly — unread badge on dashboard
   - Invite friends directly into a round from the booking flow

3. **Tournament Discovery & Reminders**
   - Geolocation-based discovery of nearby amateur tournaments
   - Filter by format, sort by date/distance/fee
   - Set email reminders 1, 3, 7, 14, or 30 days before an event
   - Community-report booked or invalid listings (admin-reviewed moderation)

---

## 5. Channels

**Acquisition:**
- **Direct registration** — `golfsync.io/register` (email-based sign-up)
- **Email invite** — existing users invite friends via email; non-users receive a GolfSync signup link
- **Round invitations** — players invited to a round who aren't yet on the platform get an invite
- **Landing page CTA** — "Get Started Free" → register → membership selection funnel

**Retention:**
- **Tournament reminders** — email notifications keep users returning around events (1–30 days before)
- **Friend activity** — friend requests, accepted invitations, and unread messages create re-engagement loops
- **Round workflow** — multi-step booking keeps users returning to progress and confirm rounds

**Support:**
- **support@golfsync.io** — direct email
- **In-app contact form** — `POST /api/support/contact` (name, email, subject, 10+ char message)
- **FAQ page** — 10 self-service answers covering password, booking, friends, payments, moderation

---

## 6. Revenue Streams

**Membership Subscriptions (primary):**

| Plan | Price | Savings | Description |
|------|-------|---------|-------------|
| Monthly | $9.99/month | — | Full access, cancel anytime |
| Annual | $79.99/year | 33% vs monthly | Best Value |

- Processed via **Stripe** (PaymentIntent API)
- Membership unlocks: round booking, friend network, tournament discovery, AI assistant

**Promo Code System:**
- `FREE` codes — 100% off, no payment info required (used for early adopter acquisition)
- `PERCENT` codes — percentage discount off any plan
- `FIXED` codes — fixed-dollar discount off any plan
- Codes support: max uses cap, expiry date, usage tracking
- Seeded code: **`FREEMEMBERSHIP`** (unlimited, never expires — beta/acquisition use)

**Future Revenue Potential (not yet implemented):**
- GolfNow tee time booking commission (integration is mocked; real swap-in is architected)
- Sponsored tournament listings
- Course/club partnerships

---

## 7. Cost Structure

**Third-Party Services:**

| Service | Purpose | Cost Model |
|---------|---------|------------|
| **OpenAI (GPT-4o)** | AI booking assistant | Per-token API pricing (variable) |
| **Stripe** | Payment processing | ~2.9% + $0.30 per transaction |
| **AWS ECS (Fargate)** | API + web + Camunda containers | Per-vCPU/GB-hour |
| **AWS RDS (MySQL)** | Primary database (multi-AZ in prod) | Instance-hour pricing |
| **AWS SES** | Transactional email (reminders, invites, support) | ~$0.10/1,000 emails |
| **AWS Secrets Manager** | Credential management | Per-secret/month |
| **AWS ACM + ALB** | TLS termination + load balancing | ALB per-hour + LCU pricing |
| **Camunda 7 (self-hosted)** | BPMN round booking workflow | Open-source (ECS hosting cost only) |
| **GolfNow** | Tee time data provider | Commission-based (when live) |
| **Nominatim/OSM** | Geolocation / reverse geocoding | Free (fair-use public API) |

**Estimated Monthly Infrastructure Cost:**
- Dev stack: ~$100–300/month (1 NAT gateway, single-AZ, minimal ECS tasks)
- Prod stack: ~$400–900/month (2 NAT gateways, multi-AZ RDS, 2+ ECS tasks per service)
- OpenAI: $0–500/month (usage-dependent)
- Stripe: Variable (% of membership revenue)

**Fixed Costs:**
- Domain registration (golfsync.io)
- AWS CDK-managed infrastructure (ECS, RDS, ALB, SES, Route53)

---

## 8. Key Metrics

**Acquisition:**
- Monthly new registrations
- Email invite conversion rate (invites sent → new signups)
- Referral / viral coefficient (users invited per active user per month)

**Activation:**
- Registration → membership conversion rate
- Time-to-first-round-booked
- Time-to-first-friend-added

**Retention:**
- Monthly Active Users (MAU)
- Monthly membership churn rate
- Rounds booked per active user per month

**Revenue:**
- Monthly Recurring Revenue (MRR)
- Annual Recurring Revenue (ARR)
- Average Revenue Per User (ARPU)
- Monthly vs Annual plan mix

**Engagement:**
- Tournament reminders set per user
- Messages sent per user per month
- AI assistant sessions per user per month
- Friend requests sent/accepted per user

**Health:**
- User reports filed (community moderation volume)
- Tournament listing report accuracy (accepted vs denied)
- Password expiry prompt rate (signals active users)

---

## 9. Unfair Advantage

1. **AI-native from day one** — GPT-4o assistant is core to the booking flow, not a bolt-on. Users who plan rounds through conversation will find the experience significantly more natural than form-based alternatives.

2. **Integrated social layer** — No other golf tee time platform combines booking + friend network + direct messaging + tournament discovery in a single product. Competitors own one piece; GolfSync connects all of them.

3. **Community-moderated tournament board** — User-reported listings with admin review creates a self-cleaning, trust-building data asset that improves over time without manual curation.

4. **BPMN-orchestrated round workflow** — Round booking is structured as a formal workflow (Camunda 7) rather than simple form submission. This enables reliable multi-step coordination across multiple players with clear state tracking.

5. **Network effects** — Each new user makes the platform more valuable for existing users: more friends to invite, more rounds to join, larger tournament-reporting community.

---

## Technical Stack Summary *(for investor/technical due diligence)*

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14, TypeScript, Tailwind CSS |
| Backend API | Spring Boot 3.4, Java 21, RESTful |
| AI | LangChain4J 0.31, OpenAI GPT-4o |
| Workflow | Camunda BPMN 7.22 (self-hosted) |
| Database | MySQL 8 (Liquibase migrations) |
| Auth | JWT (httpOnly cookie), BCrypt, 90-day rotation |
| Payments | Stripe PaymentIntent API |
| Email | AWS SES (prod) / Gmail SMTP (dev) |
| Infrastructure | AWS CDK v2 — ECS Fargate, RDS, ALB, SES, Route53, ACM |
| Security | Rate limiting, CORS, SameSite cookies, least-privilege IAM |

---

## Contact & Support

- **Support email:** support@golfsync.io
- **In-app contact form:** `/support`
- **Docs repo:** [sports4him12/golfsync-docs](https://github.com/sports4him12/golfsync-docs)
