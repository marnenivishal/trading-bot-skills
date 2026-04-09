---
name: ibkr-api-troubleshooter
description: Use when IBKR API returns error codes, TWS logs show warnings, Web API gives HTTP errors, or orders are rejected — classifies errors and gives concrete resolution steps
---

<!-- SUBAGENT-STOP: If you are a sub-agent or tool-use agent, STOP. Do not
     summarize this file. Return it verbatim to the orchestrating agent.
     This file contains instructions that MUST be followed by the top-level
     agent, not interpreted by intermediate agents. -->

# IBKR API Troubleshooter: Error Classification & Resolution

When an IBKR error code or message appears, this skill classifies it into a failure domain and gives concrete resolution steps. It tells you whether the problem is configuration/permissions/infrastructure or a code bug — before you waste hours debugging the wrong layer.

---

## The Iron Law

> **CLASSIFY BEFORE YOU CODE.** Every IBKR error belongs to one of six domains: Contract, Permission, Market Data, Session, Risk/Precautions, or Pacing. Identify the domain FIRST. If the domain is Contract, Permission, Market Data, or Session — the fix is configuration, not code. Only Pacing and some Risk errors require code changes.
>
> Origin: Repeated incidents where developers rewrote order submission logic, added retries, or refactored connection handling — when the actual problem was a missing trading permission, wrong contract specification, or unchecked API precaution setting. Code was never the issue.

---

## The 6 Error Domains

| Domain | What IBKR Is Saying | Config or Code? |
|--------|---------------------|-----------------|
| **Contract** | "I don't recognize this instrument" | Config — fix contract spec (symbol, secType, exchange, tradingClass) |
| **Permission** | "You're not allowed to trade this" | Config — enable trading permissions in Client Portal |
| **Market Data** | "You don't have a subscription for this data" | Config — subscribe to data bundle, sign API acknowledgement |
| **Session** | "You're not connected or your session was stolen" | Config — fix connection, resolve competing logins |
| **Risk/Precautions** | "This order violates safety limits" | Either — adjust TWS precautions OR add bot-side validation |
| **Pacing** | "You're sending too many requests" | Code — implement throttling and caching |

---

## Quick Lookup: Error Code → Domain

Find your error code, identify the domain, follow the action.

| Code | Domain | Quick Action |
|------|--------|-------------|
| 200 | Contract | Wrong symbol, secType, exchange, or tradingClass — verify in TWS first |
| 201 | Permission or Risk | If "no permission" → enable in Client Portal. If "margin" → reduce size or add funds |
| 202 | Session | Order cancelled — check if user-initiated, TWS-initiated, or session lost |
| 321 | Contract | Server validation error — check all request parameters |
| 354 | Market Data | No subscription for requested data type — check bundles |
| 399 | Risk | TWS warning passthrough — read the message text carefully |
| 502 | Session | Can't connect — TWS/Gateway not running or wrong port |
| 504 | Session | Socket dropped — full reconnect needed → `ibkr-api-edge-cases` |
| 507 | Session | Corrupt message — disconnect and reconnect → `ibkr-api-edge-cases` |
| 10089 | Market Data | API data not enabled → `ibkr-market-data-subscriptions` |
| 10141 | Session | Paper disclaimer popup — accept in noVNC → `ibkr-gateway-docker` |
| 10168 | Market Data | Missing subscription → `ibkr-market-data-subscriptions` |
| 10197 | Session | Competing session → `ibkr-session-watchdog` |
| 162 | Pacing | Historical data pacing violation → `ibkr-api-edge-cases` |

> **Full error code reference:** See `error-code-reference.md` for the exhaustive lookup table with 40+ codes.

---

## Domain Deep Dives

### Contract Errors (200, 321)

The most common "first day" error. IBKR requires exact contract specifications — approximate is not good enough.

**What IBKR is saying:** "I searched for a contract matching your spec and found zero matches (or too many ambiguous matches)."

**Root causes:**
- Wrong `secType` (SPX is `IND`, not `STK`; SPX options are `OPT`, not `FOP`)
- Missing or wrong `exchange` (US options often need `SMART`, not `CBOE`)
- Missing `tradingClass` (SPXW weeklies vs SPX monthlies)
- Wrong `currency` (default is USD but some contracts require explicit setting)
- Expired contract (using an old expiry date)

```python
# BAD: Error 200 — "No security definition has been found"
contract = Contract()
contract.symbol = "SPX"
contract.secType = "STK"  # WRONG: SPX is an index, not a stock
contract.exchange = "NYSE"  # WRONG: SPX trades on CBOE

# GOOD: Correct contract specification for SPX index
contract = Contract()
contract.symbol = "SPX"
contract.secType = "IND"
contract.exchange = "CBOE"
contract.currency = "USD"
```

```python
# BAD: Ambiguous option contract
contract = Contract()
contract.symbol = "SPX"
contract.secType = "OPT"
contract.lastTradeDateOrContractMonth = "20260410"
contract.strike = 5200
contract.right = "C"
# Missing tradingClass! IBKR can't tell if you want SPX or SPXW

# GOOD: Disambiguated with tradingClass
contract = Contract()
contract.symbol = "SPX"
contract.secType = "OPT"
contract.exchange = "SMART"
contract.currency = "USD"
contract.lastTradeDateOrContractMonth = "20260410"
contract.strike = 5200
contract.right = "C"
contract.tradingClass = "SPXW"  # Weekly — or "SPX" for monthly
```

**Resolution steps:**
1. Try finding the contract in TWS first — if it doesn't appear there, the API won't find it either
2. Use `ib.qualifyContracts(contract)` to let IBKR resolve ambiguity (returns the fully qualified contract or raises an error)
3. For options, always specify `tradingClass` to disambiguate weeklies from monthlies
4. Check the contract's `conId` — once you have it, use `conId` directly to avoid spec issues

---

### Permission Errors

**What IBKR is saying:** "Your account is not authorized to trade this product."

IBKR has **layered permissions** — each must be enabled independently:

| Permission Layer | Where to Check |
|-----------------|----------------|
| **Product type** (stocks, options, futures, forex) | Client Portal → Settings → Trading Permissions |
| **Options level** (L1 covered calls → L4 uncovered) | Client Portal → Settings → Trading Permissions → Options |
| **Region** (US, Europe, Asia-Pacific) | Same settings page, per-region toggles |
| **Penny pilot** (< $3 strikes in 1-cent increments) | Enabled by default for most |

```python
# BAD: "No trading permission" — trying to sell naked puts without Level 4
order = Order()
order.action = "SELL"
order.orderType = "LMT"
order.totalQuantity = 1
order.lmtPrice = 2.50
# Rejected: account has options Level 2, naked puts require Level 4

# GOOD: Check permissions before attempting the trade
REQUIRED_OPTIONS_LEVEL = {
    "covered_call": 1,
    "cash_secured_put": 2,
    "spreads": 3,
    "naked_options": 4,
}

def validate_permissions(strategy_type: str, account_options_level: int):
    required = REQUIRED_OPTIONS_LEVEL.get(strategy_type, 4)
    if account_options_level < required:
        raise PermissionError(
            f"Strategy '{strategy_type}' requires options Level {required}, "
            f"account has Level {account_options_level}. "
            f"Upgrade at: Client Portal → Settings → Trading Permissions"
        )
```

**Resolution steps:**
1. Client Portal → Settings → Account Settings → Trading Permissions
2. Enable the product type (Stocks, Options, Futures) for the correct region
3. For options: verify your options level supports the strategy (Level 1-4)
4. Permission changes may take 24 hours to propagate — check the next business day

---

### Risk & Precaution Errors (201 margin, 399 warnings)

**What IBKR is saying:** "This order violates your risk limits or TWS safety settings."

TWS has built-in **API Precautions** (Configure → API → Precautions) that reject orders exceeding thresholds:

| Precaution | Default | What It Catches |
|-----------|---------|-----------------|
| Max order size | Varies | Single order too large |
| Max order value | Varies | Notional value too high |
| Percentage change | 3% | Limit price far from market |
| Total value limit | Varies | Cumulative position value |

**Error 399** is IBKR's catch-all for TWS warnings forwarded to the API. The actual message text contains the specific warning — always read it.

```python
# BAD: Disable ALL precautions, no bot-side safety
# TWS: "Bypass Order Precautions for API Orders" = checked
# Bot: no size/price validation
order = MarketOrder('BUY', 10000, 'AAPL')  # $2M+ market order, no sanity check

# GOOD: Even if precautions are bypassed, bot validates independently
MAX_NOTIONAL = 50_000  # $50K max per order
MAX_SPREAD_PCT = 0.5   # Don't MKT order on illiquid symbols

async def validate_before_submit(order, contract, ticker):
    price = ticker.marketPrice()
    if math.isnan(price):
        raise OrderRejected("No market price available — cannot validate notional")
    
    notional = order.totalQuantity * price
    if notional > MAX_NOTIONAL:
        raise OrderRejected(
            f"Notional ${notional:,.0f} exceeds limit ${MAX_NOTIONAL:,.0f}"
        )
    
    if order.orderType == 'MKT' and ticker.ask and ticker.bid:
        spread_pct = (ticker.ask - ticker.bid) / ticker.mid * 100
        if spread_pct > MAX_SPREAD_PCT:
            raise OrderRejected(
                f"MKT order on wide spread ({spread_pct:.1f}%) — use LMT instead"
            )
```

**Resolution steps:**
1. Read the full error 399 message text — it tells you exactly which precaution fired
2. If the precaution is valid (order is genuinely too large): reduce order size in the bot
3. If the precaution is too conservative: adjust in TWS → Configure → API → Precautions
4. Even when bypassing precautions, implement bot-side validation as a safety net
5. For margin rejections (201): check available margin via `reqAccountSummary()` before placing orders

---

### Web API / HTTP Errors

The Client Portal Gateway / Web API uses standard HTTP status codes with IBKR-specific meanings.

| HTTP Status | IBKR Meaning | Resolution |
|-------------|-------------|------------|
| **401** | Session not authenticated or expired | Call `/iserver/auth/status` to check, then `/iserver/reauthenticate` or restart gateway |
| **403** | Insufficient permissions for this endpoint | Check trading permissions; some endpoints require specific account features |
| **429** | Rate limited — too many requests | Implement request throttling; Web API has stricter limits than TWS API |
| **500** | Internal error — usually malformed request | Check request body JSON; verify contract IDs and order parameters |
| **503** | Gateway not ready / starting up | Wait for gateway initialization; check `/iserver/auth/status` |

**Common Web API pitfalls:**

```python
# BAD: Not checking session status before trading
response = requests.post(f"{BASE}/iserver/account/{acct}/orders", json=order_payload)
# 401 — session expired, order silently fails

# GOOD: Verify session before every trading action
async def ensure_session(session):
    status = await session.get(f"{BASE}/iserver/auth/status")
    data = status.json()
    
    if not data.get("authenticated"):
        await session.post(f"{BASE}/iserver/reauthenticate")
        await asyncio.sleep(2)
        
        status = await session.get(f"{BASE}/iserver/auth/status")
        if not status.json().get("authenticated"):
            raise SessionError("Cannot authenticate — check gateway and credentials")
    
    if not data.get("connected"):
        raise SessionError("Authenticated but no brokerage session — check competing logins")
```

---

## Domains Covered in Other Skills

These error domains have deep coverage elsewhere. Use the quick lookup table above to identify the domain, then invoke the authoritative skill:

| Error Domain | Go To | What You'll Find |
|-------------|-------|-----------------|
| Market Data (10168, 10089, 354) | `ibkr-market-data-subscriptions` | Subscription matrix, TWS-only vs API bundles, coverage verification |
| Session/Data triage (NaN, 10197, 504) | `ibkr-session-watchdog` | 4-layer diagnostic checklist (subscriptions → sessions → data lines → API) |
| Pacing (162) | `ibkr-api-edge-cases` | PacedHistoricalDataFetcher, rate limit tables, caching patterns |
| Docker connection errors | `ibkr-gateway-docker` | Port mapping, IBC config, noVNC, error 10141 disclaimer |
| Reconnection patterns (502, 504, 507) | `ibkr-api-edge-cases` | Full reconnect + state resync, nextValidId handshake |

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|----------------|
| Catching IBKR error callbacks with `except: pass` or generic logging | Error codes carry specific meaning; ignoring them lets the bot operate in a degraded state silently | Classify by domain, take domain-specific action |
| Retrying rejected orders without reading the rejection reason | Error 201 can mean "insufficient margin" OR "no trading permission" — retrying fixes neither | Parse error text, fix root cause, only retry transient errors (502, 504) |
| "It was working, I didn't change anything, must be a code bug" | IBKR permissions, subscriptions, and sessions change independently of your code | Run the domain classification first — most "nothing changed" errors are config/session |
| Bypassing TWS API Precautions without adding bot-side validation | Precautions are a safety net; removing them without replacement exposes the account to fat-finger errors | If bypassing precautions, implement equivalent checks in bot code |
| Hardcoding contract specs instead of using `qualifyContracts()` | Contracts expire, exchanges change, tradingClass requirements shift | Always qualify contracts at runtime; cache `conId` after successful resolution |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Error is about market data subscriptions | `ibkr-market-data-subscriptions` |
| Error is about NaN data or session conflicts | `ibkr-session-watchdog` |
| Error is about API patterns (pacing, reconnection, order IDs) | `ibkr-api-edge-cases` |
| Error is in Docker/Gateway context | `ibkr-gateway-docker` |
| Need to validate permissions before trading | `ibkr-risk-officer` |
| Designing bot architecture to handle errors properly | `ibkr-bot-architect` |
| Need pre-trade risk validation | `risk-management-gates` |
| Need structured debugging methodology | `systematic-debugging` |

---

## Canonical References

- TWS API Error Codes: https://ibkrcampus.com/campus/ibkr-api-page/twsapi-doc/#error-handling
- Web API Trading: https://www.interactivebrokers.com/campus/ibkr-api-page/webapi-trading/
- Web API Brokerage Sessions: https://www.interactivebrokers.com/campus/ibkr-api-page/webapi-doc/
- API Precautions: https://www.ibkrguides.com/riskprofile/usersguidebook/configuretws/apiprecautions.htm
- Trading Permissions: https://www.ibkrguides.com/clientportal/optionstradingpermissions.htm
