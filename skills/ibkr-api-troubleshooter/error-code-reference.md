# IBKR Error Code Reference

Exhaustive lookup table for IBKR API error codes. For the diagnostic workflow, classification methodology, and code examples, see `SKILL.md`.

## How to Use This Table

1. Find your error code in the relevant category below
2. Read the **Resolution** column for immediate next steps
3. Follow any **→ skill** references for deeper guidance
4. If your error code isn't listed, check the TWS API docs: https://ibkrcampus.com/campus/ibkr-api-page/twsapi-doc/#error-handling

---

## Contract Errors

These errors mean IBKR cannot find or validate the contract you specified.

| Code | Message | Resolution |
|------|---------|------------|
| 200 | No security definition has been found | Wrong symbol, secType, exchange, tradingClass, or currency. Verify the contract exists in TWS first. Use `qualifyContracts()` to resolve ambiguity. |
| 301 | Max number of tickers has been reached | Too many simultaneous market data subscriptions. Cancel unused subscriptions with `cancelMktData()`. → `ibkr-session-watchdog` Layer 3 |
| 321 | Server error validating request | Malformed request parameters. Check all fields for typos, invalid types, or out-of-range values. |
| 322 | Duplicate ticker ID | You reused a `reqId` that's still active. Assign unique IDs per request. |
| 354 | Requested market data is not subscribed | No subscription for this specific data type (e.g., Level 2, options). → `ibkr-market-data-subscriptions` |

---

## Permission Errors

These errors mean your account lacks authorization for the requested action.

| Code | Message | Resolution |
|------|---------|------------|
| 201 (permission) | Order rejected — no trading permission | Enable trading permissions: Client Portal → Settings → Trading Permissions. Check product type AND region. May take 24h to propagate. |
| 201 (options level) | Order rejected — insufficient options level | Check options level (L1-L4): Client Portal → Settings → Trading Permissions → Options. Covered calls = L1, spreads = L3, naked = L4. |
| 201 (futures) | Order rejected — no futures trading permission | Enable futures trading and complete any required certifications (futures knowledge questionnaire). |
| 203 | Security is not available for trading | Contract exists but is not tradeable on your account type, region, or at this time. Check market hours and contract restrictions. |

---

## Market Data Errors

These errors mean your market data subscriptions or API data access is insufficient.

| Code | Message | Resolution |
|------|---------|------------|
| 10089 | Requested market data requires subscription for API | You have TWS-only data, not API-enabled data. Sign the "Market Data API Acknowledgement" in Client Portal and subscribe to the API-enabled bundle. → `ibkr-market-data-subscriptions` |
| 10168 | Requested market data is not subscribed | Missing data bundle for this asset class/region. Subscribe at Client Portal → Market Data Subscriptions. → `ibkr-market-data-subscriptions` |
| 10169 | Market data for delayed is not available | Delayed data not available for this instrument. Subscribe to real-time data. |
| 162 | Historical Market Data Service error — pacing violation | Too many historical data requests. Wait 10+ minutes, implement request caching and throttling. → `ibkr-api-edge-cases` (PacedHistoricalDataFetcher) |
| 366 | No historical data query found for ticker | Invalid or expired contract for historical data request. Verify contract spec and date range. |
| 2104 | Market data farm connection is OK | Informational — not an error. Farm connectivity restored. |
| 2106 | HMDS data farm connection is OK | Informational — not an error. Historical data farm restored. |
| 2108 | Market data farm connection is inactive | Data farm disconnected. Usually temporary — wait and reconnect. If persistent, check internet/firewall. |
| 2158 | Sec-def data farm connection is OK | Informational — contract definition service restored. |

---

## Session & Connection Errors

These errors mean your connection to TWS/Gateway is broken or contested.

| Code | Message | Resolution |
|------|---------|------------|
| 502 | Couldn't connect to TWS | TWS or IB Gateway is not running, or you're using the wrong host/port. Paper: 7497 (TWS) / 4002 (Gateway). Live: 7496 (TWS) / 4001 (Gateway). Verify "Enable ActiveX and Socket Clients" is checked. |
| 504 | Not connected | Socket connection dropped. Perform full reconnect with state resync (not just socket reconnect). → `ibkr-api-edge-cases` (reconnect_and_resync pattern) |
| 507 | Bad message length | Corrupt data on the socket. Disconnect cleanly and reconnect. If persistent, check TWS/API version compatibility. |
| 509 | Exception caught while reading socket | Network-level failure. Implement automatic reconnection with exponential backoff. |
| 1100 | Connectivity lost | TWS/Gateway lost connection to IBKR servers. Orders with `outsideRTH=False` will be cancelled. Wait for reconnection (1101/1102). |
| 1101 | Connectivity restored — data lost | Connection restored but some data may be stale. Resync all state: open orders, positions, account values. |
| 1102 | Connectivity restored — data maintained | Connection restored, data is current. Verify open orders are still active. |
| 10141 | Paper trading disclaimer | Accept the disclaimer popup in TWS/Gateway GUI or noVNC. → `ibkr-gateway-docker` |
| 10197 | No market data during competing session | Another login (Mobile, Client Portal, another TWS) is using the same username. → `ibkr-session-watchdog` Layer 2 |

---

## Risk & Precaution Errors

These errors mean the order violates risk limits or TWS safety settings.

| Code | Message | Resolution |
|------|---------|------------|
| 201 (margin) | Order rejected — insufficient margin | Not enough margin/buying power. Check available margin via `reqAccountSummary()`. Reduce order size or add funds. |
| 399 | Order message (warning passthrough) | Catch-all for TWS warnings forwarded to the API. **Read the full message text** — it contains the specific warning (price deviation, size warning, etc.). |
| 404 | Order held — precautionary settings | Order held by TWS API Precautions. Adjust in TWS: Configure → API → Precautions. Or reduce order size/value to fit within limits. |
| 434 | Order size/value exceeds max | Exceeds API Precaution "Maximum Order Size" or "Maximum Order Value" setting. Either increase the limit in TWS or reduce order size. |
| 110 | Price does not conform to min price variation | Order price increment is too small. US stocks: $0.01, options: $0.05 (or $0.01 for penny pilot). Round prices to valid increments. |
| 135 | Can't find order with ID | Order ID not found — likely already cancelled, filled, or from a different client session. Resync open orders with `reqAllOpenOrders()`. |
| 136 | Can't modify a filled order | Order already fully filled. Check `execDetails` for fill records before attempting modifications. |
| 161 | Cancel attempted when order is not in cancellable state | Order is in a terminal state (filled, cancelled, error). Verify order status before cancel attempts. |

---

## Pacing & Rate Limit Errors

These errors mean you're sending too many requests too fast.

| Code | Message | Resolution |
|------|---------|------------|
| 162 | Pacing violation — historical data | Max 60 requests per 10 min, 6 per contract per 2 sec, 15 sec cooldown for identical requests. Implement request queuing and caching. → `ibkr-api-edge-cases` |
| 100 | Max rate of messages exceeded | Sending API messages too fast. Implement a message queue with rate limiting (max ~50 messages/sec). |
| 326 | Unable to connect — client limit exceeded | Too many concurrent API connections (max ~32). Each bot/tool using a unique clientId counts as one connection. Close unused connections. |

---

## Web API / HTTP Errors

These errors come from the Client Portal Gateway REST API (Web API).

| HTTP Status | Meaning | Resolution |
|-------------|---------|------------|
| 401 Unauthorized | Session not authenticated or expired | Call `GET /iserver/auth/status` to check. If expired, call `POST /iserver/reauthenticate`. If gateway was restarted, re-authenticate via the gateway login page. |
| 403 Forbidden | Insufficient permissions for endpoint | Check trading permissions match the endpoint's requirements. Some endpoints require specific account features or product permissions. |
| 429 Too Many Requests | Rate limited | Web API has stricter rate limits than TWS API. Implement request throttling with exponential backoff. Batch requests where possible. |
| 500 Internal Server Error | Usually malformed request body | Check JSON structure, contract IDs, and parameter types. Verify `conId` is valid and not expired. Common cause: sending string where int expected. |
| 503 Service Unavailable | Gateway starting up or not connected | Wait for gateway initialization (can take 30-60 sec). Check `GET /iserver/auth/status` for connection state. Verify no competing brokerage sessions. |

---

## Informational Messages (Not Errors)

These codes appear in logs but are NOT errors. Do not try to "fix" them.

| Code | Message | What It Means |
|------|---------|--------------|
| 2104 | Market data farm connection is OK | Data farm connectivity confirmed — healthy status |
| 2106 | HMDS data farm connection is OK | Historical data service connected — healthy status |
| 2107 | HMDS data farm connection is inactive | Historical data service temporarily disconnected — usually auto-recovers |
| 2108 | Market data farm connection is inactive | Data farm temporarily disconnected — usually auto-recovers |
| 2158 | Sec-def data farm connection is OK | Contract definition service connected — healthy status |
| 2119 | Market data farm is connecting | Data farm reconnection in progress — wait |
