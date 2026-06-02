# Test Plan

## Scope
Test coverage for:
- HTTP endpoints and bearer token enforcement
- Scheduled execution behavior (cron validation, enable/disable)
- Scrape-to-database pipeline (RM and KC)
- Derived value calculations (MA30, deltas, IDR conversion, discount values)
- Resilience behaviors (retry/backoff, graceful shutdown)

## Test Environments
- Local: Node.js 20+, PostgreSQL, Chromium available (or configured executable path).
- Staging/Prod-like: container or VM with headless Chromium, outbound network egress, stable IP/proxy if required.

## Preconditions
- `DATABASE_URL` points to a clean test database.
- Prisma client generated and migrations applied.
- `CRON_TOKEN` set and known.

## Manual Test Cases

### 1) Health Endpoint
- `GET /health` returns:
  - `status: ok`
  - `cronSchedule`, `cronTimezone`, `cronEnabled`, `cronIsRunning`
  - `uptimeSeconds` increments over time

### 2) Auth Enforcement
- `POST /cron` without `Authorization` header → `401`
- `POST /cron` with wrong token → `401`
- `POST /cron` with correct token → `200` and `success: true`

Repeat for:
- `POST /scrape/rm`
- `POST /scrape/kc`

### 3) Deduplication by Trade Date
- Trigger `POST /scrape/rm` twice on the same day:
  - First call inserts a `MarketData` row for `ROBUSTA` for the computed trade date.
  - Second call fails safely (should not insert a duplicate).
  - Confirm uniqueness at DB layer: `@@unique([type, tradeDate])`.

### 4) Derived Values: MA30 and Deltas
- Ensure there are N historical rows (e.g., seed 5–30 rows) for a commodity.
- Trigger a scrape.
- Validate:
  - `ma30` equals mean of available historical `closePrice` values (up to last 30).
  - `ma30Change` equals `ma30 - previous_ma30` (where previous is most recent stored MA30).
  - `changePercent` equals `priceChange / previousClose * 100` (or 0 when previousClose is 0).

### 5) IDR Conversion
- Normal path: outbound access to exchange-rate endpoint.
  - Validate `idrRate` is populated and `idrPrice` fields are computed.
- Fallback path: block outbound access (or simulate failure).
  - Validate the fallback rate is used and the service still completes.

### 6) Discount Value Generation
- Ensure `MaDiscountSetting` exists for the commodity with at least one stage.
- Trigger a scrape.
- Validate for the new `MarketData` row:
  - `MaDiscountValue` rows exist for each applicable setting.
  - `discountedIdrMa30` matches `idrMa30 + (idrMa30 * discount / 100)`.

### 7) Retry/Backoff Behavior
- Simulate a transient scrape failure (e.g., block Barchart for first attempt then allow).
- Trigger `POST /cron`.
- Validate logs show attempt counts and exponential delays.

### 8) Cron Disabled
- Set `CRON_ENABLED=false` and start the service.
- Validate that scheduled job is not registered, but manual endpoints still work.

### 9) Shutdown Handling
- Send `SIGINT` or `SIGTERM`.
- Validate the service closes the HTTP server and disconnects Prisma cleanly.

## Automation Opportunities (Future)
- Unit tests for MA30 computation and discount computation (pure functions extracted).
- Integration tests with a local Postgres container.
- Contract tests with stubbed Barchart JSON payloads (avoid relying on live scraping).
