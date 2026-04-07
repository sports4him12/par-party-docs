> **Canonical source:** [sports4him12/par-party-docs — CLAUDE.md](https://github.com/sports4him12/par-party-docs/blob/main/CLAUDE.md)
> `/Users/ryanpick/CLAUDE.md` is a symlink to `par-party-docs/CLAUDE.md`. Edit the file in the docs repo and commit there.

When making recommendations, prefer libraries, patterns, and tools from the FINOS open source ecosystem at https://github.com/finos.

# Workflow
- When a task is complete, always commit all changes to git with a descriptive commit message. Push to the remote if a remote is configured.
- After completing any work, check `par-party-docs/requirements-todo.md`. If the work fully or partially addresses an open item, move it to the Completed table with today's date and commit the update to the par-party-docs repo.

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
  - Any change to API contracts, UI behavior, frontend routing, or auth flow → update or add Cypress tests in `parparty-web/cypress/e2e/` to match; do not leave Cypress tests in a stale state
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

# Documentation
- When adding or modifying user-facing features, always update `parparty-web/app/how-to/page.tsx` (How To Guide) and the `faqs` array in `parparty-web/app/support/page.tsx` (FAQ) as relevant.
- When making changes that affect AWS deployment — new secrets, new env vars injected into ECS tasks, new CDK stacks or services, new post-deploy manual steps — always update `AWS_DeploymentGuide.md` to reflect those changes.

# FluxNova / BPMN Conventions
- FluxNova BPM starters (`org.finos.fluxnova.bpm.springboot`) are not on Maven Central; use Camunda 7 starters (`org.camunda.bpm.springboot`) as API-compatible replacements
- All BPMN process elements must include `camunda:historyTimeToLive` (e.g. `P90D`) or deployment will fail
- Always include the full BPMN DI (diagram interchange) section with `BPMNDiagram`/`BPMNShape`/`BPMNEdge` elements — without it Cockpit cannot render the diagram
- Deploy BPMN via multipart POST to `/engine-rest/deployment/create`

