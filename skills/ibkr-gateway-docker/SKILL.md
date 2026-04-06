---
name: ibkr-gateway-docker
description: Use when running IBKR (Interactive Brokers) via Docker with ib_insync, troubleshooting API connections, configuring market data subscriptions, or handling TWS/Gateway port forwarding issues
---

# IBKR Gateway Docker Integration

## The Iron Law

**IBKR API connections require: correct port mapping, API enabled in TWS GUI, market data subscriptions active, and unique client IDs per bot.** Missing any one of these causes silent connection failures with misleading error messages.

## Why This Skill Exists

The `ghcr.io/extrange/ibkr` Docker image and IBKR's API have multiple non-obvious configuration requirements that cause hours of debugging. These are real issues encountered in production:

- API port not listening despite gateway being logged in
- Port forwarding mismatch (socat forwards to wrong internal port)
- Market data returning NaN despite valid subscriptions
- "Competing live session" errors from multiple client IDs
- Paper trading accounts blocking API access (IBKR Lite)
- Disclaimer popups blocking API connections silently

---

## Docker Image: ghcr.io/extrange/ibkr

### Critical: Port Mapping

The image uses **socat** to forward the internal API port to **port 8888**. The internal port depends on mode:

| Mode | Application | Internal Port | Socat Forwards To |
|------|------------|---------------|-------------------|
| TWS (default) | Trader Workstation | 7497 (paper) / 7496 (live) | **8888** |
| Gateway | IB Gateway | 4002 (paper) / 4001 (live) | **8888** |

**Your docker-compose must map to 8888, NOT 4002:**

```yaml
# WRONG — will get ConnectionRefused
ports:
  - "4002:4002"

# RIGHT — socat forwards internal port to 8888
ports:
  - "4002:8888"    # external:internal(socat)
```

**Your bot must connect to port 8888 inside Docker network:**

```yaml
environment:
  IBKR_HOST: ibgateway
  IBKR_PORT: 8888    # NOT 4002
```

### GATEWAY_OR_TWS Environment Variable

Controls whether TWS or IB Gateway runs. If not set, defaults to `tws`.

```yaml
environment:
  GATEWAY_OR_TWS: gateway  # or "tws" (default)
```

**Gotcha:** IB Gateway mode may fail with "server error" during authentication on some accounts. TWS mode is more reliable but requires the API to be manually enabled via GUI.

### IBC Configuration via Environment Variables

IBC (IB Controller) settings can be set via environment variables with `IBC_` prefix:

```yaml
environment:
  IBC_TradingMode: paper
  IBC_ReadOnlyApi: "no"
  IBC_AcceptIncomingConnectionAction: accept  # CRITICAL: auto-accept API connections
  IBC_ExistingSessionDetectedAction: primary
  IBC_TwofaTimeoutAction: restart
```

**`IBC_AcceptIncomingConnectionAction: accept`** is critical. Without it, every API connection attempt waits for manual approval in the GUI, causing TimeoutError.

### API Must Be Enabled Manually (One-Time)

Even with IBC configured, the "Enable ActiveX and Socket Clients" checkbox in TWS must be checked manually via the noVNC web interface:

1. Open `http://localhost:6080` (noVNC)
2. In TWS: Edit > Global Configuration > API > Settings
3. Check "Enable ActiveX and Socket Clients"
4. Verify Socket port is 7497 (paper) or 7496 (live)
5. Click OK

This persists in the Docker volume across restarts. Only needs to be done once.

**Gotcha:** On IBKR Lite (commission-free) accounts, the API settings page shows "API support is not available for accounts that support commission free trading." You must upgrade to **IBKR Pro** first.

---

## Common Errors and Fixes

### Error: ConnectionRefusedError on port 4002

**Cause:** Bot connecting to internal port 4002, but socat forwards from 8888.
**Fix:** Change bot's IBKR_PORT to 8888 (inside Docker network).

### Error: TimeoutError (TCP connects but handshake fails)

**Cause:** API not enabled in TWS, or `AcceptIncomingConnectionAction` set to `manual`.
**Fix:** Enable API via noVNC GUI and set `IBC_AcceptIncomingConnectionAction: accept`.

### Error 10168: "Requested market data is not subscribed"

**Cause:** Missing market data subscription for the requested instrument.
**Fix:** Subscribe at ibkr.com > Settings > Market Data Subscriptions:
- US stocks/options: "US Real-Time Non Consolidated Streaming Quotes (IBKR-PRO)"
- SPX index: "Cboe One (NP,L1)" — $1/mo
- Futures: "CME Event Contracts"

**Gotcha:** New subscriptions may not activate until the next business day.

### Error 10089: "Requested market data requires additional subscription for API"

**Same as 10168** but specific to API access. Ensure "Market Data API Acknowledgement" is signed at ibkr.com.

### Error 10197: "No market data during competing live session"

**Cause:** Multiple client IDs or TWS sessions consuming the same data feed. Paper accounts have limited concurrent data streams.
**Fix:** 
- Use unique client IDs per bot (e.g., SPX=1, AlphaRunner=2, test scripts=99)
- Stop other bots before running test scripts
- During market closed hours, paper accounts may not serve any streaming data

### Error 10141: "Paper trading disclaimer must first be accepted"

**Cause:** After IBKR gateway restart, a disclaimer popup blocks the API.
**Fix:** Open noVNC (`http://localhost:6080`) and click Accept on the disclaimer dialog.

### Error: "server error, will retry in 4 seconds" (IB Gateway mode)

**Cause:** IB Gateway authentication failing. Common when switching from TWS to Gateway mode.
**Fix:** Switch back to TWS mode (`GATEWAY_OR_TWS: tws` or remove the env var). TWS mode is more reliable for paper accounts.

---

## ib_insync Patterns

### Client ID Management

Each bot MUST use a unique client ID. IBKR disconnects existing connections when a new client with the same ID connects.

```python
# SPX Trader
ib.connect("ibgateway", 8888, clientId=1)

# AlphaRunner  
ib.connect("ibgateway", 8888, clientId=2)

# Test scripts (use high numbers)
ib.connect("ibgateway", 8888, clientId=99)
```

### Snapshot vs Streaming Data

```python
# Snapshot (one-time, good for tests)
ticker = ib.reqMktData(contract, snapshot=True)
ib.sleep(3)  # Must wait for data to arrive

# Streaming (continuous updates)
ticker = ib.reqMktData(contract, snapshot=False)
ib.pendingTickersEvent += on_ticker_update
```

**Gotcha:** Snapshot requests still require market data subscription. They don't bypass subscription requirements.

### NaN Check for Prices

IBKR returns `float('nan')` for unavailable data, not `None`:

```python
# WRONG
if ticker.last is None:  # NaN is not None!

# RIGHT
if ticker.last != ticker.last:  # NaN != NaN is True
    print("No data available")

# ALSO RIGHT
import math
if math.isnan(ticker.last):
    print("No data available")
```

### asyncio + ib_insync Conflicts

`ib_insync` patches asyncio via `nest_asyncio`. This breaks newer versions of `uvicorn`:

```python
# WRONG — uvicorn.run() fails with loop_factory error
threading.Thread(target=uvicorn.run, args=(app,), kwargs={...}).start()

# RIGHT — use Config + Server with a fresh event loop
def _run():
    config = uvicorn.Config(app, host="0.0.0.0", port=port)
    server = uvicorn.Server(config)
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(server.serve())

threading.Thread(target=_run, daemon=True).start()
```

### qualifyContracts Inside asyncio.run()

```python
# WRONG — raises "This event loop is already running"
async def test():
    ib = IB()
    await ib.connectAsync(host, port, clientId)
    ib.qualifyContracts(contract)  # This calls run_until_complete internally

asyncio.run(test())

# RIGHT — use synchronous connect in sync context
ib = IB()
ib.connect(host, port, clientId)  # sync version
ib.qualifyContracts(contract)     # works fine
```

---

## Docker Compose Template

```yaml
ibgateway:
  image: ghcr.io/extrange/ibkr:latest
  container_name: spxtrader-ibgateway
  restart: unless-stopped
  ports:
    - "4002:8888"    # API: socat forwards internal port to 8888
    - "6080:6080"    # noVNC web UI
  environment:
    USERNAME: ${IBKR_USERNAME}
    PASSWORD: ${IBKR_PASSWORD}
    IBC_TradingMode: ${IBKR_TRADING_MODE:-paper}
    IBC_ReadOnlyApi: "no"
    IBC_AcceptIncomingConnectionAction: accept
    IBC_ExistingSessionDetectedAction: primary
    IBC_TwofaTimeoutAction: restart
  volumes:
    - ibgateway-data:/root/Jts
```

## Market Data Subscription Checklist

Before going live, ensure these are subscribed at ibkr.com:

- [ ] Account is IBKR Pro (not Lite/commission-free)
- [ ] "Market Data API Acknowledgement" signed
- [ ] "US Real-Time Non Consolidated Streaming Quotes" — US stocks + options
- [ ] "Cboe One (NP,L1)" — SPX/VIX index data ($1/mo)
- [ ] "CME Event Contracts" — futures (if needed)
- [ ] Market Data Subscriber Status: Non-Professional

## Persistence Volume

The `ibgateway-data` volume stores TWS settings including the API enabled checkbox. If you delete this volume, you must re-enable the API via noVNC.

```bash
# CAUTION: This resets API settings — you'll need to re-enable via noVNC
docker volume rm spx-trader_ibgateway-data
```
