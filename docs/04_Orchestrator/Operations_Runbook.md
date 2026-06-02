# Operations Runbook

## What This Service Does
- Runs an HTTP server with a scheduled cron task.
- Scrapes RM (Robusta) and KC (Arabica) daily prices from Barchart.
- Stores results to PostgreSQL and generates derived values (MA30, IDR conversions, discount values).

## Key Endpoints
All `POST` endpoints require: `Authorization: Bearer <CRON_TOKEN>`
- `GET /health` — health/status snapshot (cron schedule, timezone, uptime).
- `POST /cron` — run full task (RM then KC).
- `POST /scrape/rm` — run Robusta only.
- `POST /scrape/kc` — run Arabica only.

Implementation: [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts)

## Scheduling
- Default: `CRON_SCHEDULE=0 3 * * *` with `CRON_TIMEZONE=Asia/Bangkok`.
- Disable scheduler (keep HTTP server running): `CRON_ENABLED=false`.

## Logs
- Logs are printed to stdout and also written to daily files:
  - Directory: `LOG_DIR` (default: `logs`)
  - File name pattern: `app-YYYY-MM-DD.log`

## Common Operational Tasks
- **Trigger a run manually**
  - `curl -X POST http://localhost:3000/cron -H "Authorization: Bearer <CRON_TOKEN>"`
- **Validate cron configuration**
  - `GET /health` returns `cronSchedule`, `cronTimezone`, `cronEnabled`, and `cronIsRunning`.
- **Check deduplication**
  - The database enforces `@@unique([type, tradeDate])` on `MarketData`.
  - Re-running on the same date should not create duplicates; it should fail safely.

## Troubleshooting
- **Scrape fails with HTTP 4xx/5xx**
  - Likely IP/CDN block or upstream changes at Barchart.
  - Actions:
    - Confirm Chromium is available and launches.
    - Configure `HTTP_PROXY_URL` if needed.
    - Re-run `POST /scrape/rm` or `POST /scrape/kc` to isolate which scrape fails.
- **No data intercepted**
  - The scraper intercepts `/proxies/core-api/v1/quotes/get`.
  - Actions:
    - Confirm the Barchart page loads.
    - Check for upstream API changes.
- **Exchange-rate API down**
  - Service falls back to a hardcoded rate; IDR fields may be less accurate.
  - Actions:
    - Monitor for recovery; consider alerting if fallback is used frequently.
- **Database errors**
  - Actions:
    - Validate `DATABASE_URL`.
    - Ensure migrations are applied.
    - Check DB connectivity and disk space.

## Safe Restart
- Process handles `SIGINT` and `SIGTERM` by closing the HTTP server and disconnecting Prisma.
