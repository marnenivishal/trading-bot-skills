---
name: ibkr-contract-validation
description: Use when IBKR returns "No security definition found", contract description mismatches, empty options chains, or invalid futures contracts — validates contract fields BEFORE debugging order or data logic
---

# IBKR Contract Validation Screener

## Why This Skill Exists

"No security definition has been found for this request" is one of the most common
IBKR API errors. It almost always means the **contract object fields are wrong or
incomplete** — not that the symbol doesn't exist or the API is broken. This skill
diagnoses contract issues AFTER ruling out permissions/data problems and BEFORE
touching order or data logic code.

---

## When to Activate

Trigger on ANY of:

- "No security definition has been found for this request"
- "Contract description mismatch"
- "API call works for SPY but not for [other symbol]"
- "Options chain empty for symbol, but stock is tradable"
- "Futures contract invalid / expired / discontinued"
- "reqContractDetails returns nothing"
- "conId is 0 or missing"
- Error code 200 ("No security definition has been found")

**First assumption: Contract fields are wrong (symbol, secType, exchange, currency, expiry, strike, right).**

---

## IBKR Contract Model

Every IBKR API request requires a `Contract` object with the correct combination of fields.
If ANY field is wrong, IBKR returns "No security definition found."

### Required Fields by Security Type

#### Stocks / ETFs (secType = "STK")

| Field | Required | Example | Notes |
|-------|----------|---------|-------|
| symbol | Yes | "AAPL" | Must match IBKR's symbol exactly |
| secType | Yes | "STK" | |
| currency | Yes | "USD" | |
| exchange | Yes | "SMART" | SMART routes to best exchange; or use specific like "ARCA", "ISLAND" |
| primaryExchange | Recommended | "NASDAQ" | Disambiguates when symbol exists on multiple exchanges |

#### Options (secType = "OPT")

| Field | Required | Example | Notes |
|-------|----------|---------|-------|
| symbol | Yes | "AAPL" | Underlying symbol |
| secType | Yes | "OPT" | |
| lastTradeDateOrContractMonth | Yes | "20260410" | Format: YYYYMMDD (NOT "expiry") |
| strike | Yes | 175.0 | Must be exact strike price |
| right | Yes | "C" or "P" | Call or Put |
| exchange | Yes | "SMART" | |
| currency | Yes | "USD" | |
| multiplier | Recommended | "100" | Usually 100 for equity options |

**Common option mistakes:**
- Using `expiry` instead of `lastTradeDateOrContractMonth` (ibapi uses the latter)
- Wrong strike price (must match IBKR's exact strikes, not rounded)
- Wrong date format (must be YYYYMMDD, not YYYY-MM-DD or MM/DD/YYYY)
- Missing multiplier field

#### Index Options (secType = "OPT", e.g., SPX)

| Field | Required | Example | Notes |
|-------|----------|---------|-------|
| symbol | Yes | "SPX" | NOT "$SPX" or "^SPX" |
| secType | Yes | "OPT" | |
| lastTradeDateOrContractMonth | Yes | "20260410" | |
| strike | Yes | 5200.0 | SPX uses 5-point strike increments |
| right | Yes | "C" or "P" | |
| exchange | Yes | "SMART" or "CBOE" | |
| currency | Yes | "USD" | |
| multiplier | Yes | "100" | |

**SPX-specific notes:**
- SPX options are European-style, cash-settled
- 0DTE SPX options expire at market close (PM settlement)
- SPXW (weekly) options use symbol "SPX" with the correct expiry date
- Strike increments: $5 for near-the-money, wider for far OTM

#### Futures (secType = "FUT")

| Field | Required | Example | Notes |
|-------|----------|---------|-------|
| symbol | Yes | "ES" | IBKR's symbol (not Bloomberg/Reuters) |
| secType | Yes | "FUT" | |
| lastTradeDateOrContractMonth | Yes | "202406" or "20240621" | YYYYMM or YYYYMMDD |
| exchange | Yes | "CME" | Must match actual exchange |
| currency | Yes | "USD" | |
| multiplier | Recommended | "50" | ES = $50 per point |

**Common futures mistakes:**
- Using expired contract month
- Wrong exchange (ES is CME, NQ is CME, CL is NYMEX)
- Using continuous contract symbol (no such thing in ibapi)

---

## Diagnostic Checklist

### Step 1: Reproduce in TWS

Ask the user:
> "Can you find this exact instrument in TWS using the symbol search?
> What fields does TWS show for it?"

If they can't find it in TWS with the same fields, the API contract is wrong.

TWS symbol search: Click the contract field in TWS, type the symbol, and check:
- The exact symbol IBKR uses
- Which exchange it lists
- The available expiry dates and strikes (for options/futures)

### Step 2: Compare Fields One by One

For the failing contract, verify each field:

```
Contract being sent:
  symbol              = ??? (check spelling, case)
  secType             = ??? (STK, OPT, FUT, FOP, IND, CASH)
  exchange            = ??? (SMART, CBOE, CME, ARCA, etc.)
  primaryExchange     = ??? (needed for ambiguous symbols)
  currency            = ??? (USD, EUR, GBP, etc.)
  lastTradeDateOrContractMonth = ??? (YYYYMMDD for options, YYYYMM for futures)
  strike              = ??? (exact decimal value)
  right               = ??? (C or P, not CALL or PUT)
  multiplier          = ??? (100 for equity options)
```

Common field mismatches:

| What's Wrong | Symptom | Fix |
|-------------|---------|-----|
| `expiry` instead of `lastTradeDateOrContractMonth` | "No security definition" | Use `contract.lastTradeDateOrContractMonth` |
| Symbol case or spelling | "No security definition" | Check TWS for exact symbol |
| Wrong exchange | "No security definition" or wrong instrument | Use SMART or check TWS |
| Missing primaryExchange | Ambiguous match or wrong instrument | Add primaryExchange for stocks |
| Expired option/future date | "No security definition" | Use a valid future expiry |
| Strike doesn't exist | "No security definition" | Check available strikes in TWS |
| `right = "CALL"` instead of `"C"` | "No security definition" | Use single letter: "C" or "P" |
| Missing currency | "No security definition" | Always set currency |

### Step 3: Use reqContractDetails to Validate

Before submitting orders, validate the contract programmatically:

```python
from ibapi.contract import Contract

def build_option_contract(symbol, expiry, strike, right):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = "OPT"
    contract.lastTradeDateOrContractMonth = expiry  # YYYYMMDD
    contract.strike = strike
    contract.right = right  # "C" or "P"
    contract.exchange = "SMART"
    contract.currency = "USD"
    contract.multiplier = "100"
    return contract

# Validate before trading
app.reqContractDetails(req_id, contract)
# In contractDetails callback: if no results, contract is invalid
```

If `reqContractDetails` returns zero results: the contract definition is wrong.
If it returns results: capture the `conId` and use it for subsequent requests (most reliable).

### Step 4: Use conId for Maximum Reliability

Once you have a valid `conId` from `reqContractDetails`, you can use it directly:

```python
contract = Contract()
contract.conId = 123456789  # From reqContractDetails
contract.exchange = "SMART"
# All other fields are resolved by IBKR from the conId
```

This eliminates symbol/expiry/strike/right mismatches entirely.

---

## Symbol Differences Between Vendors

IBKR symbols may differ from other data sources:

| Instrument | IBKR Symbol | Other Vendors |
|-----------|-------------|---------------|
| S&P 500 Index | SPX | $SPX, ^SPX, .SPX |
| Berkshire B | BRK B | BRK.B, BRK/B |
| E-mini S&P | ES | ES=F (Yahoo) |
| Euro FX | EUR | EURUSD (forex) |
| VIX Index | VIX | ^VIX, $VIX |

Always verify the IBKR symbol via TWS or `reqMatchingSymbols()`.

---

## Resolution Order

When contract errors occur, fix in this order:

1. **Verify in TWS first** — if you can't find it in TWS, the API can't find it either
2. **Fix field by field** — check each Contract field against TWS's description
3. **Use reqContractDetails** — programmatic validation before order submission
4. **Cache conIds** — once validated, use conId for reliability
5. **Only then** debug order logic or data subscription code

---

## Common Misdiagnoses This Skill Prevents

| Symptom | Wrong Diagnosis | Actual Cause |
|---------|----------------|--------------|
| "No security definition found" | "IBKR permissions issue" | Wrong symbol/exchange/expiry/strike combination |
| Options chain returns empty | "Market data subscription needed" | Wrong expiry format or symbol |
| reqHistoricalData fails for one ticker | "Pacing violation" | Contract fields don't match IBKR's listing |
| Order rejected immediately | "Order parameter error" | Contract object is invalid |
| Works for SPY but not SPX | "SPX needs special permissions" | SPX contract fields differ from SPY (index vs ETF) |

---

## Integration with Other Skills

- **ibkr-api-precautions**: Check precautions first (dialog blocking), then contract validation
- **ibkr-pacing-limits**: If contract is valid but data fails intermittently, check pacing
- **0dte-risk-management**: SPX 0DTE contracts need exact expiry date (today's YYYYMMDD)
- **options-trading-safety**: After contract is valid, check Greeks and DTE handling
