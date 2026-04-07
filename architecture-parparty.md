# GolfSync — Architecture

## Overview

GolfSync is a full-stack golf social and membership application. Users chat with an AI assistant to plan rounds, search for tee times, invite friends, and message each other. Round bookings are orchestrated through a Camunda BPMN workflow engine. Users can subscribe to a GolfSync membership via Stripe (or a promo code). Tournament discovery includes a community reporting workflow where users flag booked or invalid tournaments for admin review.

---

## System Topology

```
┌────────────────────────────────────────────────────────────┐
│                        Browser / Client                    │
│              Next.js 14  ·  TypeScript  ·  Tailwind        │
│                        localhost:3000                      │
└─────────────────────────┬──────────────────────────────────┘
                          │  HTTP (proxied via next.config.mjs)
                          ▼
┌────────────────────────────────────────────────────────────┐
│                      golfsync-api                          │
│          Spring Boot 3.4.4  ·  Java 21  ·  :8080          │
│                                                            │
│  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │ Controllers│  │ Services │  │ Security │  │  AI     │  │
│  │  REST API  │  │ Business │  │JWT+BCrypt│  │LangChain│  │
│  └────────────┘  └──────────┘  └──────────┘  └─────────┘  │
│                       │                           │        │
│               ┌───────┴──────┐           ┌────────┘        │
│               ▼              ▼           ▼                 │
│          MySQL :3306   CamundaClient  OpenAI API           │
│          (golfsync DB)  (WebClient)   (GPT-4o)             │
│                                                            │
│          Stripe SDK (payment intents for membership)       │
└────────────────────────────────────────────────────────────┘
                          │
                          │  HTTP  engine-rest
                          ▼
┌────────────────────────────────────────────────────────────┐
│                     golfsync-server                        │
│        Camunda 7.22  ·  Spring Boot  ·  H2  ·  :8090      │
│                                                            │
│   BPMN:  golf-round-booking-process                        │
│          tournament-report-process                         │
│   Cockpit / Tasklist:  demo / demo                         │
│   H2 Console:  /h2-console                                 │
└────────────────────────────────────────────────────────────┘
```

---

## Repositories

| Module | Path | Port | Stack |
|---|---|---|---|
| `golfsync-server` | `GolfSyncApplication/golfsync-server` | 8090 | Java / Camunda 7.22 / H2 file DB |
| `golfsync-api` | `GolfSyncApplication/golfsync-api` | 8080 | Java 21 / Spring Boot 3.4.4 / MySQL |
| `golfsync-web` | `GolfSyncApplication/golfsync-web` | 3000 | TypeScript / Next.js 14 / Tailwind |

---

## Data Model

All tables are owned by **Liquibase** (`spring.jpa.hibernate.ddl-auto=none`). Migrations live in `src/main/resources/db/changelog/changes/`.

```
users                                               (001)
  id, username (unique), name, email (unique), password_hash,
  home_address, phone, handicap, role {USER|ADMIN}, created_at
  Seed: admin@golfsync.io / admin123

courses                                             (002)
  id, name, location, description, par, holes, created_at
  (5 courses seeded at startup by CourseService)

rounds                                              (003)
  id, creator_id → users, course_id → courses,
  round_date, round_time, max_players,
  status {PENDING|BOOKED|COMPLETED|CANCELLED},
  workflow_instance_id, notes, created_at

round_players                                       (004)
  round_id → rounds, user_id → users,
  status {INVITED|ACCEPTED|DECLINED}, responded_at

friend_requests                                     (005)
  id, requester_id → users, addressee_id → users,
  status {PENDING|ACCEPTED|DECLINED}, created_at

messages                                            (006)
  id, sender_id → users, recipient_id → users,
  body, sent_at, read_at

tournament_reports                                  (007)
  id, tournament_id, tournament_name,
  report_type {BOOKED|INVALID}, explanation,
  reporter_id → users, status {PENDING|ACCEPTED|DENIED},
  workflow_instance_id, decision_reason, created_at

promo_codes                                         (008)
  id, code (unique), discount_type {FREE|PERCENT|FIXED},
  discount_value, max_uses (0 = unlimited), used_count,
  expires_at, active, created_at
  Seed: FREEMEMBERSHIP (FREE, unlimited, never expires)

favorite_courses                                    (009)
  user_id → users, course_id → courses (composite PK)

tournament_reminders                                (010)
  id, user_id → users, tournament_id, tournament_name,
  tournament_date, reminder_days_before, created_at

users.password_changed_at                           (011)
  Column added to users table; default = created_at.
  AuthService enforces 90-day rotation (PARPARTY_PASSWORD_EXPIRY_DAYS).

users.banned + user_reports                         (012)
  users.banned TINYINT(1) — set to 1 when a user report is ACCEPTED by admin.
  Banned users cannot log in; their email cannot be re-registered.
  user_reports: id, reported_user_id → users, reporter_id → users,
    report_type {HARASSMENT|INAPPROPRIATE|FAKE_ACCOUNT|OTHER},
    explanation, status {PENDING|ACCEPTED|DENIED}, decision_reason, created_at
```

---

## Security

All API endpoints require a **Bearer JWT token** except:
- `POST /api/auth/**`
- `GET /api/courses`
- `GET /api/tournaments`
- `GET /api/payments/config`

### Auth Flow

```
POST /api/auth/register  or  POST /api/auth/login
        │
        ▼  [RateLimitFilter — 10 req/min per IP]
        │
        ▼
AuthService → BCryptPasswordEncoder → UserRepository
  • Banned users blocked at login and re-registration
  • Password expiry enforced (default 90 days) — PasswordExpiredException → 401 PASSWORD_EXPIRED
  • Email enumeration mitigated — generic message for duplicate email/banned
        │
        ▼
JwtUtil.generateToken(email, userId, role)
  → HMAC-SHA256, 24h expiry
  → Secret loaded from PARPARTY_JWT_SECRET env var (no insecure default)
        │
        ▼  { token, userId, username, role }
        │
Client stores token in localStorage (key: golfsync_token)
  • auth.ts checks exp claim on every read — stale tokens removed
  • api.ts auto-redirects to /login on any 401 (except PASSWORD_EXPIRED)
        │
Every subsequent request
        │
        ▼
JwtAuthenticationFilter (OncePerRequestFilter)
  reads Authorization: Bearer <token>
  → JwtUtil.validateToken()
  → UserDetailsServiceImpl.loadUserByUsername(email)
  → SecurityContextHolder.setAuthentication(...)
```

Admin endpoints (`/api/admin/**`) additionally require `ROLE_ADMIN`, enforced by `@PreAuthorize("hasRole('ADMIN')")` on `AdminController`. Impersonation actions are audit-logged at WARN level with admin identity, target userId, and username.

### CORS
Allowed origin is configured via `PARPARTY_CORS_ALLOWED_ORIGIN` env var (default: `http://localhost:3000`). The wildcard `*` is never used.

### Rate Limiting
`RateLimitFilter` (applied before `JwtAuthenticationFilter`) enforces a sliding-window limit of 10 requests/minute per IP on `/api/auth/login` and `/api/auth/register`. Exceeding the limit returns `429 RATE_LIMIT_EXCEEDED`. Configurable via `golfsync.rate-limit.max-requests` and `golfsync.rate-limit.enabled` properties (disabled in tests).

---

## REST API Reference

### Authentication — `/api/auth`

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/auth/register` | None | `{username, name, email, password, homeAddress?, phone?, handicap?}` | `201 {token, userId, username, role}` |
| POST | `/api/auth/login` | None | `{email, password}` | `200 {token, userId, username, role}` — `401 PASSWORD_EXPIRED` if password is older than 90 days |
| POST | `/api/auth/change-password` | None | `{email, currentPassword, newPassword}` | `204` |

### Users — `/api/users`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/users/me` | User | Own profile |
| PUT | `/api/users/me` | User | Update own profile: `{name?, homeAddress?, phone?, handicap?}` — typed `UpdateProfileRequest` DTO |
| GET | `/api/users/search?q=` | User | Search by username or name |
| GET | `/api/users/{id}` | User | Fetch user by ID |
| POST | `/api/users/{id}/report` | User | Report a user: `{reportType, explanation}` |

### Courses — `/api/courses`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/courses` | **None** | List all seeded courses |
| GET | `/api/courses/{id}` | User | Get course detail |
| POST | `/api/courses` | Admin | Create course |

### Rounds — `/api/rounds`

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/api/rounds` | User | Create round; body includes `inviteeUsernames[]` |
| GET | `/api/rounds` | User | All rounds where user is creator or invited |
| GET | `/api/rounds/{id}` | User | Round detail with player list |
| PUT | `/api/rounds/{id}/respond` | User | Accept/decline invite: `{accept: true\|false}` |
| DELETE | `/api/rounds/{id}` | User (creator) | Sets status → CANCELLED |

### Workflow — `/api/rounds/{roundId}/workflow`

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/start` | User | Starts Camunda `golf-round-booking-process` |
| GET | `/status` | User | Live process instance from Camunda |
| GET | `/tasks` | User | Active user tasks for this round |
| POST | `/tasks/{taskId}/complete` | User | Complete a Camunda task; body = `{variables}` |

### Tournaments — `/api/tournaments`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/tournaments` | **None** | `?lat=&lng=&radiusMiles=` — returns nearby tournaments (accepted/hidden filtered out) |
| POST | `/api/tournaments/report` | User | Submit report: `{tournamentId, tournamentName, reportType, explanation}` |

### Friends — `/api/friends`

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/api/friends/request` | User | Send request: `{addresseeId}` |
| PUT | `/api/friends/{requestId}/respond` | User | Accept/decline: `{accept: true\|false}` |
| GET | `/api/friends` | User | List accepted friends |
| GET | `/api/friends/pending` | User | Incoming pending requests |

### Messages — `/api/messages`

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/api/messages` | User | Send DM: `{recipientId, body}` — friends only |
| GET | `/api/messages/{otherUserId}` | User | Full conversation ordered by `sent_at` |
| PUT | `/api/messages/{messageId}/read` | User (recipient) | Sets `read_at`; returns `204` |
| GET | `/api/messages/unread/count` | User | Unread message count badge |

### Payments / Membership — `/api/payments`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/payments/config` | **None** | Returns `{enabled, publishableKey, monthlyPriceCents, annualPriceCents}` |
| GET | `/api/payments/validate-promo` | User | `?code=&plan=` — validates a promo code without consuming it |
| POST | `/api/payments/membership/create-intent` | User | `{plan: MONTHLY\|ANNUAL, promoCode?}` → `{clientSecret, amountCents, free}` |

When `STRIPE_SECRET_KEY` env var is not set, `config.enabled` is `false` and `create-intent` returns 503. The frontend shows a "Payments Coming Soon" placeholder.

When a FREE promo code is applied, `create-intent` returns `{free: true, amountCents: 0, clientSecret: null}` — no Stripe payment is needed.

### GolfNow Tee Times — `/api/golfnow`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/golfnow/tee-times` | User | `?lat=&lng=&date=&players=` — mock tee times |
| POST | `/api/golfnow/book` | User | `{teeTimeId, roundId}` → `{confirmationNumber}` |

### AI Chat — `/api/ai`

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/api/ai/chat` | User | `{message, conversationId}` → `{response}` |

### Admin — `/api/admin`

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/admin/users` | Admin | All users |
| PUT | `/api/admin/users/{id}` | Admin | Update user fields via `UpdateProfileRequest` DTO |
| DELETE | `/api/admin/users/{id}` | Admin | Hard-delete via `UserService.deleteUser()` |
| GET | `/api/admin/rounds` | Admin | All rounds |
| PUT | `/api/admin/rounds/{id}` | Admin | Change round status |
| DELETE | `/api/admin/rounds/{id}` | Admin | Cancel round via `RoundService.adminDeleteRound()` |
| POST | `/api/admin/impersonate/{userId}` | Admin | Returns scoped JWT for target user — **audit-logged at WARN** |
| GET | `/api/admin/processes` | Admin | Live Camunda task info for all rounds with a workflow — exceptions logged (not swallowed) |
| GET | `/api/admin/bpmn` | Admin | BPMN XML fetched live from Camunda |
| GET | `/api/admin/tournament-reports` | Admin | All tournament reports |
| POST | `/api/admin/tournament-reports/{id}/decide` | Admin | `{decision: ACCEPT\|DENY, decisionReason}` |
| GET | `/api/admin/promo-codes` | Admin | All promo codes |
| POST | `/api/admin/promo-codes` | Admin | Create: `{code, discountType, discountValue, maxUses, expiresAt?}` |
| DELETE | `/api/admin/promo-codes/{id}` | Admin | Deactivate (soft delete) |
| GET | `/api/admin/user-reports` | Admin | All user flag reports |
| POST | `/api/admin/user-reports/{id}/decide` | Admin | `{decision: ACCEPTED\|DENIED, decisionReason}` — ACCEPTED bans the reported user |

---

## Membership & Payments

### Flow (Stripe active)

```
User selects plan (MONTHLY $9.99 / ANNUAL $79.99)
        │
Optional: enter promo code → GET /api/payments/validate-promo
        │
        ▼
POST /api/payments/membership/create-intent
  → PromoCodeService.validate() → applies discount
  → If amount == 0 (FREE promo): return {free: true} — skip Stripe
  → Else: Stripe PaymentIntent.create(amount, currency, email)
        │
        ▼
Frontend renders Stripe Elements PaymentElement
  → stripe.confirmPayment() → redirect to /membership/success
        │
On payment confirmation, promoCode.usedCount++
```

### Promo Code Types

| Type | Behaviour |
|---|---|
| `FREE` | Sets amount to 0; Stripe skipped entirely |
| `PERCENT` | Reduces amount by `discountValue`% |
| `FIXED` | Reduces amount by `discountValue` cents |

Seeded code: **`FREEMEMBERSHIP`** (FREE, unlimited uses, never expires).

---

## Tournament Reporting Workflow

Users can flag a tournament as **BOOKED** or **INVALID** from the dashboard. This starts a Camunda workflow (`tournament-report-process`) for admin review.

```
User submits report (dashboard → Report button)
        │
        ▼
POST /api/tournaments/report
  → TournamentReportService.submitReport()
  → Saves TournamentReport (status=PENDING)
  → CamundaClient.startProcess("tournament-report-process")
        │
Admin reviews in /admin → Tournament Reports tab
        │
POST /api/admin/tournament-reports/{id}/decide
  → decision = ACCEPT:
      TournamentReport.status = ACCEPTED
      TournamentController filters this tournamentId from future GET /api/tournaments responses
  → decision = DENY:
      TournamentReport.status = DENIED
      EmailService.sendTournamentDenialEmail(reporter, reason)
      (graceful failure if SMTP not configured)
```

---

## AI Assistant

Built with **LangChain4J 0.31.0** using OpenAI GPT-4o.

```
AiController  POST /api/ai/chat
      │
      ▼
BookingAssistant  (@AiService interface)
      │
      ├── ChatMemoryProvider (InMemoryChatMemoryStore)
      │     keyed by conversationId → MessageWindowChatMemory (20 messages)
      │
      └── BookingAgentTools  (@Tool methods)
            ├── getCourses()      → CourseService.findAll()
            └── searchFriends()   → UserService.searchUsers(query)
```

Memory is in-process per `conversationId` and does not persist across API restarts.

---

## Camunda Workflow Integration

`golfsync-api` communicates with `golfsync-server` exclusively via **HTTP** (Camunda `engine-rest`). No embedded engine.

```
golfsync-api  (CamundaClient.java — WebClient)
      │  Base URL: http://localhost:8090/engine-rest
      ▼
golfsync-server  (Camunda 7.22)
      ├─  POST /process-definition/key/{key}/start
      ├─  GET  /task?processInstanceId={id}
      ├─  POST /task/{id}/complete
      ├─  GET  /process-instance/{id}
      ├─  DELETE /process-instance/{id}
      └─  GET  /process-definition/key/{key}/xml
```

### BPMN: `golf-round-booking-process`

```
[Start] → [Collect Round Details] → [Confirm Details] → <Confirmed?> ─No─┐
           (User Task)               (User Task)          Gateway          │
                                                              │Yes          │
                                                              ▼            │
                                                   [Await Invitee Responses]◄┘
                                                       (User Task)
                                                              │
                                                              ▼
                                                   [Book Round (Service Task)] → [End]
```

### BPMN: `tournament-report-process`

```
[Start] → [Review Tournament Report] → [End]
           (User Task — admin)
           review-tournament-report
```

---

## GolfNow Integration

Tee time booking is accessed through an interface/mock pair for easy real-API swap.

```
GolfNowController → GolfNowService → GolfNowClient (interface)
                                          │
                                          └── GolfNowClientMock (@Primary)
                                                Returns fake tee times
                                                Confirmation: "GNOW-XXXXXXXX"
```

Round booking is paid **directly with the tee time provider** (GolfNow), not through GolfSync. GolfSync membership payments are separate (Stripe).

---

## Frontend Architecture

```
Next.js 14 App Router (TypeScript)
│
├── app/
│   ├── page.tsx                 Landing page (public)
│   ├── login/page.tsx           Login → admin goes to /admin, user to /dashboard
│   ├── register/page.tsx        Registration → redirects to /membership
│   ├── dashboard/page.tsx       Stats, rounds, tournaments, hamburger nav
│   ├── membership/
│   │   ├── page.tsx             Plan selector, promo code, Stripe checkout
│   │   └── success/page.tsx     Post-payment confirmation
│   ├── rounds/
│   │   ├── new/page.tsx         AI chat + tee time search sidebar
│   │   └── [id]/page.tsx        Round detail, workflow tasks, accept/decline
│   ├── change-password/page.tsx Pre-fills email from ?email= query param; redirects to /login
│   ├── friends/page.tsx         User search, friend requests, friends list + report modal
│   ├── messages/
│   │   ├── page.tsx             Conversation list with unread badges
│   │   └── [userId]/page.tsx    DM thread
│   └── admin/page.tsx           Users · Rounds · Tournament Reports · User Reports · Promo Codes tabs
│
├── components/
│   └── Toast.tsx                ToastContainer component (success/error/info toasts)
│
├── hooks/
│   └── useToast.ts              useToast() hook — showToast(message, type), auto-dismiss 4s
│
├── lib/
│   ├── api.ts                   All fetch calls (Bearer token injected; auto-redirect on 401)
│   ├── auth.ts                  localStorage token helpers, JWT exp check, stale token removal
│   └── types.ts                 TypeScript interfaces: User, Round, TeeTime, MembershipIntent,
│                                PromoValidation, UserReport, TournamentReport, PaymentConfig, …
│
└── tailwind.config.ts           GolfSync brand palette
      primary: #D03027 (burnt orange/red)
      navy:    #004879
```

### API Proxy

`next.config.mjs` rewrites `/api/*` → `http://localhost:8080/api/*` (dev only).

### Auth State

`lib/auth.ts` decodes the JWT payload client-side to extract `userId`, `email`, and `role` without an extra API call. Admins are redirected to `/admin` on login.

---

## Local Development Setup

### Prerequisites

- Java 21 (`/opt/homebrew/opt/openjdk@21`)
- Maven 3.9+ (`/opt/homebrew/bin/mvn`)
- MySQL at `localhost:3306` (credentials via `DB_PASSWORD` env var)
- Node.js 18+

### Start Sequence

```bash
# 1. Camunda engine (terminal 1)
cd GolfSyncApplication/golfsync-server
JAVA_HOME=/opt/homebrew/opt/openjdk@21 mvn spring-boot:run
# → http://localhost:8090  Cockpit/Tasklist: demo/demo

# 2. Spring Boot API (terminal 2)
cd GolfSyncApplication/golfsync-api
export DB_PASSWORD=changeme           # matches your MySQL root password
export PARPARTY_JWT_SECRET=$(openssl rand -hex 32)
export OPENAI_API_KEY=sk-...
# Optional for live payments:
# export STRIPE_SECRET_KEY=sk_test_...
# export STRIPE_PUBLISHABLE_KEY=pk_test_...
JAVA_HOME=/opt/homebrew/opt/openjdk@21 mvn spring-boot:run
# → http://localhost:8080  Liquibase migrations run on startup

# 3. Next.js frontend (terminal 3)
cd GolfSyncApplication/golfsync-web
npm install && npm run dev
# → http://localhost:3000

# Admin login: admin@golfsync.io / admin123
```

### MySQL Database

The `golfsync` database must exist before starting the API:

```sql
CREATE DATABASE IF NOT EXISTS golfsync
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Liquibase applies all 8 changesets automatically on startup, including the admin seed user and `FREEMEMBERSHIP` promo code.

### Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `DB_PASSWORD` | **Yes** | MySQL root password |
| `PARPARTY_JWT_SECRET` | **Yes** | HMAC-SHA256 signing key (min 256 bits); generate with `openssl rand -hex 32` |
| `OPENAI_API_KEY` | Yes (AI chat) | GPT-4o API key |
| `PARPARTY_CORS_ALLOWED_ORIGIN` | No | Frontend origin for CORS (default: `http://localhost:3000`) |
| `STRIPE_SECRET_KEY` | No | Activates live Stripe payments |
| `STRIPE_PUBLISHABLE_KEY` | No | Sent to frontend via `/api/payments/config` |
| `MAIL_HOST` | No | SMTP host (default: smtp.gmail.com) |
| `MAIL_USERNAME` | No | SMTP username |
| `MAIL_PASSWORD` | No | SMTP password |
| `MAIL_FROM` | No | From address (default: noreply@golfsync.io) |

Copy `.env.example` → `.env` and fill in values before running `docker-compose up`.

---

## Testing

### Unit Tests (`@ExtendWith(MockitoExtension.class)`)

Service-layer tests with mocked dependencies. No Spring context.

### Slice Tests (`@WebMvcTest`)

Controller-layer tests. Each test class extends `ControllerTestBase` which provides `@MockitoBean JwtUtil` and `@MockitoBean UserDetailsServiceImpl`. `TestSecurityConfig` replaces the production `SecurityFilterChain` with a CSRF-disabled equivalent that mirrors the same permit rules.

### Acceptance Tests (`@SpringBootTest(RANDOM_PORT)`)

Full integration tests against an in-memory H2 database. `CamundaClient` and `BookingAssistant` are `@MockitoBean`. Database cleared via `@BeforeEach`.

| Class | Tests | Coverage |
|---|---|---|
| `AuthAcceptanceTest` | 10 | Register, login, duplicates, JWT access |
| `RoundLifecycleAcceptanceTest` | 15 | Full round CRUD, invites, workflow |
| `FriendMessagingAcceptanceTest` | 14 | Friend requests, DMs, access rules |
| `AdminAcceptanceTest` | 12 | Admin CRUD, impersonation, role enforcement |
| `GolfNowAcceptanceTest` | 8 | Tee time search/book, auth guards |
| `SecurityAcceptanceTest` | 16 | Public vs protected, tampered JWTs, no password leaks |
| `TournamentAcceptanceTest` | varies | Tournament list, report submission |

JaCoCo enforces **80% instruction coverage** on service and controller classes (enforced at `mvn verify`).

---

## Key Design Decisions

**Camunda decoupled via HTTP** — the API never imports Camunda Java libraries. All process interactions go through `CamundaClient.java` (WebClient), making the workflow engine replaceable and independently deployable.

**GolfNow behind an interface** — `GolfNowClient` is an interface. The mock is `@Primary` today. Swapping to the real GolfNow sandbox is a single-class addition with no controller or service changes. Round payments go to the tee time provider directly — GolfSync never handles round booking payments.

**Stripe graceful degradation** — when `STRIPE_SECRET_KEY` is not set, `StripeConfig.isConfigured()` returns false. The API returns 503 on intent creation; the frontend shows "Payments Coming Soon." No code path change needed to activate payments.

**Promo codes skip Stripe for FREE type** — when `amountCents == 0` after promo, `PaymentService` returns `{free: true}` and skips `PaymentIntent.create()` entirely (Stripe requires amount > 0).

**Tournament hiding via DB set** — tournaments are static in-memory data, not DB rows. When an admin accepts a report, the `tournament_reports` table records the `tournamentId`. `TournamentController` calls `TournamentReportService.getHiddenTournamentIds()` and filters them before every response.

**JWT stateless security** — no sessions. The token carries `userId`, `email`, and `role`, decoded server-side (filter) and client-side (auth.ts) without a round trip.

**Messaging gated by friendship** — `MessageService.sendMessage` calls `FriendService.areFriends()` before persisting. Only accepted friends can exchange messages.

**Liquibase-owned schema** — no `ddl-auto=create` or `update`. All schema changes go through versioned changesets in `db/changelog/changes/` (001–012).

**Typed profile updates** — `UserService.updateProfile()` accepts `UpdateProfileRequest` DTO instead of `Map<String, Object>`, preventing arbitrary field injection.

**Service-layer deletes** — admin delete endpoints call `UserService.deleteUser()` and `RoundService.adminDeleteRound()` rather than accessing repositories directly, keeping business logic in the service layer.

**Password rotation** — `AuthService.login()` rejects passwords older than `golfsync.password.expiry-days` (default 90) with `PasswordExpiredException` → HTTP 401 `PASSWORD_EXPIRED`. The frontend redirects to `/change-password`. `POST /api/auth/change-password` resets the `password_changed_at` timestamp.

**User banning** — when an admin accepts a `UserReport`, `UserReportService.decide()` sets `user.banned = true`. Banned users receive `403` on login and their email is blocked from re-registration with a generic message to avoid confirming the ban state to the registrant.

**No hardcoded secrets** — `DB_PASSWORD` and `PARPARTY_JWT_SECRET` have no insecure defaults; the application fails fast on startup if either is missing. `.env.example` documents all required variables; `.env` is gitignored.

**Email graceful failure** — `EmailService` wraps `mailSender.send()` in a try-catch. Denial emails fail silently (logged) when SMTP is not configured, so the workflow always completes.
