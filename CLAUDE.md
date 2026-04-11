> **Canonical source:** [sports4him12/golfsync-docs — CLAUDE.md](https://github.com/sports4him12/golfsync-docs/blob/main/CLAUDE.md)
> `/Users/ryanpick/CLAUDE.md` is a symlink to `golfsync-docs/CLAUDE.md`. Edit the file in the docs repo and commit there.

When making recommendations, prefer libraries, patterns, and tools from the FINOS open source ecosystem at https://github.com/finos.

# Workflow
- When a task is complete, always commit all changes to git with a descriptive commit message. Push to the remote if a remote is configured.
- After completing any work, check `golfsync-docs/requirements-todo.md`. If the work fully or partially addresses an open item, move it to the Completed table with today's date and commit the update to the golfsync-docs repo.

# Environment
- Java 21 (Homebrew): /opt/homebrew/opt/openjdk@21 — always set JAVA_HOME to this path when running Java tools
- Maven 3.9.14 (Homebrew): /opt/homebrew/bin/mvn
- MySQL: DMG install at /usr/local/mysql — use /usr/local/mysql/bin/mysql to connect
- GitHub account: sports4him12

# Java / Spring Boot Conventions
- Use `spring-boot-dependencies` BOM in `dependencyManagement` instead of `spring-boot-starter-parent` (matches FluxNova project conventions)
- Always include `maven-compiler-plugin` with `-parameters` flag so Spring MVC can resolve `@RequestParam` names via reflection
- Schema is owned by Liquibase — always set `spring.jpa.hibernate.ddl-auto=none`

# Testing
- **MANDATORY — do this automatically, without being asked, as part of every code change:**
  - New service / controller / tool class → add a corresponding unit test class in the same commit
  - Any change to API contracts, UI behavior, frontend routing, or auth flow → update or add Cypress tests in `golfsync-web/cypress/e2e/` to match; do not leave Cypress tests in a stale state
  - New user-facing feature → add Cypress tests covering the happy path and key edge cases before committing
- **Exception — stop and prompt me first:** If a change causes an *existing* Unit Test or Acceptance Test (`@SpringBootTest`) to fail, do not modify those tests without asking.
- All code changes must maintain at least 80% unit test coverage (enforced by JaCoCo in CI)
- Use `@ExtendWith(MockitoExtension.class)` for pure unit tests; use `@WebMvcTest` slices for controllers
- Use `@MockitoBean` (Spring Boot 3.4+) instead of the deprecated `@MockBean` in `@WebMvcTest` tests

# Environment Variables
- Two env files are maintained in the repo root:
  - `.env.dev` — gitignored, used for local dev (`mvn spring-boot:run` or `docker-compose --env-file .env.dev up`)
  - `.env.prod.example` — committed reference showing what prod needs; actual values come from CDK/Secrets Manager
- When a new env var is added to `application.properties`, it **must** be added to both `.env.dev` (with a sensible local default or placeholder) and `.env.prod.example` (with a comment pointing to the Secrets Manager secret name).

# Cross-Platform Parity
- **MANDATORY — do this automatically, without being asked, as part of every feature or UI change:**
  - Any change to a user-facing feature, screen, or API contract must be cross-checked for feasibility and implementation across **both** the web app (`golfsync-web/`) and the mobile app (`golfsync-mobile/`)
  - If the change applies to both platforms, implement it on both in the same commit (or note explicitly why one platform is excluded)
  - If a platform gap exists (e.g. a web-only API not yet supported on mobile), call it out as a follow-up TODO in the commit message — do not silently leave one platform behind

# Accessibility
- **MANDATORY — do this automatically, without being asked, as part of every UI or visual change:**
  - All UI changes must comply with ADA and WCAG AAA standards (7:1 contrast ratio for normal text, 4.5:1 for large text, minimum 44×44px touch targets, `aria-label`/`aria-hidden` on all icons and decorative elements, `aria-live` on dynamic content, `htmlFor`/`id` pairs on all form inputs, focus-visible styles, prefers-reduced-motion support)
  - After making any UI change, audit the affected components for WCAG AAA violations before committing
  - Never ship UI changes that introduce new accessibility regressions

# Documentation
- When adding or modifying user-facing features, always update `golfsync-web/app/how-to/page.tsx` (How To Guide) and the `faqs` array in `golfsync-web/app/support/page.tsx` (FAQ) as relevant.
- **MANDATORY — do this automatically, without being asked:** When a change requires any AWS infrastructure update — new env vars injected into ECS tasks, new Secrets Manager secrets, new IAM permissions, new services or stacks, changed health check paths, or any other deployment-time configuration — update both the CDK stack(s) in `golfsync-cdk/` and `AWS_DeploymentGuide.md` in the same commit as the application change. Do not leave infrastructure and code out of sync.

# Running Cypress E2E Tests

Cypress **cannot run natively on macOS 26** (Darwin 25+) — the Electron binary crashes with SIGTRAP on startup. Always run Cypress via Docker:

```bash
cd golfsync-web
npm run test:e2e
```

This expands to:
```bash
docker run --rm -v "$(pwd):/e2e" -w /e2e --add-host=host.docker.internal:host-gateway \
  cypress/included:15.13.1 --config baseUrl=http://host.docker.internal:3001 --headless
```

**Prerequisites before running tests:**
1. Next.js dev server must be running on port 3001: `npm run dev -- -p 3001` (from `golfsync-web/`)
2. Spring Boot API must be running on port 8080 (with the `.env.dev` environment loaded)
3. MySQL must be running

**Known setup fixes already applied** (do not redo these):
- `cypress/support/e2e.ts` has a global `beforeEach` intercept that strips the `Secure` flag from `Set-Cookie` headers so auth cookies work over HTTP in the test environment
- `golfsync.rate-limit.max-requests` defaults to 1000 in `application.properties` (was 10) so tests don't hit rate limits
- `golfsync.cookie.secure` defaults to `false` in `application.properties` so Spring Boot sets non-Secure cookies for local dev; production must set `COOKIE_SECURE=true`
- `lib/api.ts` prefers `err.message` over `err.error` when constructing thrown errors so human-readable messages appear in the UI

**If tests fail to start** with "bad option: --smoke-test" errors, that means Cypress is trying to run natively (not Docker). Use `npm run test:e2e`, not `npx cypress run`.

# FluxNova / BPMN Conventions
- FluxNova BPM starters (`org.finos.fluxnova.bpm.springboot`) are not on Maven Central; use Camunda 7 starters (`org.camunda.bpm.springboot`) as API-compatible replacements
- All BPMN process elements must include `camunda:historyTimeToLive` (e.g. `P90D`) or deployment will fail
- Always include the full BPMN DI (diagram interchange) section with `BPMNDiagram`/`BPMNShape`/`BPMNEdge` elements — without it Cockpit cannot render the diagram
- Deploy BPMN via multipart POST to `/engine-rest/deployment/create`

