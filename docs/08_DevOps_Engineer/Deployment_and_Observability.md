# Deployment and Observability

## Deployment Options
- Bare VM / server process (Node.js 20+)
- Containerized deployment (Dockerfile included)

## Container Build (Docker)
Dockerfile: [Dockerfile](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/Dockerfile)
- Installs system Chromium and configures Puppeteer to use it:
  - `PUPPETEER_SKIP_DOWNLOAD=true`
  - `PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium`
- Builds TypeScript into `dist/` and prunes dev dependencies.
- Writes logs to `logs/` directory in the container.

## Required Runtime Configuration
- `DATABASE_URL` must be provided in all environments.
- `CRON_TOKEN` must be set to a strong secret in production.
- Optional operational tuning:
  - `CRON_SCHEDULE`, `CRON_TIMEZONE`, `CRON_ENABLED`
  - `LOG_DIR`
  - `HTTP_PROXY_URL` if scraping is blocked

## Database Migrations
- Local development uses:
  - `npm run prisma:migrate` (dev workflow)
- For production:
  - Apply migrations as part of a controlled deployment step (outside of build), using Prisma’s production workflow (recommended).

## Health Checks
- `GET /health` provides:
  - service status and timestamp
  - cron schedule, timezone, enabled flag
  - whether a cron execution is currently running
  - uptime

## Logging
- Logs are emitted to stdout and appended to daily log files in `LOG_DIR` (default `logs`).
- Operational recommendations:
  - Ship stdout logs to a centralized log system.
  - Enforce log retention and protect logs because they may contain sensitive operational data.

## Resource Considerations
- Puppeteer/Chromium is CPU and memory heavy.
- Recommendations:
  - Allocate sufficient memory for headless Chromium.
  - Set CPU limits appropriately if containerized.
  - Consider running behind a job scheduler/queue if manual triggers are frequent.

## Common Production Failure Modes
- Chromium missing/misconfigured path → scraper fails immediately.
- Barchart blocked → scraper fails; configure proxy and stabilize egress.
- Database unavailable or migrations missing → insert failures.
- Exchange-rate API unavailable → fallback rate used (reduced accuracy).
