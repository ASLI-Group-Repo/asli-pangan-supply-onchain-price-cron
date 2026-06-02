# Product Requirements Document (PRD)

## Overview
This service runs as a small HTTP application with a scheduled cron job that:
- Scrapes Robusta (RM) and Arabica (KC) coffee futures daily prices from Barchart.
- Persists the daily OHLCV snapshot to PostgreSQL.
- Computes a rolling moving average (MA30) and deltas.
- Converts the USD-derived prices to IDR using a live exchange rate (with fallback).
- Derives discount values from configurable discount settings (per commodity + stage) and stores them.
- Triggers CCR recalculation for relevant entities after each successful price update.

## Problem Statement
Downstream supply-chain calculations require a consistent, daily, auditable price index for Arabica and Robusta with:
- A stable moving-average reference (MA30), not only spot price.
- Prices expressed in IDR for local business calculations.
- Derived “discounted” price variants per supply-chain stage (configurable).

## Goals
- Collect daily price data for RM (Robusta) and KC (Arabica) and store it once per date (deduplicated).
- Produce MA30, MA30 change, IDR conversion, and IDR deltas for each commodity.
- Produce stage-based discounted price values driven by configuration in the database.
- Provide a simple operational interface (health + protected trigger endpoints).

## Non-Goals
- Real-time streaming prices or intraday updates.
- A public-facing API (endpoints are intended for internal orchestration).
- UI/dashboard (monitoring is via logs/health and database inspection).

## Users / Stakeholders
- Operations / Orchestrator: runs and monitors scheduled tasks, retries, and failures.
- Engineering: maintains scraper reliability and data correctness.
- Data / Business: consumes stored MarketData and derived discount outputs.
- Security / Compliance: ensures safe secret handling, minimal exposure, and auditability.

## Key Functional Requirements
- **Scheduling:** configurable cron schedule + timezone; ability to disable cron.
- **Manual triggers:** token-protected endpoints to run full task or per-commodity scrape.
- **Data persistence:** write one record per commodity per trade date; prevent duplicates.
- **Computation:**
  - MA30 computed over up to last 30 available records.
  - MA30 change computed vs the most recent historical MA30.
  - IDR conversion fetched from an external API with a safe fallback.
  - Discount values generated from discount settings for the new MarketData record.
- **Resilience:** retry with exponential backoff for scrape tasks.
- **Observability:** logs written to console and rotating daily log files.

## Success Metrics
- ≥ 99% days successfully scraped and persisted for both commodities.
- No duplicate rows for `[type, tradeDate]`.
- MA30 and IDR fields present for each successful daily record.
- Mean time to detect failures: minutes (via health/log monitoring).

## Constraints & Dependencies
- Barchart access (scrape reliability depends on site behavior and potential IP/CDN blocking).
- Headless Chromium availability in runtime environments.
- PostgreSQL availability.
- Exchange-rate API availability (fallback covers temporary failures).

## Risks & Mitigations (Product Level)
- **Scraper breaks due to UI/API changes:** include runbook and quick validation steps; isolate scraping logic.
- **Blocked by anti-bot:** support proxy configuration; monitor failure rates.
- **Incorrect unit conversion:** document unit assumptions and validate outputs periodically.
- **Misconfigured discount settings:** validate presence/coverage of settings per stage in DB.

## Out of Scope (Future Enhancements)
- Alerts (Slack/email) on repeated failures.
- Dashboard for last-run status and recent data diffs.
- Automated seed/setup for discount settings.
