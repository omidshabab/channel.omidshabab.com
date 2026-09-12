# DS-CS Channel — omidshabab.com/channel

A self-hosted content-sharing channel (posts, shorts, stories) for `channel.omidshabab.com`, run from the official Docker image `dastyaresocial/ds-cs:latest`.

This repo is the deployment bundle: the installer script, the Docker Compose stack, the Vercel image wrapper, and the channel identity/config files.

## Method 1 — One-command install (recommended)

Requires Docker and curl. Run this exactly as-is:

```sh
curl -fsSL https://raw.githubusercontent.com/omidshabab/omidshabab-channel/main/scripts/install.sh | bash
```

What it does:

- Checks that Docker and curl are installed.
- Downloads `docker-compose.yml` from this repo (keeps your existing file if present).
- Creates `.env` if it doesn't exist, pre-seeded for the bundled stack (local Postgres + S3 emulation, admin `hey@omidshabab.com`).
- Generates a random `API_KEY` and `BETTER_AUTH_SECRET`.
- Starts the stack: `docker compose -f docker-compose.yml up -d`.

### After install

1. Open `http://localhost:8729` and log in with the seeded admin credentials `hey@omidshabab.com` / `change-this-password`.
2. Edit `.env` to replace `ADMIN_PASSWORD` (and `ADMIN_EMAIL` if you want a different address) with real values, then restart:

   ```sh
   docker compose -f docker-compose.yml up -d
   ```

## Method 2 — Manual fallback

Use this if the script can't reach raw.githubusercontent.com, or you prefer to wire things up yourself.

```sh
curl -fsSL -o docker-compose.yml https://raw.githubusercontent.com/omidshabab/omidshabab-channel/main/docker-compose.yml
cp .env.example .env
```

- Point `DATABASE_URL` at the Compose database: `postgresql://postgres:postgres@db:5432/dastyare_social_cs`.
- Leave the `S3_*` values that target `http://rustfs:9000` as-is for the bundled emulation, or change them to your external S3/MinIO/R2 bucket.
- Start and open the app:

  ```sh
  docker compose -f docker-compose.yml up -d
  # open http://localhost:8729
  ```

- To serve it on a domain, reverse-proxy `localhost:8729` with Nginx or Caddy and update `BETTER_AUTH_URL` / `NEXT_PUBLIC_APP_URL` to the public URL.

## Method 3 — Vercel-only (channel.omidshabab.com)

The repo's `Dockerfile.vercel` makes Vercel pull the prebuilt `dastyaresocial/ds-cs:latest` image and serve it with `next start` — nothing is compiled on Vercel. Because Vercel gives no persistent filesystem or Postgres, the bundled `db` and `rustfs` containers are not used:

1. Provision an external Postgres (e.g. Neon) and an S3 bucket (e.g. Cloudflare R2, Supabase storage, MinIO). Set `S3_PUBLIC_BASE_URL` to the bucket's public base URL.
2. Set the runtime env vars from the table below (minus the Docker-only ones: `S3_ENDPOINT`, `S3_FORCE_PATH_STYLE`) on the Vercel project, including `DATABASE_URL`, `BETTER_AUTH_URL`, `BETTER_AUTH_SECRET`, `API_KEY`, `ADMIN_EMAIL`, and the S3 credentials. Secrets such as `BUSINESS` values go into the production environment only.
3. Run the one-off migration and admin bootstrap against the external database:

   ```sh
   docker run --rm -e DATABASE_URL=... -e ADMIN_EMAIL=hey@omidshabab.com \
     dastyaresocial/ds-cs:latest \
     sh -c "npm run db:migrate && npm run bootstrap:admin"
   ```

4. Deploy: `vercel --prod`

Note that client-facing `NEXT_PUBLIC_*` branding is inlined into the image at build time; the source of truth for channel branding is `config/app.config.yml` and `config/about.config.yml` in this repo.

## Environment variables

`NEXT_PUBLIC_*` variables are client-facing build-time values. Everything else is a server-only secret.

| Variable | Notes |
| --- | --- |
| `DATABASE_URL` | Postgres connection string. Locally: `postgresql://postgres:postgres@db:5432/dastyare_social_cs` |
| `ADMIN_EMAIL` | Admin account email (default `hey@omidshabab.com`). Change after install. |
| `ADMIN_PASSWORD` | Admin account password. Set a real one after install. |
| `API_KEY` | Required API key for posts/stories automation; `MCP_API_KEY` (optional) falls back to this. |
| `API_KEY_RATE_LIMIT_MAX_REQUESTS` | Rate limit window budget (default 30). |
| `API_KEY_RATE_LIMIT_WINDOW_MS` | Rate limit window in ms (default 60000). |
| `BETTER_AUTH_URL` | Public URL of the app, used for auth. |
| `NEXT_PUBLIC_APP_URL` | Public URL of the app. |
| `BETTER_AUTH_SECRET` | Auth session secret (auto-generated on install). |
| `S3_ENDPOINT` | Local: `http://rustfs:9000` (bundled emulation). |
| `S3_REGION` | Region for the S3 bucket (e.g. `us-east-1`). |
| `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | Set by the installer for the bundled emulation, or your real provider creds. |
| `S3_BUCKET_NAME` | Bucket name (bundled stack creates `ds-cs` automatically). |
| `S3_FORCE_PATH_STYLE` | `true` for path-style addresses (bundled emulation). |
| `S3_PUBLIC_BASE_URL` | Public base URL of the S3 bucket (needed on Vercel). |
| `DS_SH_URL` / `DS_SH_API_KEY` | Optional external shortener integration (empty by default). |
| Webpush keys (`WEBPUSH_PUBLIC_KEY`, `WEBPUSH_PRIVATE_KEY`, `WEBPUSH_SUBJECT`) | Optional push notifications; empty by default. |

## Config files

`config/` holds the channel identity served at runtime:

- `app.config.yml` — site identity: username `omidshabab`, admin email, name "Omid Shabab".
- `about.config.yml` — the /resume page: founder profile, work experience, education, contacts.

Edit these, then restart the stack (or redeploy) to apply.

## Notes

- The Compose project name is pinned to `ds-cs`, so container names are stable. Keep only one deployment of this stack per host.
- PostHog analytics is disabled by default in the environment file.
- `.env` is git-ignored — never commit secrets. `.env.example` documents every supported variable.
- Methods 1 and 2 run the full stack (Postgres, S3 emulation, app) locally via Docker. Method 3 is Vercel only and uses external Postgres/S3.