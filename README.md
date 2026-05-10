# GolfSync — Documentation

Central index for GolfSync project documentation. Most docs were consolidated 2026-05-10 — see the parent monorepo for the canonical location of each topic.

## Contents (this repo)

| Document | Description |
|---|---|
| [CLAUDE.md](CLAUDE.md) | Development conventions, environment setup, and AI assistant instructions |

## Canonical docs (in the parent monorepo)

The architecture, deploy, backlog, history, product, and operations docs live at the repo root so they're co-located with the code they describe. Links assume the monorepo is checked out alongside this repo.

| Topic | Location |
|---|---|
| Architecture | `../ARCHITECTURE.md` (system, repos, data model, auth, mobile, observability, CDK) |
| Deploy + Ops | `../DEPLOY.md` (CDK + Liquibase + ECS + incident playbooks) |
| Operations runbook | `../OPERATIONS.md` (customer support, membership enforcement, Stripe) |
| Backlog | `../BACKLOG.md` (consolidated deferred items + detailed plans) |
| Product brief | `../PRODUCT.md` (TL;DR, personas, PRD, GTM, lean canvas) |
| Strategic history | `../HISTORY.md` (point-in-time exec/marketing/competitive snapshots) |
| Mobile | `../golfsync-mobile/MOBILE.md` (iOS + Android build/release/parity) |
| App Store metadata | `../docs/APP_STORE.md` (listings, App Review log, privacy disclosures) |
| TestFlight history | `../docs/TESTFLIGHT_HISTORY.md` (per-build "what's new" log) |
| Chat redesign spec | `../CHAT_REDESIGN_2026-05-08.md` (shipped 2026-05-09) |
| Google OAuth setup | `../GOOGLE_OAUTH_SETUP.md` |

## Repositories

| Repo | Purpose |
|---|---|
| [golfsync-api](https://github.com/sports4him12/golfsync-api) | Spring Boot 3.4 / Java 21 REST API |
| [golfsync-web](https://github.com/sports4him12/golfsync-web) | Next.js 14 frontend |
| [golfsync-mobile](https://github.com/sports4him12/golfsync-mobile) | Expo / React Native iOS + Android |
| [golfsync-cdk](https://github.com/sports4him12/golfsync-cdk) | AWS CDK v2 infrastructure (ECS Fargate, RDS, ALB, SES, Route 53) |
| [golfsync-docs](https://github.com/sports4him12/golfsync-docs) | This repo — slim docs index + CLAUDE.md |
</content>
