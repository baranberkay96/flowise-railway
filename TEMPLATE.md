# Flowise 3 Agentflows (Queue Mode) — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Flowise 3 Agentflows (Queue Mode)** (version label `3` = Flowise major version; the project tags `flowise@3.x.y` and markets "Flowise 3.0") |
| Description | Visual LLM agent builder with Redis queue workers for heavy, parallel runs. (75 chars) |
| Category | AI/ML |
| Tags | flowise, agents, llm, low-code, queue, bullmq, ai |
| User input at deploy | **None.** Optional later: SMTP variables for invitations / password reset. |

## Base market template

- **`A7Dwg9` "FlowiseAI"** (806 projects, 503 active) — chosen as structural base: Postgres-backed, `/api/v1/ping` healthcheck, pinned 3.1.4 (via its Dockerfile), explicit storage/log paths. Its repo (`Somi-AI/flowiseai-railway`) has **no license**: read only; its note that BullMQ needs `?family=0` on Redis URLs for IPv6 private networking was verified against ioredis behaviour and applied. Replaced: custom image build → official images, PostGIS with public TCP proxy → Railway Postgres, local volume → Bucket, single instance → queue mode.
- `pn4G8S` "flowise-railway" (8,547 projects, 5,611 active — the listing leader) — builds the unpinned Flowise `main` branch from source, SQLite inside the container with no volume (data lost on redeploy), no Redis/worker. Nothing reused.
- `Xx4_lu` "flowise-with-n8n-ai-automation" — `flowiseai/flowise` unpinned (`latest`) plus an n8n service; not relevant to queue mode. Nothing reused.

## Pinned versions

| Service | Image / source | Tag | Release date | Checked |
|---|---|---|---|---|
| Flowise (main) | `flowiseai/flowise` | `3.1.4` | 2026-07-29 | 2026-10-10 |
| Flowise Worker | `flowiseai/flowise-worker` | `3.1.4` | 2026-07-29 | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` | `18` | Railway base | 2026-10-10 |
| Redis | `redis` | `8.2` | Railway base | 2026-10-10 |

- `flowise@3.1.4` is the latest stable GitHub release (2026-07-29); both images carry the same tag.
- Upgrade path: back up Postgres, change **both** tags to the same new version, redeploy (main runs TypeORM migrations on start).

## Architecture

```
Internet ──HTTPS──▶ Flowise (main) :3000  ── UI, REST API, prediction endpoints, schedule beat
                      │  enqueue (BullMQ)            ▲ results / SSE events (Redis streams + pub/sub)
                      ▼                              │
                    Redis ◀──────────────── Flowise Worker ×N :5566 (/healthz)
Flowise + Worker ─▶ Postgres (flows, users, credentials, executions)
Flowise + Worker ─▶ Bucket   (uploaded files, document-store files, generated images)
```

Upstream reference: `docker/docker-compose-queue-prebuilt.yml` (flowise + flowise-worker + redis). Upstream shares `~/.flowise` between main and worker on one host; Railway volumes cannot be shared, so everything that must be common lives outside the containers:

| Shared state | Upstream compose | This template |
|---|---|---|
| Database | SQLite in `~/.flowise` (shared mount) | Railway Postgres |
| Uploaded files | `~/.flowise/storage` (shared mount) | Railway Bucket (`STORAGE_TYPE=s3`, `S3_ENDPOINT_URL`) |
| Credential encryption key | file in `~/.flowise` | `FLOWISE_SECRETKEY_OVERWRITE` (same value on both) |
| JWT / session / token-hash secrets | files in `~/.flowise` | env vars (same values on both, referenced from the main service) |
| Sessions | — | Redis (queue mode uses a Redis session store) |

Result: neither service needs a volume, so the worker scales horizontally with Railway replicas.

## Services

Create in this order. Values are entered exactly as shown (Railway resolves `${{...}}`).

### Flowise

- **Source:** Docker image `flowiseai/flowise:3.1.4`
- **Start command:** image default
- **Healthcheck:** `/api/v1/ping` (timeout 300 s)
- **Public domain:** yes → target port `3000`
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | HTTP port (public domain target). |
| `MODE` | `queue` | Queue mode: the main server enqueues executions in Redis (BullMQ), workers run them. |
| `QUEUE_NAME` | `flowise-queue` | BullMQ queue name (must match on main and workers). |
| `REDIS_URL` | `redis://${{Redis.REDISUSER}}:${{Redis.REDIS_PASSWORD}}@${{Redis.REDISHOST}}:${{Redis.REDISPORT}}?family=0` | Redis for queue, sessions and streaming. family=0 lets ioredis use IPv6 private networking. |
| `QUEUE_REDIS_EVENT_STREAM_MAX_LEN` | `10000` | Max events kept per Redis stream. |
| `REMOVE_ON_AGE` | `86400` | Remove finished jobs after N seconds. |
| `REMOVE_ON_COUNT` | `10000` | Keep at most N finished jobs. |
| `DATABASE_TYPE` | `postgres` | Database engine. |
| `DATABASE_HOST` | `${{Postgres.PGHOST}}` | Postgres private host. |
| `DATABASE_PORT` | `${{Postgres.PGPORT}}` | Postgres port. |
| `DATABASE_NAME` | `${{Postgres.PGDATABASE}}` | Postgres database. |
| `DATABASE_USER` | `${{Postgres.PGUSER}}` | Postgres user. |
| `DATABASE_PASSWORD` | `${{Postgres.PGPASSWORD}}` | Postgres password. |
| `DATABASE_SSL` | `false` | Private network; TLS not required. |
| `FLOWISE_SECRETKEY_OVERWRITE` | `${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Encryption key for stored credentials, shared by main and workers. Never change after first boot. |
| `JWT_AUTH_TOKEN_SECRET` | `${{secret(64, "abcdef0123456789")}}` | JWT signing secret for access tokens. |
| `JWT_REFRESH_TOKEN_SECRET` | `${{secret(64, "abcdef0123456789")}}` | JWT signing secret for refresh tokens. |
| `JWT_ISSUER` | `Flowise` | JWT issuer. |
| `JWT_AUDIENCE` | `Flowise` | JWT audience. |
| `JWT_TOKEN_EXPIRY_IN_MINUTES` | `360` | Access token lifetime. |
| `JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES` | `43200` | Refresh token lifetime. |
| `EXPRESS_SESSION_SECRET` | `${{secret(64, "abcdef0123456789")}}` | Session cookie signing secret. |
| `TOKEN_HASH_SECRET` | `${{secret(64, "abcdef0123456789")}}` | Secret used to hash reset/invite tokens. |
| `STORAGE_TYPE` | `s3` | Uploaded files go to the Railway Bucket (shared by main and workers). |
| `S3_STORAGE_BUCKET_NAME` | `${{Bucket.BUCKET}}` | Railway Bucket name. |
| `S3_STORAGE_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Railway Bucket access key. |
| `S3_STORAGE_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Railway Bucket secret key. |
| `S3_STORAGE_REGION` | `${{Bucket.REGION}}` | Railway Bucket region. |
| `S3_ENDPOINT_URL` | `${{Bucket.ENDPOINT}}` | Railway Bucket S3 endpoint. |
| `S3_FORCE_PATH_STYLE` | `false` | Railway Buckets use virtual-hosted style; set true only if the bucket's Credentials tab says path-style. |
| `APP_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public URL (links in emails, secure cookies). |
| `LOG_LEVEL` | `info` | error \| warn \| info \| verbose \| debug. |
| `DISABLE_FLOWISE_TELEMETRY` | `true` | Disable anonymous telemetry. |
| `FLOWISE_FILE_SIZE_LIMIT` | `50mb` | Max upload size. |
| `ENABLE_BULLMQ_DASHBOARD` | `true` | Queue dashboard at /admin/queues (requires a logged-in user). |
| `SMTP_HOST` | *(empty)* | Optional: SMTP server for invites and password resets. |
| `SMTP_PORT` | *(empty)* | Optional: SMTP port (e.g. 465 or 587). |
| `SMTP_USER` | *(empty)* | Optional: SMTP username. |
| `SMTP_PASSWORD` | *(empty)* | Optional: SMTP password. |
| `SMTP_SECURE` | `true` | Use TLS for SMTP. |
| `SENDER_EMAIL` | *(empty)* | Optional: From address for emails. |

### Flowise Worker

- **Source:** Docker image `flowiseai/flowise-worker:3.1.4`
- **Start command:** image default
- **Healthcheck:** `/healthz` (timeout 300 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `5566` | Port of the worker's health endpoint (Railway healthcheck). |
| `WORKER_PORT` | `5566` | Health endpoint port inside the worker image. |
| `WORKER_CONCURRENCY` | `25` | Executions processed in parallel per worker replica. |
| `MODE` | `${{Flowise.MODE}}` | Queue mode: the main server enqueues executions in Redis (BullMQ), workers run them. |
| `QUEUE_NAME` | `${{Flowise.QUEUE_NAME}}` | BullMQ queue name (must match on main and workers). |
| `REDIS_URL` | `${{Flowise.REDIS_URL}}` | Redis for queue, sessions and streaming. family=0 lets ioredis use IPv6 private networking. |
| `QUEUE_REDIS_EVENT_STREAM_MAX_LEN` | `${{Flowise.QUEUE_REDIS_EVENT_STREAM_MAX_LEN}}` | Max events kept per Redis stream. |
| `REMOVE_ON_AGE` | `${{Flowise.REMOVE_ON_AGE}}` | Remove finished jobs after N seconds. |
| `REMOVE_ON_COUNT` | `${{Flowise.REMOVE_ON_COUNT}}` | Keep at most N finished jobs. |
| `DATABASE_TYPE` | `${{Flowise.DATABASE_TYPE}}` | Database engine. |
| `DATABASE_HOST` | `${{Flowise.DATABASE_HOST}}` | Postgres private host. |
| `DATABASE_PORT` | `${{Flowise.DATABASE_PORT}}` | Postgres port. |
| `DATABASE_NAME` | `${{Flowise.DATABASE_NAME}}` | Postgres database. |
| `DATABASE_USER` | `${{Flowise.DATABASE_USER}}` | Postgres user. |
| `DATABASE_PASSWORD` | `${{Flowise.DATABASE_PASSWORD}}` | Postgres password. |
| `DATABASE_SSL` | `${{Flowise.DATABASE_SSL}}` | Private network; TLS not required. |
| `FLOWISE_SECRETKEY_OVERWRITE` | `${{Flowise.FLOWISE_SECRETKEY_OVERWRITE}}` | Encryption key for stored credentials, shared by main and workers. Never change after first boot. |
| `JWT_AUTH_TOKEN_SECRET` | `${{Flowise.JWT_AUTH_TOKEN_SECRET}}` | JWT signing secret for access tokens. |
| `JWT_REFRESH_TOKEN_SECRET` | `${{Flowise.JWT_REFRESH_TOKEN_SECRET}}` | JWT signing secret for refresh tokens. |
| `JWT_ISSUER` | `${{Flowise.JWT_ISSUER}}` | JWT issuer. |
| `JWT_AUDIENCE` | `${{Flowise.JWT_AUDIENCE}}` | JWT audience. |
| `JWT_TOKEN_EXPIRY_IN_MINUTES` | `${{Flowise.JWT_TOKEN_EXPIRY_IN_MINUTES}}` | Access token lifetime. |
| `JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES` | `${{Flowise.JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES}}` | Refresh token lifetime. |
| `EXPRESS_SESSION_SECRET` | `${{Flowise.EXPRESS_SESSION_SECRET}}` | Session cookie signing secret. |
| `TOKEN_HASH_SECRET` | `${{Flowise.TOKEN_HASH_SECRET}}` | Secret used to hash reset/invite tokens. |
| `STORAGE_TYPE` | `${{Flowise.STORAGE_TYPE}}` | Uploaded files go to the Railway Bucket (shared by main and workers). |
| `S3_STORAGE_BUCKET_NAME` | `${{Flowise.S3_STORAGE_BUCKET_NAME}}` | Railway Bucket name. |
| `S3_STORAGE_ACCESS_KEY_ID` | `${{Flowise.S3_STORAGE_ACCESS_KEY_ID}}` | Railway Bucket access key. |
| `S3_STORAGE_SECRET_ACCESS_KEY` | `${{Flowise.S3_STORAGE_SECRET_ACCESS_KEY}}` | Railway Bucket secret key. |
| `S3_STORAGE_REGION` | `${{Flowise.S3_STORAGE_REGION}}` | Railway Bucket region. |
| `S3_ENDPOINT_URL` | `${{Flowise.S3_ENDPOINT_URL}}` | Railway Bucket S3 endpoint. |
| `S3_FORCE_PATH_STYLE` | `${{Flowise.S3_FORCE_PATH_STYLE}}` | Railway Buckets use virtual-hosted style; set true only if the bucket's Credentials tab says path-style. |
| `APP_URL` | `${{Flowise.APP_URL}}` | Public URL (links in emails, secure cookies). |
| `LOG_LEVEL` | `${{Flowise.LOG_LEVEL}}` | error \| warn \| info \| verbose \| debug. |
| `DISABLE_FLOWISE_TELEMETRY` | `${{Flowise.DISABLE_FLOWISE_TELEMETRY}}` | Disable anonymous telemetry. |
| `FLOWISE_FILE_SIZE_LIMIT` | `${{Flowise.FLOWISE_FILE_SIZE_LIMIT}}` | Max upload size. |

### Postgres

- **Source:** Docker image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Copied unchanged from templates/_railway-base/postgres.json

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down |

### Redis

- **Source:** Docker image `redis:8.2`
- **Start command:** `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Copied unchanged from templates/_railway-base/redis.json

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment |
| `REDISPORT` | `6379` | Port that Redis listens on |
| `REDISUSER` | `default` | Username for authenticating with Redis |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis |

### Bucket (Railway Bucket)

- S3-compatible, private. Provides `BUCKET`, `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`, `REGION`, `ENDPOINT` (referenced as `${{Bucket.ENDPOINT}}` etc.).

## Improvements over market templates

1. **Real queue mode**: main + dedicated `flowise-worker` image + Redis (BullMQ), as in upstream's queue compose. No market template runs workers.
2. **Pinned official images** (`3.1.4` for both) instead of building `main` from source (`pn4G8S`) or `latest` (`Xx4_lu`).
3. **No data loss on redeploy**: the listing leader keeps SQLite inside the container without a volume. Here Postgres holds data, the Bucket holds files, and secrets live in variables.
4. **Shared secrets done right**: `FLOWISE_SECRETKEY_OVERWRITE`, `JWT_*`, `EXPRESS_SESSION_SECRET`, `TOKEN_HASH_SECRET` generated once on the main service and referenced by the worker (otherwise each container writes its own key file and credentials fail to decrypt on the worker).
5. **Object storage without MinIO**: Railway Bucket via Flowise's native S3 driver (`S3_ENDPOINT_URL`).
6. **IPv6-safe Redis URL** (`?family=0`) for ioredis/BullMQ on IPv6-only private networks.
7. **Postgres not publicly exposed** (`A7Dwg9` adds a TCP proxy to PostGIS); PostGIS replaced by the Railway Postgres base (Flowise does not use PostGIS).
8. Health checks on both services (`/api/v1/ping`, worker `/healthz`), BullMQ dashboard at `/admin/queues` behind Flowise login, telemetry off, `APP_URL` set so cookies are `Secure`.

## Scaling

- **Flowise Worker**: stateless — raise Railway replicas for more parallel executions, or `WORKER_CONCURRENCY` per replica (IO-bound LLM calls tolerate high values; document loaders with Chromium need more RAM).
- **Flowise (main)**: keep 1 replica by default (it also runs the schedule beat); scale vertically first.
- Postgres connections: Flowise uses TypeORM on node-postgres, whose default pool is 10 connections per process, so a few replicas stay well under Postgres' 100-connection limit.

## Resource estimate

Prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, Bucket $0.015/GB-month.

| Service | Idle RAM | Typical RAM | Typical vCPU | Volume |
|---|---|---|---|---|
| Flowise (main) | 350 MB | 600 MB | 0.02 idle / 0.15 | — |
| Flowise Worker (1 replica) | 300 MB | 800 MB | 0.01 idle / 0.25 | — |
| Postgres | 80 MB | 200 MB | 0.02 | ~1 GB |
| Redis | 15 MB | 60 MB | 0.01 | ~0.2 GB |
| Bucket | — | — | — | ~2 GB of files |
| **Total** | **≈ 0.75 GB** | **≈ 1.65 GB** | **0.06 / 0.43** | **≈ 1.2 GB + bucket** |

- **Idle:** 0.75 × $10 + 0.06 × $20 + 1.2 × $0.15 + 2 × $0.015 ≈ **$9/month**
- **Typical:** 1.65 × $10 + 0.43 × $20 + $0.2 ≈ **$25/month**
- Heavy parallel runs (3 worker replicas): ≈ $45–60/month.
- **Minimum plan: Hobby.** Fits Trial limits per service, but four services plus a bucket with continuous use exceed the Trial credit; Pro for many worker replicas.

## Verification checklist

1. `GET https://<domain>/api/v1/ping` → `pong`; worker logs show `Healthcheck server listening on port 5566` and the worker connecting to the queue.
2. **Immediately** open `https://<domain>/` and create the owner account (Flowise has no environment-based admin bootstrap; the first visitor registers the only organization).
3. Add a credential (e.g. OpenAI key), build a simple Chatflow/Agentflow and chat with it → the main logs show the job enqueued and the **worker** logs show it executed; streaming tokens arrive in the UI.
4. `https://<domain>/admin/queues` (logged in) shows completed jobs in `flowise-queue-prediction`.
5. Upload a PDF to a Document Store and upsert it → the object appears in the Railway Bucket; preview works.
6. Scale the worker to 2 replicas, run several chats in parallel → jobs spread across replicas.
7. Redeploy both services → login session survives (Redis session store), credentials still decrypt on the worker, files still downloadable.

## Open questions

- **Not deployed yet**: verified against upstream compose/source for `flowise@3.1.4`.
- **First-visitor owner registration**: Flowise 3 open source creates the owner on first registration with no env guard. Deploy, then claim the account right away (or remove the public domain until you have).
- **Railway Bucket compatibility**: Flowise uses AWS SDK v3 with `S3_ENDPOINT_URL`, region `auto` and virtual-hosted addressing; expected to work with Railway's S3 API, confirm upload/download on first deploy. Older buckets may need `S3_FORCE_PATH_STYLE=true`.
- **`?family=0`** relies on ioredis URL-option parsing (documented by ioredis/Railway); node-redis clients in Flowise ignore the query string and resolve IPv6 natively.
- **Custom MCP stdio servers / Chromium-based loaders** run inside the worker; they increase RAM per execution — size `WORKER_CONCURRENCY` accordingly.
- **Flowise license**: Apache-2.0 except `packages/server/src/enterprise` (commercial license; enterprise features stay disabled without `FLOWISE_EE_LICENSE_KEY`).
