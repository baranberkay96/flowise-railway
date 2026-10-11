# Notice

## Upstream software

| Software | License | Images used (publisher) |
|---|---|---|
| Flowise | Apache-2.0, except `packages/server/src/enterprise` and files with explicit notices (FlowiseAI commercial license). Copyright (c) 2023-present FlowiseAI, Inc. | `flowiseai/flowise:3.1.4`, `flowiseai/flowise-worker:3.1.4` (FlowiseAI, Docker Hub) |
| PostgreSQL | PostgreSQL License | `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway) |
| Redis | RSALv2 / SSPLv1 / AGPLv3 (Redis 8 tri-license) | `redis:8.2` (Docker Official Image) |

## Verified against

| What | Upstream file | Ref | URL | Checked |
|---|---|---|---|---|
| Queue topology, worker image + command, env per service | `docker/docker-compose-queue-prebuilt.yml` | `flowise@3.1.4` | https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/docker/docker-compose-queue-prebuilt.yml | 2026-10-10 |
| Variable names/defaults (DB, secrets, S3, queue, security) | `docker/.env.example`, `packages/server/.env.example` | `flowise@3.1.4` | https://github.com/FlowiseAI/Flowise/tree/flowise%403.1.4/docker | 2026-10-10 |
| Worker image, healthcheck server on `WORKER_PORT` | `docker/worker/Dockerfile`, `docker/worker/healthcheck/healthcheck.js`, `docker/worker/README.md` | `flowise@3.1.4` | https://github.com/FlowiseAI/Flowise/tree/flowise%403.1.4/docker/worker | 2026-10-10 |
| Main image (user `node`, `pnpm start`, port 3000) | `Dockerfile` | `flowise@3.1.4` | https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/Dockerfile | 2026-10-10 |
| BullMQ / ioredis / node-redis connection handling, queue names | `packages/server/src/queue/QueueManager.ts`, `packages/server/src/utils/redis.ts`, `packages/server/src/enterprise/middleware/passport/SessionPersistance.ts` | `flowise@3.1.4` | https://github.com/FlowiseAI/Flowise/tree/flowise%403.1.4/packages/server/src | 2026-10-10 |
| Auth secrets read from env | `packages/server/src/enterprise/utils/authSecrets.ts` | `flowise@3.1.4` | same | 2026-10-10 |
| Secure cookies from `APP_URL`, Redis session store in queue mode | `packages/server/src/enterprise/middleware/passport/index.ts` | `flowise@3.1.4` | same | 2026-10-10 |
| Trust-proxy default, queue mode not license-gated | `packages/server/src/index.ts` | `flowise@3.1.4` | same | 2026-10-10 |
| Open-source owner registration (no env bootstrap) | `packages/server/src/enterprise/services/account.service.ts` | `flowise@3.1.4` | same | 2026-10-10 |
| Queue mode docs | Running Flowise using Queue | — | https://docs.flowiseai.com/configuration/running-flowise-using-queue | 2026-10-10 |
| Railway Buckets | Storage Buckets guide | — | https://docs.railway.com/guides/storage-buckets | 2026-10-10 |

## Adapted code

None. The template uses the published images unmodified and configures them through documented environment variables.

## Reference only (not copied)

- `_upstream/Somi-AI__flowiseai-railway` — no license (`gh api repos/Somi-AI/flowiseai-railway` reports none): read for ideas only (Railway variable layout, the `?family=0` Redis URL hint, which was independently checked against ioredis behaviour). No files or text copied.
- `_upstream/derekcheungsa__n8n-railway-custom` — no license; unrelated to this template (belongs to the n8n service in market template `Xx4_lu`). Not used.
- Marketplace templates `pn4G8S` ("flowise-railway"), `A7Dwg9` ("FlowiseAI"), `Xx4_lu` ("flowise-with-n8n-ai-automation"): no license; reviewed for structure only. No files or text copied.

## Trademarks

"Flowise" and its logo belong to FlowiseAI, Inc. "Railway" belongs to Railway Corporation. This is a community template and is not affiliated with or endorsed by them.
