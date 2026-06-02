# Security Review

## Assets
- `CRON_TOKEN` (API bearer token) used to authorize manual triggers.
- Database credentials in `DATABASE_URL`.
- Proxy credentials potentially embedded in `HTTP_PROXY_URL`.
- Stored market data and derived values in PostgreSQL.

## Trust Boundaries
- External web scraping target (Barchart): untrusted content and availability.
- External exchange-rate API: untrusted availability and response integrity.
- Inbound HTTP requests: potentially untrusted network clients if exposed.

## Primary Threats
- **Unauthorized job triggers**
  - Risk: token leakage or weak token default.
  - Mitigations:
    - Set strong `CRON_TOKEN` in production; do not rely on defaults.
    - Restrict network access (firewall/VPC rules) so only trusted callers can reach the service.
- **Secret exposure via logs**
  - Risk: logging `Authorization` headers or proxy URLs containing credentials.
  - Notes:
    - Token validation lives in [src/index.ts](file:///Users/a/Documents/TRAE/ASLI/asli-pangan-supply-onchain-price-cron/src/index.ts). Avoid logging request headers/tokens in production logs.
  - Mitigations:
    - Redact secrets from logs; enforce log retention and access controls.
- **Supply-chain / dependency risks**
  - Puppeteer and related packages are large and frequently updated; keep dependencies patched.
  - Use lockfiles and periodic vulnerability scanning.
- **Outbound request abuse**
  - Scraper makes outbound network requests; if the service is compromised it can be used for data exfiltration.
  - Mitigations:
    - Limit outbound egress to required domains where possible.
    - Run with least privilege and read-only filesystem where feasible.
- **Denial of Service**
  - Puppeteer can be resource intensive; repeated triggers can exhaust CPU/RAM.
  - Mitigations:
    - Keep `isCronRunning` guard for scheduled ticks.
    - Add rate limiting at ingress layer (reverse proxy/WAF).
    - Consider concurrency controls for manual triggers.

## Recommended Security Controls (Deployment)
- Run behind a reverse proxy with TLS termination.
- Restrict inbound traffic to trusted IPs or private network.
- Store secrets in a secret manager; inject via env at runtime.
- Run as non-root in containers/VMs; drop unnecessary capabilities.
- Ensure `LOG_DIR` is writable but does not expose logs publicly.

## Security Test Checklist
- Verify `POST` endpoints reject missing/invalid bearer tokens.
- Verify logs do not contain bearer tokens, database URLs, or proxy credentials.
- Verify dependency update process exists (monthly patch window or automated alerts).
