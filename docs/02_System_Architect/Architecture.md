# System Architecture

## Scope
Architecture for the Supply Onchain Price Cron service: a Node.js/TypeScript process that hosts an HTTP API and a background scheduler which scrapes and persists coffee futures price data and derived values.

## High-Level Components
- **HTTP Server (Express):** exposes health and token-protected trigger endpoints.
- **Scheduler (node-cron):** runs the main job on a schedule (or disabled via configuration).
- **Scraper (Puppeteer):** navigates Barchart pages and intercepts the internal quote JSON response.
- **Computation Layer:** MA30 computation, change deltas, IDR conversion, discount value generation.
- **Persistence Layer (Prisma + PostgreSQL):** stores MarketData and discount outputs, reads discount settings.
- **CCR Updater:** recalculates CCR values after each commodity update.

Primary entrypoint: [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts)  
Scraper and computation: [price-index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/lib/price-index.ts)  
Database client: [db.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/server/db.ts)

## Data Flow
1. **Trigger**
   - Scheduled tick (cron) or manual trigger (`POST /cron`, `POST /scrape/rm`, `POST /scrape/kc`).
2. **Scrape**
   - Launch headless Chromium.
   - Load Barchart futures page (RM*0 or KC*0).
   - Intercept the `/proxies/core-api/v1/quotes/get` JSON response.
   - Extract the active contract symbol from the page header and select its quote data.
3. **Transform**
   - Map Barchart quote fields to internal values.
   - Fetch USD→IDR exchange rate (with fallback).
   - Compute MA30 and MA30 change (based on available history).
   - Compute IDR fields and deltas.
4. **Persist**
   - Insert `MarketData` row for `(type, tradeDate)` with unique constraint preventing duplicates.
   - Generate `MaDiscountValue` rows from `MaDiscountSetting` configuration.
5. **Post-Processing**
   - Trigger CCR updates for downstream entities.

## External Dependencies
- **Barchart (web + internal JSON endpoint):** source of RM/KC daily prices.
- **Exchange rate API:** `https://api.exchangerate-api.com/v4/latest/USD` for USD→IDR conversion.
- **PostgreSQL:** primary storage.
- **Chromium/Chrome:** required for Puppeteer in production environments (path configurable).

## Configuration Surfaces
- Runtime env vars described in root [README.md](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/README.md).
- Additional scraper-related env vars referenced in code:
  - `HTTP_PROXY_URL` for proxy routing + optional basic auth.
  - `PUPPETEER_EXECUTABLE_PATH` for Chromium location.

## Key Data Models (Relevant Subset)
From [schema.prisma](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/prisma/schema.prisma):
- `MarketData`: one row per commodity per trade date, includes OHLCV, MA30, IDR conversions, deltas.
- `MaDiscountSetting`: configuration per commodity + stage type, holds a percentage.
- `MaDiscountValue`: generated outputs per `MarketData` × `MaDiscountSetting`.

## Runtime Characteristics
- **Long-running process:** the service stays up to execute scheduled tasks and accept manual triggers.
- **Idempotency by date:** duplicates are avoided using a unique constraint and explicit existence check.
- **Retry strategy:** exponential backoff retry around the commodity scrapes.
- **Logging:** console + file logs (daily file name).

## Failure Modes (Architectural)
- Barchart blocked or API changes → scrape failure; recovered by retries; may require proxy.
- Exchange-rate API outage → safe fallback rate used, reduces accuracy.
- DB unavailable → write failure; cron tick fails; requires operational response.
- Puppeteer executable missing → scrape failure; requires deployment configuration.
