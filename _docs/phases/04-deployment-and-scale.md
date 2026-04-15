# Phase 04 — Deployment + Scale (DevOps + distributed realtime)

## Goal
Ship a **deployable**, horizontally-scalable version: Dockerized services, CI/CD, AWS infrastructure, and Redis-backed Socket.IO scaling.

## Scope
- Local dev stays simple (Compose), production deploy becomes repeatable.
- API can run multiple instances and still broadcast messages/presence via Redis.
- CI runs lint/test/build; deployment pushes images and updates runtime.

## Deliverables
- **Docker**: multi-stage builds for web + api; non-root images
- **CI**: GitHub Actions pipeline for lint/test/build, image push
- **AWS**: RDS (Postgres), ElastiCache (Redis), ECS (Fargate) or EC2/ECS
- **Realtime scale**: `@socket.io/redis-adapter` wired; stateless API instances
- **Ops**: health checks, structured logs to CloudWatch, basic alarms hooks

## Learning goals
- **Infrastructure thinking**: configs, secrets, environments, deployment safety.
- **Distributed systems**: pub/sub, multi-instance consistency, failure modes.
- **Operational maturity**: observability, health checks, rollback habits.

## Features

### 1) Production Dockerization
- Create Dockerfiles for `apps/api` and `apps/web` using multi-stage builds.
- Add a production-oriented Compose file for local “prod-like” testing.
- Ensure images run as non-root and contain no secrets.
- Add container health checks (`/health`) and graceful shutdown handling.
- Document local runbooks (build/run/debug) in `README.md`.

### 2) CI pipeline (quality gates)
- Add GitHub Actions workflow: install → lint → test → build.
- Cache dependencies to speed up CI runs.
- Build Docker images on main branch and tag with commit SHA.
- Fail fast on type errors and schema drift (Prisma migration checks).
- Store build artifacts/logs in CI for debugging failures.

### 3) AWS runtime (baseline)
- Provision RDS Postgres and ElastiCache Redis (document or IaC).
- Provision ECS cluster + service(s) for API; configure env + secrets.
- Configure web hosting strategy (static hosting + CDN or containerized web).
- Add HTTPS and secure cookie settings for production domains.
- Add basic operational dashboards/log access patterns (CloudWatch).

### 4) Multi-instance realtime (Redis adapter)
- Wire `@socket.io/redis-adapter` so rooms broadcast across instances.
- Ensure presence storage uses Redis as source of truth (sets + TTL where needed).
- Validate message fanout: user on instance A receives message from instance B.
- Add heartbeat/ping strategy to reduce ghost connections.
- Load test basic concurrency locally (documented script; small and repeatable).

## Exit criteria (demo script)
- Deploy web + api to AWS and login works.
- Run 2+ API instances → realtime messages still deliver across instances.
- CI runs on PRs and blocks merges on failing checks.

