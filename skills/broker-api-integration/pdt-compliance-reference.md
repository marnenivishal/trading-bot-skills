# PDT Compliance Reference

Detailed reference for Pattern Day Trader rules, settlement timelines, and account type considerations for automated trading bots.

---

## T+1 Settlement Timeline

Equities in the US settle on a T+1 basis (trade date + 1 business day).

```
Day 0 (Trade Day)     Day 1 (Settlement Day)     Day 2+
─────────────────     ──────────────────────     ──────────
  You buy 100 AAPL      Trade settles.             Funds fully
  at $150.              Shares are yours.          available for
  Cash is committed     Cash is debited            withdrawal.
  but not yet           from your account.
  debited.

  You sell 100 AAPL     Trade settles.             Sale proceeds
  at $155.              Proceeds credited.         available as
  Shares committed      For cash accounts:         settled cash.
  but not yet           funds now settled
  transferred.          and reusable.
```

### Key Points

- **Margin accounts**: buying power is available immediately (broker extends credit).
- **Cash accounts**: you must wait for settlement before reusing proceeds. Using unsettled funds to buy = free-riding violation.
- **Options**: also T+1 settlement.
- **Futures**: T+0 settlement (same day). This is one reason futures are popular for active traders.

---

## Account Type Decision Matrix

| Feature | Cash Account | Margin Account | Portfolio Margin |
|---|---|---|---|
| **PDT Rule Applies** | No | Yes (under $25k) | Yes (under $25k) |
| **Buying Power** | Settled cash only | Up to 4x intraday (2x overnight) | Risk-based, often 6x+ |
| **Settlement Wait** | T+1 for equities | No wait (margin credit) | No wait (margin credit) |
| **Short Selling** | Not allowed | Allowed | Allowed |
| **Minimum Equity** | None (broker-specific) | $2,000 FINRA minimum | $110,000+ typically |
| **Interest Charges** | None | Margin interest on borrowed funds | Margin interest on borrowed funds |
| **Free-Riding Risk** | Yes, if using unsettled funds | No (margin covers settlement gap) | No |
| **Best For** | Swing trading, PDT avoidance | Active day trading with $25k+ | Large accounts, complex strategies |

### Pros and Cons Summary

**Cash Account**
- Pros: no PDT restrictions, no margin interest, no margin calls, simpler tax reporting
- Cons: limited to settled cash, no short selling, T+1 wait limits trading frequency

**Margin Account**
- Pros: immediate buying power, short selling, leverage up to 4x intraday
- Cons: PDT rule if under $25k, margin interest, margin calls possible, more complex risk

**Portfolio Margin**
- Pros: highest leverage, risk-based margining favors hedged positions
- Cons: high minimum equity, PDT still applies, complex margin calculations, not available at all brokers

---

## Day Trade Counting Rules

### Definition

A **day trade** is a round-trip in the same security on the same trading day:
- Buy 100 AAPL at 9:35 AM, sell 100 AAPL at 2:15 PM = **1 day trade**
- The buy and sell (or sell short and buy to cover) must occur on the same calendar trading day

### Counting Examples

| Action | Day Trade Count | Explanation |
|---|---|---|
| Buy 100 AAPL, sell 100 AAPL same day | 1 | One round-trip |
| Buy 100 AAPL, sell 50 AAPL same day, sell 50 next day | 1 | Only the 50 closed same day counts (partial) |
| Buy 100 AAPL AM, sell 100 AAPL AM, buy 100 AAPL PM, sell 100 AAPL PM | 2 | Two separate round-trips |
| Buy 100 AAPL Monday, sell 100 AAPL Tuesday | 0 | Not same day -- this is a swing trade |
| Sell short 100 TSLA, buy to cover 100 TSLA same day | 1 | Short round-trip counts too |

### Partial Fills

If you buy 100 shares in a single order but sell in two separate orders on the same day (e.g., sell 60 then sell 40), this counts as **1 day trade** (one round-trip, regardless of how many orders close it).

If you buy in two separate orders (buy 60, then buy 40) and sell all 100 in one order same day, this counts as **2 day trades** (two opening transactions, each paired with the closing sale).

### What Does NOT Count

- Buying today, selling tomorrow (swing trade)
- Buying and holding for weeks (position trade)
- Options assignments or exercises (not voluntary round-trips)
- Futures trades (CFTC-regulated, not FINRA)

---

## Rolling 5-Day Window

The PDT rule uses a **rolling 5 business day** window, not a calendar week.

### Example

```
Monday    - 1 day trade (AAPL round-trip)
Tuesday   - 0 day trades
Wednesday - 1 day trade (TSLA round-trip)
Thursday  - 1 day trade (SPY round-trip)
Friday    - ??? <-- Can you day trade today?

Answer: You have 3 day trades in the last 5 business days.
        If your account is margin with equity < $25k,
        you CANNOT make another day trade today without
        triggering PDT designation.

Next Monday - Monday's day trade drops off the window.
              You now have 2 day trades in the rolling window.
              You can make 1 more day trade.
```

### Implementation Considerations

- Count business days, not calendar days. Weekends and market holidays do not count.
- The window looks back from today, inclusive of today.
- If the account is flagged as PDT, the restriction lasts 90 calendar days unless equity is restored above $25k.
- Some brokers (e.g., Alpaca) expose the day trade count via API. Use it as a secondary check, but always maintain your own count as the primary gate.

---

## Futures and Index Options Exemptions

The following instruments are regulated by the CFTC, not FINRA, and are therefore **exempt from PDT rules**:

### Futures Contracts (PDT-Exempt)

| Symbol | Description | Exchange |
|---|---|---|
| /ES | E-mini S&P 500 | CME |
| /NQ | E-mini Nasdaq 100 | CME |
| /YM | E-mini Dow | CBOT |
| /RTY | E-mini Russell 2000 | CME |
| /CL | Crude Oil | NYMEX |
| /GC | Gold | COMEX |
| /MES | Micro E-mini S&P 500 | CME |
| /MNQ | Micro E-mini Nasdaq 100 | CME |

### Index Options (PDT-Exempt)

| Symbol | Description | Style |
|---|---|---|
| SPX | S&P 500 Index Options | European (cash-settled) |
| NDX | Nasdaq 100 Index Options | European (cash-settled) |
| XSP | Mini-SPX Options | European (cash-settled) |
| RUT | Russell 2000 Index Options | European (cash-settled) |
| VIX | CBOE Volatility Index Options | European (cash-settled) |

### What IS Subject to PDT

- Individual stock trades (AAPL, TSLA, etc.)
- ETF trades (SPY, QQQ, IWM, etc.) -- even though SPY tracks the S&P 500, it is an equity product
- Single-stock options (AAPL calls/puts, etc.)
- ETF options (SPY options, QQQ options, etc.)

### Strategy Implication

Active traders with accounts under $25k can use futures micro contracts (/MES, /MNQ) for intraday strategies without PDT restrictions. The micro contracts have lower margin requirements ($1,000-$2,000 per contract) and provide equivalent market exposure on a smaller scale.
