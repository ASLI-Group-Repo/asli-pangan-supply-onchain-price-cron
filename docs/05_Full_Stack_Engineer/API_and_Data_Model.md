# API and Data Model

## HTTP API
Server: Express in [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts)

### Authentication
For all `POST` routes:
- Header: `Authorization: Bearer <CRON_TOKEN>`

### Endpoints
- `GET /health`
  - Purpose: health check + runtime configuration snapshot.
  - Response includes: `cronSchedule`, `cronTimezone`, `cronEnabled`, `cronIsRunning`, `uptimeSeconds`.
- `POST /cron`
  - Purpose: runs `scrapeRm()` then `scrapeKC()` sequentially with retries per task.
- `POST /scrape/rm`
  - Purpose: runs Robusta scrape only.
- `POST /scrape/kc`
  - Purpose: runs Arabica scrape only.

## Environment Variables (Implemented in Code)
From [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts) and [price-index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/lib/price-index.ts):
- `DATABASE_URL` (required): PostgreSQL connection string (used by Prisma).
- `CRON_TOKEN` (recommended): bearer token for `POST` endpoints.
- `PORT` (default `3000`): HTTP port.
- `CRON_SCHEDULE` (default `0 3 * * *`): cron expression (5-field).
- `CRON_TIMEZONE` (default `Asia/Bangkok`): cron timezone.
- `CRON_ENABLED` (default enabled): set to `false` to disable scheduled execution.
- `LOG_DIR` (default `logs`): directory for daily log files.
- `PUPPETEER_EXECUTABLE_PATH` (optional): path to chromium/chrome executable.
- `HTTP_PROXY_URL` (optional): proxy URL; supports embedding username/password.

## Database Access
Prisma client singleton: [db.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/server/db.ts)

## Core Data Models (Relevant Subset)
All models come from [schema.prisma](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/prisma/schema.prisma).

### MarketData
Represents a single daily record for one commodity:
- Key fields: `type`, `tradeDate`, OHLC (`openPrice`, `highPrice`, `lowPrice`, `closePrice`), `volume`, `openInterest`
- Derived fields:
  - `ma30`, `ma30Change`
  - `idrRate`, `idrPrice`, `idrPriceChange`, `idrMa30`, `idrMa30Change`
- Constraints:
  - `@@unique([type, tradeDate])` ensures one entry per commodity/day

### MaDiscountSetting
Configuration for discount computation:
- Fields: `commodity`, `type` (stage), `discount` (percentage)

### MaDiscountValue
Generated outputs per `MarketData` × `MaDiscountSetting`:
- Fields include: `discountedMa30`, `discountedIdrMa30`, `discountPercentage`
- Uniqueness is enforced by a compound key (see prisma schema)

## Application Logic Notes (for API Consumers)
- The service treats each “trade date” as a daily snapshot based on Barchart’s `dailyDate1dAgo` field.
- Deduplication is performed both in application logic (existence check) and at DB level (unique constraint).
- CCR updates are triggered after the `MarketData` insert and discount generation; failures in CCR updates should not invalidate the scrape.
