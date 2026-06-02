# Engineering Guide

## Repository Purpose
Supply Onchain Price Cron is a TypeScript/Node.js service that scrapes daily coffee futures prices (RM/KC) from Barchart, stores results in PostgreSQL (Prisma), computes MA30 + deltas, generates discount values from database configuration, and updates CCR-related values.

## Local Development
- Install dependencies:
  - `npm install`
- Configure environment:
  - `cp .env.example .env`
  - Set at minimum: `DATABASE_URL` and `CRON_TOKEN`
- Generate Prisma client:
  - `npm run prisma:generate`
- Apply schema migrations:
  - `npm run prisma:migrate`
- Run in watch mode:
  - `npm run dev`

## Build & Run
- Build:
  - `npm run build`
- Run:
  - `npm start`

## Project Structure (Key Files)
- HTTP server + cron scheduling: [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts)
- Scraping + persistence + computations: [price-index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/lib/price-index.ts)
- CCR update logic: [ccr.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/lib/ccr.ts)
- Prisma schema: [schema.prisma](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/prisma/schema.prisma)
- Prisma client singleton: [db.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/server/db.ts)

## Engineering Conventions (Project-Specific)
- **Configuration via env:** behavior is controlled via environment variables (cron schedule, timezone, token, log dir, puppeteer executable path, proxy).
- **Idempotency by date:** the `(type, tradeDate)` unique constraint is a core invariant; avoid changes that could break it.
- **Database writes:**
  - Create `MarketData` first.
  - Generate `MaDiscountValue` based on `MaDiscountSetting` for that commodity.
- **Post-processing:** CCR updates run after each commodity update; treat them as best-effort to avoid blocking the scrape.

## Operational Readiness Checklist (Before Shipping)
- Cron schedule validates and is correct for the target timezone.
- `CRON_TOKEN` set to a strong secret in production.
- Chromium path is correct in the target runtime (Docker/VM).
- Database connectivity and migrations applied.
- Logs are persisted and rotated (log directory writable).
- Proxy config set if scraping is blocked.

## Common Change Areas
- **Scraper reliability:** changes in Barchart HTML/API interception logic.
- **Unit conversion correctness:** confirm units for RM (USD/tonne) and KC (¢/lb) and the IDR conversion math.
- **Schema evolution:** Prisma migrations must be applied in deployment workflow.
