# Aine Forge Starter

Monorepo with a Next.js frontend and a long-running orchestrator service, communicating through PostgreSQL job tables with LISTEN/NOTIFY.

## Standup app

The frontend is an async team standup: sign in with a GitHub personal access token, post
yesterday / today / blockers, read the team feed, work the blocker board, and generate a
weekly AI summary. Updates live in PostgreSQL; the summary and the "draft from my GitHub
activity" button are queued as jobs and answered by the orchestrator via Bedrock.

Set `SESSION_SECRET` (see `.env.example`) — it encrypts the session cookie that holds the PAT.

## Structure

```
apps/
  forge-fe/              ← Next.js 16 frontend (port 3000 / Lambda)
  forge-orchestrator/    ← Fastify job processor (port 3001 / EC2 Docker)
packages/
  db/                    ← Prisma schema, migrations, shared pool + types
.github/workflows/
  deploy.yml             ← Build + deploy on every push (branch deploys)
  cleanup.yml            ← Destroy branch envs on delete + nightly sweep
```

## Tech Stack

- **Next.js 16** — App Router, TypeScript, Tailwind CSS 4
- **Fastify** — Orchestrator HTTP + health check
- **PostgreSQL** — Job queue via LISTEN/NOTIFY
- **Prisma** — Schema management + migrations
- **Jest** — Unit testing
- **npm workspaces** — Monorepo management

## Deployment Architecture

- **FE** → Lambda (container image via `aws-lambda-web-adapter`)
- **Orchestrator** → Docker containers on a shared EC2 instance
- **Database** → Single RDS PostgreSQL, per-branch `CREATE DATABASE`
- **Service registry** → DynamoDB maps branch → port, status, DB name
- **CI/CD** → GitHub Actions with OIDC auth (no stored AWS keys)
- **Branch deploys** → Each branch gets its own Lambda, orchestrator container, and database
- **Auto-sleep** → Idle orchestrators stopped after 5 min, woken on demand

Infrastructure is provisioned via Terraform in [aine-forge-infra](https://github.com/wwtdigital/aine-forge-infra).

## Getting Started

1. **Install dependencies**

   ```bash
   npm install
   ```

2. **Set up environment variables**

   ```bash
   cp .env.example .env
   ```

   Edit `.env` with your PostgreSQL connection string.

3. **Run database migrations**

   ```bash
   npm run db:migrate
   ```

4. **Start both services**

   ```bash
   npm run dev:fe    # Next.js on :3000
   npm run dev:orch  # Orchestrator on :3001
   ```

5. **Run tests**

   ```bash
   npm test
   ```

## Deploying to AWS

1. **Provision infrastructure** — see [aine-forge-infra](https://github.com/wwtdigital/aine-forge-infra)

2. **Configure GitHub repo** — the infra setup script (`setup-github.sh`) pushes these automatically:
   - **Variables:** `AWS_ROLE_ARN`, `EC2_PUBLIC_IP`, `DYNAMODB_TABLE`, `LAMBDA_ROLE_ARN`, `LAMBDA_SG_ID`, `PRIVATE_SUBNET_IDS`, `RDS_ENDPOINT`, `EC2_SSH_KEY_ARN`
   - The EC2 SSH key is stored in AWS Secrets Manager and fetched at runtime via OIDC — no repo secret needed

3. **Push any branch** — GitHub Actions will automatically build, deploy, and output URLs

## Custom Environment Variables

Custom env vars for deployed services are stored as **GitHub Actions variables** (non-secret) and **GitHub Actions secrets** (sensitive). Nothing is committed to the repo.

### Quick start

1. Create local `.env`-format files in a `deploy/` directory (gitignored):

   ```bash
   mkdir -p deploy

   # Non-secret orchestrator config
   cat > deploy/orchestrator.env << 'EOF'
   BEDROCK_MODEL_ID=us.anthropic.claude-haiku-4-5-20251001-v1:0
   EOF

   # Secret orchestrator config
   cat > deploy/orchestrator.secrets << 'EOF'
   EXTERNAL_API_KEY=sk-...
   EOF

   # Non-secret FE config (NEXT_PUBLIC_* also available at build time)
   cat > deploy/fe.env << 'EOF'
   NEXT_PUBLIC_APP_NAME=My App
   EOF
   ```

2. Push to GitHub:

   ```bash
   ./scripts/push-deploy-env.sh
   ```

   Or set directly via the GitHub CLI:

   ```bash
   gh variable set DEPLOY_ORCH_ENV --body '{"BEDROCK_MODEL_ID":"us.anthropic.claude-haiku-4-5-20251001-v1:0"}'
   gh secret set DEPLOY_ORCH_SECRETS --body '{"EXTERNAL_API_KEY":"sk-..."}'
   ```

### Naming convention

| GitHub variable / secret | Injected into | Type |
|---|---|---|
| `DEPLOY_SHARED_ENV` | Both FE + orchestrator | Variable |
| `DEPLOY_FE_ENV` | Lambda function env | Variable |
| `DEPLOY_ORCH_ENV` | Docker container `-e` flags | Variable |
| `DEPLOY_SHARED_SECRETS` | Both FE + orchestrator | Secret |
| `DEPLOY_FE_SECRETS` | Lambda function env | Secret |
| `DEPLOY_ORCH_SECRETS` | Docker container `-e` flags | Secret |

Values are JSON objects (`{"KEY":"value"}`). Shared vars are merged into both targets; secrets override variables on key conflict. `NEXT_PUBLIC_*` keys in FE are also written to `.env.production` at build time.