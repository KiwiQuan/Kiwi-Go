# Phase 04 — Deployment + Scale (DevOps + distributed realtime)

## Goal
Make the app **deployable and horizontally scalable**: Dockerized services, a CI/CD pipeline that runs on every push, AWS infrastructure, and Redis-backed Socket.IO so multiple API instances stay in sync. This is the DevOps and distributed systems phase.

## Scope
- Local dev stays simple (Docker Compose). Production deploy becomes repeatable and scripted.
- The API can run as 2+ instances and still broadcast messages and presence to all users.
- CI blocks merges on failing lint/tests/build. CD pushes images and deploys on merge to main.

## Deliverables
- **Docker**: multi-stage builds for web + api; non-root user; no secrets in images
- **GitHub Actions**: lint → test → build → push image → deploy
- **AWS**: RDS (Postgres), ElastiCache (Redis), ECS Fargate (API), S3 + CloudFront (web)
- **Redis adapter**: `@socket.io/redis-adapter` makes rooms work across multiple API instances
- **Ops**: `/health` endpoint → ECS health check; structured JSON logs → CloudWatch

## New technologies introduced this phase
| Technology | What you'll use it for | Your current level |
|---|---|---|
| **Docker** | Containerize web + api for consistent deployments | New |
| **GitHub Actions** | Automate lint/test/build/deploy on every push | New |
| **AWS ECS Fargate** | Run containers without managing servers | New |
| **AWS RDS** | Managed Postgres in production | New |
| **AWS ElastiCache** | Managed Redis in production | New |
| **@socket.io/redis-adapter** | Sync Socket.IO rooms across multiple API instances | New concept |

## Learning approach
Mini-lab Docker (containerize a tiny Express app) and mini-lab GitHub Actions (one workflow, one job) before wiring up the real pipeline.

---

## Section A — Docker

### A1) Mini-lab: Docker basics (before containerizing the real app)

> **Why**: Docker is completely new. Spend 1 day containerizing a tiny Express server before tackling the multi-stage build for the real API. The concepts (layers, stages, non-root, env vars) will click much faster on a simple target.

- [ ] Install Docker Desktop if you haven't already. Run `docker --version` to confirm it works.
- [ ] Write a minimal `Dockerfile` for a tiny Express server (single file, no TypeScript). Build it: `docker build -t my-test-api .`. Run it: `docker run -p 3000:3000 my-test-api`.
- [ ] Verify the container's app is accessible at `localhost:3000` but you cannot SSH into it easily (it's isolated).
- [ ] Try passing an env var: `docker run -e PORT=4000 -p 4000:4000 my-test-api` — confirm the app picks it up.
- [ ] Write down in `_docs/decisions/docker-notes.md`: what a layer is, why you shouldn't run containers as root, why you never bake `.env` files into an image.

**Examples**

```dockerfile
# (example) simplest possible Dockerfile for a Node.js app
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# Build and run the mini-lab container
docker build -t my-test-api .
docker run -p 3000:3000 my-test-api

# List running containers
docker ps

# Stop it
docker stop <container-id>
```

**Done checks**
- [ ] Container runs and responds to HTTP.
- [ ] You can explain: what `FROM`, `WORKDIR`, `COPY`, `RUN`, `CMD` each do.

---

### A2) Multi-stage Dockerfile for the API (TypeScript)

- [ ] Add `infra/docker/api.Dockerfile` with three named stages: `builder` (installs all deps + compiles TS), `deps` (installs production deps only), `runner` (copies `dist/` + `node_modules/` and runs as non-root).
- [ ] In the `runner` stage, switch to the built-in `node` user: `USER node` — never run as root in production.
- [ ] Add a `.dockerignore` to exclude `node_modules/`, `.env`, `dist/`, and `*.log`.
- [ ] Build and run the production image locally: `docker build -f infra/docker/api.Dockerfile -t kiwigo-api .`.
- [ ] Confirm the image runs correctly with env vars passed via `--env-file .env.docker` (a production-safe env file with no secrets, just structure).

**Examples**

```dockerfile
# infra/docker/api.Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app
# Copy only what's needed for production — no dev deps, no source files
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/dist ./dist

# Run as non-root user — security best practice
USER node

EXPOSE 4000
CMD ["node", "dist/server.js"]
```

```text
# .dockerignore
node_modules
dist
.env
*.log
```

**Done checks**
- [ ] `docker build` completes with no errors.
- [ ] Running the production image → API starts, `/health` responds.
- [ ] `docker inspect <image>` → no `.env` file visible inside the image.

---

### A3) Docker Compose for local prod-like testing

- [ ] Add a `compose.prod.yml` that runs the API image alongside Postgres + Redis (mirrors production layout locally).
- [ ] Pass env vars via `environment:` keys (referencing host env vars with `${VAR_NAME}`) — no hardcoded secrets.
- [ ] Add container health checks for the API service (point to `/health`).
- [ ] Document the "prod-like local test" runbook in `README.md`: build images → `docker compose -f compose.prod.yml up`.
- [ ] Confirm end-to-end works: login → send message → message persists in the Postgres container.

**Done checks**
- [ ] `docker compose -f compose.prod.yml up` starts all three services; app works end-to-end.

---

## Section B — CI/CD (GitHub Actions)

### B1) Mini-lab: GitHub Actions basics (one workflow, one job)

> **Why**: GitHub Actions syntax (YAML indentation, `on:`, `jobs:`, `steps:`) is new. Spend half a day getting a trivial workflow passing before writing the real CI pipeline.

- [ ] Create `.github/workflows/hello.yml` that triggers on every push, has one job, and runs `echo "CI is working"`.
- [ ] Push to GitHub and find the workflow in the **Actions** tab — watch it run.
- [ ] Extend it: add a step that runs `npm install` and a step that runs `echo "tests passed"`.
- [ ] Intentionally break the workflow (a syntax error) — observe the red X and fix it.
- [ ] Write down in `_docs/decisions/ci-notes.md`: what a trigger is, what a job is, what a step is, what `secrets.GITHUB_TOKEN` is.

**Examples**

```yaml
# .github/workflows/hello.yml
name: Hello CI

on: [push]

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Say hello
        run: echo "CI is working on branch ${{ github.ref_name }}"
```

**Done checks**
- [ ] Workflow appears in the GitHub Actions tab and shows green on a clean push.

---

### B2) Real CI pipeline (lint + typecheck + test + build)

- [ ] Create `.github/workflows/ci.yml` that triggers on `push` and `pull_request` to `main`.
- [ ] Add steps: checkout → install deps (with npm cache) → lint → typecheck → test → build.
- [ ] Cache `~/.npm` using `actions/cache` so installs are fast after the first run.
- [ ] Add a `prisma migrate diff` check that fails CI if there are uncommitted schema changes.
- [ ] Confirm: pushing a branch with a failing test blocks the PR with a red status check.

**Examples**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
```

**Done checks**
- [ ] All steps pass on a clean commit.
- [ ] Breaking a TypeScript type → CI fails on the `typecheck` step with a clear error.

---

### B3) Docker image build + push to AWS ECR

- [ ] Create an ECR repository for the API in the AWS Console. Add `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_REGION` to GitHub Secrets.
- [ ] Add a `deploy.yml` workflow that triggers only on `push` to `main` (after CI passes): login to ECR → build image → tag with commit SHA → push.
- [ ] Use commit SHA as the image tag (never `latest` in production — it makes rollbacks ambiguous).
- [ ] Add a step that logs the pushed image URI so you can identify it in AWS.
- [ ] Confirm the image appears in ECR after a push to main.

**Examples**

```yaml
# (example) ECR login and push step
# - name: Login to Amazon ECR
#   uses: aws-actions/amazon-ecr-login@v2
#
# - name: Build and push image
#   env:
#     ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
#     IMAGE_TAG: ${{ github.sha }}
#   run: |
#     docker build -f infra/docker/api.Dockerfile -t $ECR_REGISTRY/kiwigo-api:$IMAGE_TAG .
#     docker push $ECR_REGISTRY/kiwigo-api:$IMAGE_TAG
#     echo "Pushed image: $ECR_REGISTRY/kiwigo-api:$IMAGE_TAG"
```

**Done checks**
- [ ] Push to `main` → image appears in ECR tagged with the commit SHA.
- [ ] Rolling back is just changing the task definition to point to a previous SHA tag.

---

## Section C — AWS Infrastructure

### C1) Mini-lab: AWS console orientation (before provisioning)

> **Why**: AWS has dozens of services and menus. Spend half a day clicking around before you provision anything, so you understand the relationships: VPC → subnets → security groups → RDS/ECS/ElastiCache.

- [ ] In the AWS console, navigate to EC2 → VPC → find the default VPC and its subnets. Write down what a VPC is in your own words.
- [ ] Navigate to RDS → Databases — read what instance class and Multi-AZ mean.
- [ ] Navigate to ECS → Clusters — read what a cluster, service, and task definition mean.
- [ ] Navigate to IAM → understand roles vs. users — your ECS task will need a role, not a user with a key.
- [ ] Write a 5-bullet summary in `_docs/decisions/aws-notes.md` — your mental model before you start spending money.

**Done checks**
- [ ] You can explain the difference between: RDS instance vs. ElastiCache cluster, ECS Service vs. ECS Task, IAM Role vs. IAM User.

---

### C2) Provision RDS (Postgres) + ElastiCache (Redis)

- [ ] Create an RDS Postgres instance (choose `db.t3.micro` for free tier). Enable automated backups. Store the connection string in AWS Secrets Manager.
- [ ] Create an ElastiCache Redis cluster (choose `cache.t3.micro`). Keep it in the same VPC as RDS.
- [ ] Configure security groups: RDS and ElastiCache should only accept connections from within the VPC (not the public internet).
- [ ] Add `DATABASE_URL` (pointing to RDS) and `REDIS_URL` (pointing to ElastiCache) to AWS Secrets Manager and reference them in your ECS task definition.
- [ ] Run `npx prisma migrate deploy` against the production database from a local machine once to initialize the schema.

**Done checks**
- [ ] Can connect to RDS from a local machine (with temporary security group rule) and run `prisma migrate status`.
- [ ] Cannot connect to RDS directly from the public internet without the security group rule.

---

### C3) ECS Fargate service for the API

- [ ] Create an ECS cluster and a task definition for the API: reference the ECR image, set CPU/memory (0.5 vCPU / 1GB), inject env secrets from Secrets Manager.
- [ ] Create an ECS Service (Fargate launch type): start with 1 desired task. Add a health check pointing to `/health`.
- [ ] Put the ECS service behind an Application Load Balancer — the ALB is the only public endpoint.
- [ ] Configure HTTPS on the ALB using an ACM certificate (AWS Certificate Manager — free for AWS-managed certs).
- [ ] Test: hit the public ALB URL → API responds; hit `/health` → `{ status: "ok" }`.

**Done checks**
- [ ] ALB URL reaches the API over HTTPS.
- [ ] ECS task shows as healthy in the ECS console.
- [ ] Env vars are loaded from Secrets Manager, not hardcoded anywhere.

---

## Section D — Multi-instance Realtime (Redis Adapter)

### D1) Mini-lab: Redis adapter for Socket.IO (simulate 2 instances locally)

> **Why**: The default Socket.IO setup only shares rooms within one server process. When you scale to 2 instances, users on instance A can't receive messages from users on instance B — unless you wire up the Redis adapter. Test this problem first so the fix makes sense.

- [ ] Start two separate Socket.IO servers on different ports (4000 and 4001). Connect two browser tabs — one to each server.
- [ ] Without the Redis adapter: send a message to a room from port 4000. Confirm the tab on port 4001 does NOT receive it. (This is the scaling problem.)
- [ ] Install `@socket.io/redis-adapter` and `ioredis`. Wire both servers to use the adapter with the same Redis instance.
- [ ] Re-test: send a message from port 4000 — confirm the tab on port 4001 now receives it. (Problem solved.)
- [ ] Write down in `_docs/decisions/redis-adapter-notes.md`: what pub/sub means, why you need two Redis connections (one pub, one sub), what would break without this.

**Examples**

```ts
// (example) wiring the Redis adapter — add to your Socket.IO server setup
import { Redis } from "ioredis";
import { createAdapter } from "@socket.io/redis-adapter";

const pubClient = new Redis(env.REDIS_URL);
const subClient = pubClient.duplicate(); // MUST be a separate connection for sub mode

io.adapter(createAdapter(pubClient, subClient));
// Now io.to("room_xyz").emit(...) works across ALL instances of your API
```

**Done checks**
- [ ] You proved the problem (message not delivered across instances) and the fix (delivered with adapter).
- [ ] You can explain: why pub/sub needs two clients, what "adapter" means in Socket.IO.

---

### D2) Redis adapter integration + presence fix

- [ ] Wire `@socket.io/redis-adapter` into the real API's Socket.IO server under `apps/api/src/realtime/`.
- [ ] Confirm presence tracking (`SADD`/`SREM`) still works correctly: it already uses Redis Sets directly — no change needed here, the adapter only affects rooms.
- [ ] Scale to 2 ECS tasks (update desired count to 2) and verify end-to-end: send from a client on instance A, received by client on instance B.
- [ ] Add a heartbeat/ping check (Socket.IO's built-in `pingInterval`/`pingTimeout`) to detect and clean up ghost connections.
- [ ] Document the multi-instance architecture in `_docs/decisions/scaling-notes.md` with a simple diagram.

**Done checks**
- [ ] 2 ECS tasks running → send a message → both users receive it regardless of which instance they're on.
- [ ] `SMEMBERS online_users` in Redis matches actual connected users.

---

## Exit criteria (demo script)

- [ ] Push to `main` → CI passes → Docker image is built and pushed to ECR.
- [ ] ECS deploys the new image automatically.
- [ ] Visit the production URL (HTTPS) → login works → send a message.
- [ ] Scale ECS to 2 tasks → realtime messaging still works across both instances.
- [ ] Roll back: update ECS task definition to previous commit SHA image → app reverts cleanly.
