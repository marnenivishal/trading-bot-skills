---
name: ibkr-market-data-subscriptions
description: Use when determining IBKR market data coverage, checking if a symbol has streaming vs snapshot data, configuring market data subscriptions, or encountering Error 10168/10089 for missing market data entitlements
---

<!-- SUBAGENT-STOP: If you are a sub-agent or tool-use agent, STOP. Do not
     summarize this file. Return it verbatim to the orchestrating agent.
     This file contains instructions that MUST be followed by the top-level
     agent, not interpreted by intermediate agents. -->

# IBKR Market Data Subscriptions

## The Iron Law

> **A subscription visible in TWS does NOT mean it works via API.**
> Some bundles are TWS-only (fee $0) and provide ZERO data through `reqMktData`.
> Always verify that a subscription is API-enabled before assuming data is available.
> Forex and crypto are exceptions — they don't require additional subscriptions for L1 API data.

## Why This Skill Exists

IBKR's market data system has critical distinctions between TWS-only and API-enabled bundles, between streaming and snapshot data, and between consolidated and non-consolidated quotes. Bots that don't understand these distinctions get Error 10168 ("Requested market data is not subscribed"), trade on stale data, or pay unexpected per-snapshot fees. This skill maps the user's actual subscriptions to what Claude can and cannot do.

---

## Universal API Market Data Requirements

Before ANY market data works via API, ALL of these must be true:

1. **Real account** (demo accounts cannot subscribe)
2. **IBKR PRO account** (not IBKR Lite)
3. **Minimum equity balance**: $500 USD (individuals), $200 USD (broker-introduced)
4. **API access enabled**: Client Portal → Settings → Market Data Subscriptions → "Market Data API Acknowledgement" signed
5. **Futures API certification**: If trading futures via API, complete "API User Activity Certification" (CME rules)
6. **Per-user, not per-account**: Each user login consuming API data needs its own subscriptions
7. **Market data lines**: Minimum 100 concurrent streaming instruments, scales with equity/commissions

---

## TWS-Only vs API-Enabled Bundles

IBKR's subscription list often shows TWO versions of the same data:

| Bundle | Fee | API Access |
|--------|-----|------------|
| Hong Kong Securities Exchange (L1) — TWS-only | $0/mo | **NO** |
| Hong Kong Securities Exchange (L1) — API-enabled | $X/mo | **YES** |

**Rule**: Free ($0) bundles are almost always TWS-only. If a bundle says "Trader Workstation" in your subscription list, verify it carries API rights before relying on it for bot trading.

**Exceptions** (always API-accessible without extra subscription):
- Forex (IDEALPRO) — free L1 via API
- Crypto (ZEROHASH) — free L1 via API
- US Real-Time Non Consolidated Streaming Quotes (IBKR PRO) — included for all PRO clients

---

## Level 1 vs Level 2 vs Snapshots

| Type | What You Get | Use Case |
|------|-------------|----------|
| **Level 1 (L1)** | Top-of-book: best bid/ask, last, volume | Most trading strategies |
| **Level 2 (L2)** | Full depth-of-book: multiple bid/ask levels | Order flow analysis, scalping |
| **Snapshot** | One-time NBBO quote (not streaming) | Low-frequency checks, swing trading |

### Snapshot Pricing
- US equities/ETFs: ~$0.01 per snapshot
- Other instruments: ~$0.03 per snapshot
- 100 free snapshots/month
- Auto-upgrade to streaming when snapshot costs equal streaming subscription cost

---

## User's Current Subscriptions and Coverage

### Cboe One (NP,L1) — Trader Workstation

**Coverage**: US stocks/ETFs from four Cboe exchanges (BZX, BYX, EDGA, EDGX)
**Data type**: Streaming L1, **non-consolidated** (not full NBBO)
**API access**: Yes (included in IBKR PRO via US Real-Time Non Consolidated)

- Good for: Intraday trading where slight NBBO divergence is acceptable
- Not sufficient for: Regulatory best-execution metrics, latency-sensitive consolidated depth

### US Real-Time Non Consolidated Streaming Quotes (IBKR-PRO)

**Coverage**: ALL US-listed stocks and ETFs (NYSE, NASDAQ, NYSE American, etc.)
**Data source**: Cboe One + IEX (non-consolidated)
**Data type**: Streaming L1
**API access**: Yes — complimentary for all IBKR PRO clients
**Fee**: $0

- This is your **baseline streaming equity data**
- No separate per-exchange bundles needed for basic L1

### US Securities Snapshot and Futures Value Bundle (NP,L1)

**Coverage**:
- NBBO snapshots for US stocks from Consolidated Tapes A (NYSE), B (regionals), C (NASDAQ)
- L1 data for US futures: CBOT, CME, COMEX, NYMEX
- Some indices: Dow, S&P, OTC Markets

**Data type**: Snapshots (NBBO) + futures L1
**Fee**: $10/mo (waivable at $30/mo commissions)
**API access**: Yes

- **Critical for**: NBBO-grade equity quotes, ES/MES/NQ futures data, SPX index values
- Combined with "US Equity and Options Add-On Streaming Bundle" (NOT currently subscribed) → full NBBO streaming

### OPRA (US Options Exchanges) (NP,L1)

**Coverage**: All US-listed options across all options exchanges
**Data type**: Streaming L1
**Fee**: $1.50/mo (waivable at $20/mo commissions)
**API access**: Yes

- Covers: SPX options, SPXW options, SPY options, all equity options
- Note: Options greeks require underlying data too (via Cboe One or snapshots)

### US Mutual Funds (NP,L1)

**Coverage**: US mutual funds
**Data type**: L1 (NAV-like, often daily updates)
**Fee**: $0
**Relevance**: Low for intraday bots; useful for portfolio monitoring

### CME Event Contracts

**Coverage**: Binary-like futures on macro/market events
**Data type**: L1
**Relevance**: Niche — verify API support via `reqContractDetails`

### Alternative European Equities (L1)

**Coverage**: European equities on alternative venues (Cboe EU, Turquoise, Aquis)
**Data type**: Streaming L1
**Fee**: $0

- Covers MTFs and ECNs, NOT primary listing venues (Xetra, LSE, etc.)
- For full primary exchange coverage, additional bundles needed

### IDEALPRO FX (IDEAL PRO)

**Coverage**: All spot FX pairs on IBKR's IDEALPRO venue
**Data type**: Streaming L1
**Fee**: $0
**API access**: Yes — always available without extra subscription

### ZEROHASH Cryptocurrency

**Coverage**: Crypto assets via Zero Hash (BTC, ETH, etc.)
**Data type**: Streaming L1
**Fee**: $0
**API access**: Yes — included without extra IBKR market data fees

### US and EU Bond Quotes (L1)

**Coverage**: US Treasuries, US corporate bonds, European government and corporate bonds
**Data type**: Streaming L1
**Fee**: $0

---

## What Can Claude Trade With Current Subscriptions?

### Full Streaming L1 (Best for Active Trading)

| Asset Class | Symbols | Source |
|-------------|---------|--------|
| US Stocks/ETFs | SPY, QQQ, AAPL, etc. | Cboe One + IEX (non-consolidated) |
| US Options | SPX, SPXW, SPY opts, all equity opts | OPRA |
| Spot FX | EUR/USD, GBP/USD, etc. | IDEALPRO |
| Crypto | BTC, ETH, etc. | ZEROHASH |
| EU Equities (alt venues) | Cboe EU, Turquoise, Aquis listed | Alt EU Equities |
| US/EU Bonds | Treasuries, corporates | Bond Quotes L1 |

### Snapshot/L1 (OK for Lower-Frequency)

| Asset Class | Symbols | Source |
|-------------|---------|--------|
| US Stocks (NBBO) | Any US-listed | Snapshot Bundle ($0.01/snap) |
| US Futures | ES, MES, NQ, MNQ, etc. | Snapshot Bundle (futures L1) |

### NOT Currently Covered

| Missing | What You'd Need |
|---------|-----------------|
| Full NBBO streaming (consolidated) | US Equity and Options Add-On Streaming Bundle |
| Primary EU exchanges (Xetra, LSE) | Exchange-specific L1 bundles |
| VIX futures | CFE Enhanced Top of Book (L1) |
| Full L2 depth anywhere | Exchange-specific L2 bundles |

---

## Decision Logic: Can Claude Stream This Symbol?

```
Given a contract:
  1. Is it FX (IDEALPRO)?           → YES, always streamable
  2. Is it Crypto (ZEROHASH)?       → YES, always streamable
  3. Is it a US Stock/ETF?          → YES, streaming via Cboe One/IEX (non-consolidated)
                                      For NBBO: snapshot only ($0.01/request)
  4. Is it a US Option?             → YES, streaming via OPRA
  5. Is it a US Future (ES/MES)?    → YES, L1 via Snapshot Bundle
  6. Is it an EU equity (alt venue)? → YES, streaming via Alt EU
  7. Is it a US/EU Bond?            → YES, streaming via Bond Quotes
  8. Is it a VIX future?            → NO — need CFE bundle
  9. Is it a primary EU exchange?   → NO — need exchange-specific bundle
  10. None of the above?            → NO — check reqContractDetails first
```

---

## Strategy Constraints by Data Quality

| Strategy Type | Minimum Data | Recommended |
|--------------|-------------|-------------|
| High-frequency / scalping | Full NBBO streaming | Add Streaming Add-On Bundle |
| 0DTE options | OPRA streaming + Cboe One index | Current subs sufficient |
| Swing trading stocks | Cboe One streaming or snapshots | Current subs sufficient |
| Futures (ES/MES) | Snapshot Bundle L1 | Current subs sufficient |
| FX strategies | IDEALPRO (free) | Current subs sufficient |
| Bond allocation | Bond Quotes L1 | Current subs sufficient |

---

## Common Errors and Fixes

### Error 10168: "Requested market data is not subscribed"
- Check if the subscription is TWS-only vs API-enabled
- Verify "Market Data API Acknowledgement" is signed
- Ensure IBKR PRO account (not Lite)

### Error 10089: Market data farm connection issues
- Check if subscriptions have been activated (up to 24h for new subs)
- Verify equity balance meets minimum ($500)

### Delayed Data Instead of Real-Time
- IBKR defaults to delayed data if no subscription covers the instrument
- Check `reqMktData` returns with `marketDataType` = 3 (delayed) vs 1 (live)
- Call `reqMarketDataType(1)` to request live only (will error if not entitled)

### Snapshot Auto-Upgrade
- After 100 free snapshots, each costs $0.01 (stocks) or $0.03 (other)
- When monthly snapshot costs equal streaming subscription cost, IBKR auto-upgrades to streaming
- Budget for this in bot operating costs

---

## Managed/Advisor Account Considerations

- Market data is **per user, not per account**
- FA master user's subscriptions apply across all sub-accounts they trade
- Each user login needs its own subscriptions and market data lines
- Run `ibkr_check_session` / subscription check once at bot startup

---

## Integration Points

- **`spx-contract-resolution`** — SPX-specific contract specs and 0DTE discovery
- **`ibkr-gateway-docker`** — Docker TWS/Gateway setup
- **`ibkr-api-edge-cases`** — API safety patterns (order IDs, reconnection, pacing)
- **`market-data-pipeline`** — Handling stale prices and sentinel values
- **`broker-api-integration`** — General broker API patterns
- **`options-trading-safety`** — Options-specific data requirements for greeks
