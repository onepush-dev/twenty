# Twenty CRM — open-source Salesforce alternative (v2.48.0, server + worker + Postgres 16 + Redis). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[Twenty](https://twenty.com) is the open-source CRM from [twentyhq/twenty](https://github.com/twentyhq/twenty): companies, people, opportunities, tasks and notes. It has custom objects and fields you build in the UI, kanban/table/calendar views, workflows, email and calendar sync, roles and permissions, and GraphQL + REST APIs for every object. This template runs the **official** `twentycrm/twenty:v2.48.0` image, pinned by digest, on the upstream `packages/twenty-docker/docker-compose.yml` layout: **server**, **worker**, **Postgres 16** and **Redis**. It needs **no deploy-time input**. Open the URL and create your account and workspace.

## Why this template
- **Pinned, current release.** `v2.48.0` (released 2026-10-16) is the latest stable v2 release, pinned by tag **and** digest. Other Twenty templates on Railway float on `twentycrm/twenty:latest`, so a redeploy can silently jump versions and run migrations you didn't plan for.
- **Upstream compose, mapped 1:1.** The server runs the image's own entrypoint (DB init, `upgrade`, cron registration). The worker runs `yarn worker:prod` with `DISABLE_DB_MIGRATIONS=true` and `DISABLE_CRON_JOBS_REGISTRATION=true`, exactly as the compose file sets them. Postgres 16 matches the compose `postgres:16`, and Redis uses `--maxmemory-policy noeviction`, which BullMQ needs.
- **Files persist.** The server's local storage (`/app/packages/twenty-server/.local-storage`: avatars, attachments, workspace logos) is on a Railway volume. The image runs as uid 1000 and Railway volumes mount root-owned, so the server sets `RAILWAY_RUN_UID=0` and keeps the original entrypoint (no start-command override).
- **Generated secrets.** `APP_SECRET` (JWT/session signing) and `ENCRYPTION_KEY` (at-rest encryption of stored credentials) are generated per deploy, and the worker references the server's values. The Postgres and Redis passwords are generated too.
- **Private by default.** Only the `twenty` server gets a public domain. The worker, Postgres and Redis talk over Railway's private network.

## Services
| Service | Source | Port | Volume | Notes |
|---|---|---|---|---|
| twenty | `twentycrm/twenty:v2.48.0@sha256:43fab4bffacdcd63ff3c9257b7f8985a4424d81e9eb43982b9c16a841d555e52` | `3000` (public) | `/app/packages/twenty-server/.local-storage` | API + web app. Health `/healthz` (timeout 600 s). No start command (image entrypoint runs migrations, then `node dist/main`) |
| twenty-worker | same image and digest | none (private) | none | start command `/app/entrypoint.sh yarn worker:prod`. BullMQ jobs: emails, calendar, workflows, timeline, webhooks, crons |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:16.15@sha256:34c3e49e7ed2aaef6c575fb23bb37b878007e97752d3bf27ccb550046f893ca7` | `5432` (private) | `/var/lib/postgresql/data` | Railway Postgres image, major 16 as in upstream compose |
| Redis | `redis:8.2.10@sha256:47670742d7924adbcdb404d1288b6327815b23141969c0939b8e5d61f78c2634` | `6379` (private) | `/data` | Railway Redis start command + `--maxmemory-policy noeviction --bind '* -::*'` |

### Start commands
Railway replaces an image's ENTRYPOINT when a start command is set, so commands call the upstream entrypoint themselves (also in `railway/*-start-command.txt`).

**twenty-worker:**
```
/app/entrypoint.sh yarn worker:prod
```
**Redis:**
```
/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH --maxmemory-policy noeviction --bind '* -::*'"
```
**twenty** and **Postgres:** none (image defaults).

## Variables

### twenty (server)
| Variable | Template value | Notes |
|---|---|---|
| NODE_PORT | `3000` | port the server listens on |
| PORT | `3000` | port Railway routes the public domain to |
| SERVER_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | public URL; must be a valid URL or the server refuses to boot |
| PG_DATABASE_URL | `${{Postgres.DATABASE_URL}}` | |
| REDIS_URL | `${{Redis.REDIS_URL}}` | |
| APP_SECRET | `${{secret(32)}}` | signs JWTs/sessions; keep it stable |
| ENCRYPTION_KEY | `${{secret(32)}}` | encrypts stored secrets (OAuth tokens, app variables); never change it without `FALLBACK_ENCRYPTION_KEY` |
| STORAGE_TYPE | `local` | files on the server volume |
| DISABLE_DB_MIGRATIONS | `false` | server runs init/upgrade on boot |
| DISABLE_CRON_JOBS_REGISTRATION | `false` | server registers background crons |
| RAILWAY_RUN_UID | `0` | run as root so the root-owned volume is writable |

### twenty-worker
| Variable | Template value |
|---|---|
| SERVER_URL | `https://${{twenty.RAILWAY_PUBLIC_DOMAIN}}` |
| PG_DATABASE_URL | `${{Postgres.DATABASE_URL}}` |
| REDIS_URL | `${{Redis.REDIS_URL}}` |
| APP_SECRET | `${{twenty.APP_SECRET}}` |
| ENCRYPTION_KEY | `${{twenty.ENCRYPTION_KEY}}` |
| STORAGE_TYPE | `local` |
| DISABLE_DB_MIGRATIONS | `true` |
| DISABLE_CRON_JOBS_REGISTRATION | `true` |

### Postgres
| Variable | Template value |
|---|---|
| POSTGRES_USER | `postgres` |
| POSTGRES_PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")}}` |
| POSTGRES_DB | `railway` |
| PGDATA | `/var/lib/postgresql/data/pgdata` |
| PGHOST / PGPORT / PGUSER / PGPASSWORD / PGDATABASE | `${{RAILWAY_PRIVATE_DOMAIN}}` / `5432` / `${{POSTGRES_USER}}` / `${{POSTGRES_PASSWORD}}` / `${{POSTGRES_DB}}` |
| DATABASE_URL | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` |
| SSL_CERT_DAYS | `820` |
| RAILWAY_DEPLOYMENT_DRAINING_SECONDS | `60` |

### Redis
| Variable | Template value |
|---|---|
| REDIS_PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")}}` |
| REDISHOST / REDISPORT / REDISUSER / REDISPASSWORD | `${{RAILWAY_PRIVATE_DOMAIN}}` / `6379` / `default` / `${{REDIS_PASSWORD}}` |
| REDIS_URL | `redis://${{REDISUSER}}:${{REDIS_PASSWORD}}@${{REDISHOST}}:${{REDISPORT}}` |

Every value is also in `env.example` (and `railway/template-variables.env`).

## After deploy
1. Wait for `twenty` to go healthy. The first boot creates the schema and runs about 180 migrations plus upgrade commands, which takes 2–3 minutes after the image pull.
2. Open `https://<your-domain>` → **Continue with Email** → enter an email and password. On a fresh instance the first sign-up creates the workspace, and that user is the workspace admin. No email verification is needed.
3. Name your workspace. It starts with demo companies and people that you can delete.
4. Invite teammates from **Settings → Members** with the invite link (no SMTP needed). After the first workspace exists, `IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS` (default `true`) stops strangers from creating more workspaces. They can only join through an invite.

Optional, add on `twenty` **and** `twenty-worker`:
- Email: `EMAIL_DRIVER=smtp`, `EMAIL_SMTP_HOST`, `EMAIL_SMTP_PORT`, `EMAIL_SMTP_USER`, `EMAIL_SMTP_PASSWORD`, `EMAIL_FROM_ADDRESS`, `EMAIL_FROM_NAME`.
- Google/Microsoft sign-in and Gmail/Outlook sync: the `AUTH_GOOGLE_*` / `AUTH_MICROSOFT_*` and `MESSAGING_PROVIDER_*` / `CALENDAR_PROVIDER_*` variables from the upstream compose file.

## Notes
- **Storage and the worker.** On Railway a volume attaches to one service only, so the worker can't share the server's `.local-storage` the way it does in Docker Compose. Uploads, avatars and logos go through the server, so they work. Worker-side file jobs write to the worker's ephemeral disk instead. That covers file deletion, the optional async CSV export (behind a feature flag, off by default) and SDK-client archives, which the server regenerates on demand. If you rely on those, switch both services to S3 storage: `STORAGE_TYPE=s3`, `STORAGE_S3_REGION`, `STORAGE_S3_NAME`, `STORAGE_S3_ENDPOINT`, `STORAGE_S3_ACCESS_KEY_ID`, `STORAGE_S3_SECRET_ACCESS_KEY`, for example from a Railway Bucket.
- **Keep secrets stable.** Changing `APP_SECRET` logs everyone out. Changing `ENCRYPTION_KEY` makes stored credentials unreadable unless you set the old value as `FALLBACK_ENCRYPTION_KEY`.
- **First boot must succeed with a valid `SERVER_URL`.** The entrypoint only runs `database:init` when the `core` schema is missing. A first boot that dies halfway (for example with no public domain yet) leaves a half-initialized database that later boots won't repair. If that happens on an empty install, wipe the Postgres volume and redeploy.
- **Upgrades:** change the image tag and digest on **both** `twenty` and `twenty-worker`. The server runs `upgrade` on boot. Read the [upgrade guide](https://docs.twenty.com/developers/self-host/capabilities/upgrade-guide) and back up Postgres first.
- **RAM:** the server and worker each use about 0.5–1 GB at idle. Postgres and Redis are small.
- Upstream: https://github.com/twentyhq/twenty · compose at v2.48.0: https://github.com/twentyhq/twenty/tree/twenty/v2.48.0/packages/twenty-docker · Docs: https://docs.twenty.com
