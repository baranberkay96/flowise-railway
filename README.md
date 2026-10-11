# Deploy and Host Flowise with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/flowise-3-agentflows-queue-mode?referralCode=20H68q)

Flowise is an open-source, low-code builder for LLM chatbots, agents and multi-agent Agentflows. This community template runs Flowise 3 in queue mode: a main server for the UI and API, scalable worker containers that execute flows from a Redis queue, Postgres for data and a Railway Bucket for files.

## About Hosting Flowise

A single Flowise container handles the editor, the API and every flow execution in one Node.js process, so one heavy document upsert or long agent run slows everyone down. Flowise's queue mode splits this: the main server accepts requests and pushes executions to Redis (BullMQ), and dedicated workers pick them up in parallel. Running it properly needs Postgres instead of SQLite, shared object storage for uploads, and the same encryption and session secrets on every container. This template wires those pieces on Railway with generated secrets and private networking, so workers can be scaled with replicas.

## Common Use Cases

- Customer-support and internal-knowledge chatbots over your documents (RAG with document stores)
- Multi-agent Agentflows that call tools, APIs and other agents
- Embeddable chat widgets and prediction APIs for your own apps
- Scheduled or high-volume LLM pipelines that need parallel workers
- Rapid prototyping of LLM features by product teams without backend code

## Dependencies for Flowise Hosting

- Flowise main server (`flowiseai/flowise`)
- Flowise worker (`flowiseai/flowise-worker`)
- Redis (BullMQ queue, sessions, streaming events)
- PostgreSQL (flows, users, credentials, executions)
- Railway Bucket (S3-compatible storage for uploaded files)

### Deployment Dependencies

- Flowise repository: https://github.com/FlowiseAI/Flowise
- Docker and queue compose files: https://github.com/FlowiseAI/Flowise/tree/main/docker
- Running Flowise using queue: https://docs.flowiseai.com/configuration/running-flowise-using-queue
- Environment variables: https://docs.flowiseai.com/configuration/environment-variables
- Railway Buckets: https://docs.railway.com/guides/storage-buckets

### Implementation Details

| Service | Image | Role | Public |
|---|---|---|---|
| Flowise | `flowiseai/flowise:3.1.4` | UI, REST/prediction API, enqueues executions, schedule beat | **yes** (port 3000) |
| Flowise Worker | `flowiseai/flowise-worker:3.1.4` | Executes predictions, upserts and scheduled runs from the queue | no |
| Redis | Railway Redis 8.2 | BullMQ queues, session store, streaming events | no |
| Postgres | Railway Postgres 18 | Application database | no |
| Bucket | Railway Bucket | Uploaded and generated files | — |

**First login**

1. Open the public URL as soon as the deploy is healthy and create the owner account. Flowise has no environment-based admin bootstrap: the first person to register becomes the owner, so do this before sharing the URL.
2. Add credentials (model provider keys) under *Credentials*. They are encrypted with `FLOWISE_SECRETKEY_OVERWRITE`; never change that value after first boot.
3. Optional: set the SMTP variables on the Flowise service to send user invitations and password-reset emails.
4. The BullMQ dashboard is available at `/admin/queues` for logged-in users.

**Scaling workers:** increase replicas on **Flowise Worker** (it has no volume) or raise `WORKER_CONCURRENCY`. Keep the main **Flowise** service at one replica and scale it vertically. All shared settings on the worker are references to the main service, so they stay in sync.

**Custom domain:** add it to the Flowise service and update `APP_URL`.

**Pinning and upgrading:** both images are pinned to `3.1.4`. To upgrade, back up Postgres, set the same new tag on Flowise and Flowise Worker, and redeploy; the main server migrates the database on start.

**License note:** Flowise is Apache-2.0, except the `packages/server/src/enterprise` directory, which is under a commercial license (enterprise features need a license key and stay off by default).

### Why Deploy Flowise on Railway?

Railway runs the Flowise server, its workers, Redis, Postgres and an S3-compatible bucket in one project on a private network with managed HTTPS. Adding worker capacity is a replica change rather than new infrastructure, and upgrades are a tag change with one-click rollback. This is a community template and is not affiliated with FlowiseAI, Inc.
