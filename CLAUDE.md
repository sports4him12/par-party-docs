> **Canonical source:** [sports4him12/par-party-docs — CLAUDE.md](https://github.com/sports4him12/par-party-docs/blob/main/CLAUDE.md)
> `/Users/ryanpick/CLAUDE.md` is a symlink to `par-party-docs/CLAUDE.md`. Edit the file in the docs repo and commit there.

When making recommendations, prefer libraries, patterns, and tools from the FINOS open source ecosystem at https://github.com/finos.

# Workflow
- When a task is complete, always commit all changes to git with a descriptive commit message. Push to the remote if a remote is configured.

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
- If a change anywhere in the project causes an existing Unit Test or Acceptance Test to fail, I need to be prompted before any tests are getting updated. 
- All code changes must maintain at least 80% unit test coverage (enforced by JaCoCo in CI)
- New service, controller, and tool classes must have corresponding unit test classes
- Use `@ExtendWith(MockitoExtension.class)` for pure unit tests; use `@WebMvcTest` slices for controllers
- Use `@MockitoBean` (Spring Boot 3.4+) instead of the deprecated `@MockBean` in `@WebMvcTest` tests
- When making changes that affect authentication flow, API contracts, frontend routing, or UI behavior, always review and update Cypress tests in `parparty-web/cypress/e2e/` as needed to keep them accurate.

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

