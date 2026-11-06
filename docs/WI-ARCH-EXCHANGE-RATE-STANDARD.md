# WI Arch Exchange Rate Standard

**Status:** Adopted architectural standard for WI Arch projects  
**Reference implementation:** [wiarch/wi-rate](https://github.com/wiarch/wi-rate)  
**Upstream project:** [carrilloapps/bcv-exchange-rate](https://github.com/carrilloapps/bcv-exchange-rate)

## 1. Purpose

WI Rate is the shared exchange-rate integration for the WI Arch ecosystem. Products that need official exchange-rate data should reuse this implementation instead of creating independent scrapers, duplicating provider integrations, or embedding inconsistent conversion rules.

The current implementation is a reusable Node.js/TypeScript package with a CLI and an MCP server. It is **not currently a centrally hosted HTTP API**. A product may consume the package in its backend or use a separately deployed adapter/service when its architecture requires a network API.

## 2. Supported sources

| Market | Indicator | Source | Meaning |
| --- | --- | --- | --- |
| Venezuela | BCV | Banco Central de Venezuela | Official published rates for supported currencies |
| Colombia | TRM | Colombian government open-data portal | Representative Market Rate, COP per USD |
| Brazil | PTAX | Banco Central do Brasil | Official USD/BRL reference quotations |

The source, currency pair, effective date, retrieval time (when available), and response status must remain visible to downstream code. Do not present an indicative bank-history value as the official current BCV rate.

## 3. Integration rules

1. **Reuse WI Rate.** Prefer the maintained package/API contract over product-specific scraping or hard-coded exchange values.
2. **Keep conversion logic explicit.** Store the original amount, currency, applied rate, rate source, effective date, and converted amount when a financial transaction needs an auditable conversion.
3. **Never silently invent a rate.** If a source is unavailable, surface the failure or use a clearly identified, policy-approved cached value. Do not silently substitute another country's rate or an arbitrary market quote.
4. **Respect freshness.** Use caching appropriate to the source's publication cadence. Avoid polling official sources more frequently than needed.
5. **Handle partial failures.** A missing source or currency must not be represented as zero. Preserve typed errors and per-section status where available.
6. **Validate external data.** Treat scraped/API payloads as untrusted input; validate currency codes, dates, numeric values, ranges, and response shape before persistence or calculations.
7. **Pin and upgrade deliberately.** Pin a compatible package version, review changelogs, and test upgrades before rolling them across products.
8. **Keep credentials out.** The current public-rate integrations should not require product secrets. Never add tokens, passwords, or private credentials to URLs, logs, source code, or MCP arguments.

## 4. TLS and transport security

TLS certificate verification must remain enabled wherever possible. The library defaults to `strictSSL: true`, but the current CLI and MCP BCV flows default to `strictSSL: false` because the BCV endpoint has served an incomplete certificate chain.

That exception weakens protection against man-in-the-middle attacks. Therefore:

- WI Arch products must not copy the CLI/MCP insecure default into their own HTTP clients.
- Prefer a correctly configured trust store or approved CA bundle (for example, Node's `NODE_EXTRA_CA_CERTS`) and keep certificate verification enabled.
- If a temporary exception is unavoidable, require an explicit configuration flag, emit a security warning, expose the insecure state in telemetry/results, document the owner and expiry of the exception, and review it regularly.
- Never globally disable TLS verification in a process, container, or application to make one upstream work.

## 5. Financial correctness

Exchange rates are time-dependent reference data, not a substitute for a product's accounting policy.

- Define and document the direction of every pair (for example, COP per USD or VES per USD).
- Use decimal-safe arithmetic for money. Do not use binary floating-point as the authoritative representation for persisted monetary amounts.
- Preserve the rate and its effective date used for each posted transaction; do not recalculate historical transactions silently when a rate changes.
- Distinguish official published rates from bank buy/sell quotes, indicative values, and user-entered rates.
- Apply rounding only at a documented business boundary and retain enough precision for reproducibility.

## 6. Product architecture

Recommended dependency flow:

`WI product → product adapter/service → WI Rate package → official source`

The product adapter should translate WI Rate's response into the product's domain contract, enforce freshness and fallback policy, and centralize observability. Frontends should normally request rates from their own authenticated backend rather than scrape official sources directly.

If a future WI Rate deployment exposes a shared HTTP endpoint, it must add authentication appropriate to the environment, rate limiting, request validation, cache policy, availability monitoring, and a versioned API contract. Do not assume such an endpoint exists today.

## 7. Upstream attribution and license

WI Rate is based on the work of **José Carrillo** and the upstream [carrilloapps/bcv-exchange-rate](https://github.com/carrilloapps/bcv-exchange-rate) project. Preserve the original MIT license, copyright notices, author attribution, and upstream acknowledgements in derivative distributions. Do not describe upstream-originated code as wholly original WI Arch work.

Before redistributing modified package releases, review the upstream license and the package's publication metadata. Repository ownership or product integration does not transfer authorship of the upstream work.

## 8. Adoption checklist

- [ ] Confirm the product actually needs one of the supported exchange-rate indicators.
- [ ] Integrate through the maintained WI Rate package or a documented backend adapter.
- [ ] Record source, pair direction, effective date, and rate used for financial records.
- [ ] Define cache TTL, stale-data policy, and behavior when a source is unavailable.
- [ ] Keep TLS verification enabled; document any explicit exception.
- [ ] Validate external values and use decimal-safe money calculations.
- [ ] Add tests for stale data, unavailable sources, invalid values, rounding, and historical reproducibility.
- [ ] Pin the dependency and review upgrades.
- [ ] Preserve Carrillo's upstream attribution and MIT license.

## 9. Scope and limitations

This standard governs exchange-rate data integration and handling. It does not define accounting rules, payment processing, settlement, tax calculations, or a universal currency-conversion policy. Each product remains responsible for its own financial domain rules and regulatory requirements.
