# Documentation Standard

## Purpose
Keep project documentation consistent, easy to maintain, and aligned with how the service actually behaves.

## Document Placement
- Role-oriented docs live under `docs/`:
  - `01_Product_Manager/`
  - `02_System_Architect/`
  - `03_Tech_Lead/`
  - `04_Orchestrator/`
  - `05_Full_Stack_Engineer/`
  - `06_Test_Engineer/`
  - `07_Security_Engineer/`
  - `08_DevOps_Engineer/`
  - `09_Technical_Writer/`

## Writing Rules
- Prefer short sections with clear headings.
- Use explicit defaults (e.g., default cron schedule/timezone).
- When documenting behavior, link to the actual source file that implements it.
- Avoid copying large chunks of code; describe behavior and constraints instead.

## Required Content Checklist (for New Features)
- What problem it solves and who uses it.
- How to run it locally and in production.
- How to validate it (health checks, expected DB rows).
- Operational risks and troubleshooting steps.
- Security considerations (secrets, auth, logging).

## Terminology
- **RM**: Robusta coffee futures (Barchart RM*0 series used to locate active contract).
- **KC**: Arabica coffee futures (Barchart KC*0 series used to locate active contract).
- **Trade date**: the date field associated with the daily quote used for the persisted record.
- **MA30**: moving average computed from up to the last 30 stored close prices.
- **Discount setting/value**: configuration-driven price adjustment outputs stored alongside market data.

## Update Process
- Update docs in the same change that modifies behavior.
- Ensure links to code still resolve after refactors.
- Keep environment variable documentation synchronized with the implementation.
