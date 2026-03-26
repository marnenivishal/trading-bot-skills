# MCP Server Configuration Guide for Trading

---

## Table of Contents

1. [What is MCP?](#what-is-mcp)
2. [Public.com MCP Server Setup](#publiccom-mcp-server-setup)
3. [Alpaca MCP Server Setup](#alpaca-mcp-server-setup)
4. [Environment Variable Security](#environment-variable-security)
5. [Testing Your Setup](#testing-your-setup)
6. [Troubleshooting](#troubleshooting)

---

## What is MCP?

Model Context Protocol (MCP) standardizes communication between LLMs and
external tools. Instead of building custom API integrations for each broker,
Claude discovers and calls broker tools through MCP servers. Each MCP server
exposes a set of typed tools that Claude can invoke directly, handling
authentication, request formatting, and response parsing automatically. This
means you configure a server once and Claude can trade, fetch data, and manage
positions without manual API calls.

---

## Public.com MCP Server Setup

### Configuration

Add the following to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "public-invest": {
      "command": "npx",
      "args": ["-y", "@anthropic/public-mcp-server"],
      "env": {
        "PUBLIC_API_KEY": "your-api-key",
        "PUBLIC_ACCOUNT_ID": "your-account-id"
      }
    }
  }
}
```

**Config file locations:**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

### Available Tools

| Tool | Description | Example Parameters |
|------|-------------|--------------------|
| `get_option_chain` | Fetch full option chain for a symbol | `{"symbol": "AAPL", "expiration": "2026-04-17"}` |
| `get_option_greeks` | Get Greeks for a specific contract | `{"contract_id": "AAPL260417C00200000"}` |
| `preflight_multileg` | Validate a multi-leg order before submission | `{"legs": [{"symbol": "...", "side": "buy", "quantity": 1}]}` |
| `place_multileg_order` | Execute a multi-leg options order | `{"legs": [...], "order_type": "limit", "price": 2.50}` |
| `get_account_info` | Retrieve account balance and buying power | `{}` |
| `get_positions` | List all open positions with P&L | `{}` |

### Tool Usage Examples

**Fetching an option chain:**
```
"Get me the AAPL option chain expiring April 17, 2026"
```
Claude calls `get_option_chain` with `symbol: "AAPL"` and `expiration: "2026-04-17"`,
then formats the chain with strikes, bids, asks, and volume.

**Preflight check before trading:**
```
"Check if I can open a bull call spread on SPY: buy 580 call, sell 585 call, April expiry"
```
Claude calls `preflight_multileg` to validate margin requirements, buying power,
and order feasibility before any capital is committed.

**Placing an order:**
```
"Place that bull call spread for a limit of $2.00"
```
Claude calls `place_multileg_order` only after a successful preflight. Always
confirm the order details with the user before execution.

---

## Alpaca MCP Server Setup

### Configuration

```json
{
  "mcpServers": {
    "alpaca": {
      "command": "npx",
      "args": ["-y", "@anthropic/alpaca-mcp-server"],
      "env": {
        "ALPACA_API_KEY": "your-api-key",
        "ALPACA_SECRET_KEY": "your-secret-key",
        "ALPACA_BASE_URL": "https://paper-api.alpaca.markets"
      }
    }
  }
}
```

### Paper vs. Live Trading

Switch between paper and live trading by changing `ALPACA_BASE_URL`:

| Environment | Base URL |
|-------------|----------|
| Paper (testing) | `https://paper-api.alpaca.markets` |
| Live (real money) | `https://api.alpaca.markets` |

**Always start with paper trading.** Never point to the live URL until the
strategy has been validated in paper mode for at least 2 weeks.

### Available Tools

| Tool | Description |
|------|-------------|
| `get_account` | Account details, buying power, equity |
| `list_positions` | All open positions with current market value |
| `place_order` | Submit stock/ETF orders (market, limit, stop) |
| `cancel_order` | Cancel a pending order by ID |
| `get_bars` | Historical OHLCV data for backtesting |
| `list_orders` | View open, closed, or all orders |

### Alpaca-Specific Notes

- Alpaca supports fractional shares. You can specify `qty: 0.5` for half a
  share of a high-priced stock.
- Market orders are only executed during market hours (9:30 AM - 4:00 PM ET).
- Extended hours trading requires `extended_hours: true` on limit orders.
- Alpaca's paper trading environment mirrors live behavior but uses simulated
  fills.

---

## Environment Variable Security

### Never Hardcode API Keys

API keys should never appear in configuration files that are committed to
version control. Follow these practices:

**Option 1: Environment file (.env)**
```bash
# .env file (NEVER commit this)
PUBLIC_API_KEY=pk_live_abc123...
PUBLIC_ACCOUNT_ID=acct_xyz789...
ALPACA_API_KEY=AK1234567890
ALPACA_SECRET_KEY=SK0987654321
```

**Option 2: OS Keychain**
- macOS: Use Keychain Access to store keys, retrieve with `security` command.
- Windows: Use Credential Manager or `cmdkey`.
- Linux: Use `secret-tool` (libsecret) or `pass`.

**Option 3: Secret Manager**
- AWS Secrets Manager
- HashiCorp Vault
- 1Password CLI (`op read`)

### .gitignore Patterns

Add these to your `.gitignore` to prevent accidental exposure:

```gitignore
# API keys and secrets
.env
.env.*
*.env

# Claude Desktop config (contains API keys)
claude_desktop_config.json

# Credential files
credentials.json
*_credentials*
*.key
*.pem

# OS-specific
.DS_Store
Thumbs.db
```

### Key Rotation

- Rotate API keys every 90 days.
- Immediately revoke and regenerate keys if they are ever committed to a
  repository, even a private one.
- Use separate API keys for paper and live trading.

---

## Testing Your Setup

Follow these steps in order to verify your MCP server configuration.

### Step 1: Start Claude Desktop

Launch Claude Desktop with your updated `claude_desktop_config.json`. The MCP
servers will start automatically when Claude Desktop initializes.

Check the logs for server startup messages:
- macOS/Linux: `~/.claude/logs/`
- Windows: `%USERPROFILE%\.claude\logs\`

### Step 2: Verify Tool Discovery

Ask Claude:
```
"What MCP tools are available?"
```

Claude should list all tools exposed by the configured MCP servers. If no tools
appear, the server failed to start (see Troubleshooting below).

### Step 3: Test a Read-Only Tool First

Start with a non-destructive operation:
```
"Get the AAPL option chain for the nearest expiration"
```
or
```
"Show me my current account balance"
```

This confirms authentication and network connectivity without risking any
capital.

### Step 4: Verify Paper Trading

Before going live, execute a small test order in paper mode:
```
"Buy 1 share of SPY at market price using paper trading"
```

Verify:
- The order appears in your paper account.
- The fill price is reasonable.
- The position shows up in `get_positions` / `list_positions`.

### Step 5: Graduate to Live (When Ready)

Only switch to live trading after:
- [ ] Paper trading works correctly for 2+ weeks.
- [ ] Strategy has been backtested with realistic transaction costs.
- [ ] Risk limits are configured (position sizing, stop losses).
- [ ] Kill switch is tested and accessible.

---

## Troubleshooting

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Server not found | Wrong command path or package name | Verify `npx` and `node` are installed and on PATH. Run `npx -y @anthropic/public-mcp-server --version` manually. |
| Auth failure (401/403) | Invalid or expired API key | Regenerate the key on the broker's dashboard. Verify the env var name matches exactly. |
| Timeout | Network issue or server overload | Check internet connectivity. Retry after 30 seconds. Check broker status page. |
| Tool not recognized | MCP protocol version mismatch | Update the MCP server package: `npm update -g @anthropic/public-mcp-server`. |
| Connection refused | Server crashed on startup | Check logs for startup errors. Common cause: missing required env vars. |
| Rate limited (429) | Too many API calls | Add delays between requests. Check broker's rate limit docs. |
| Invalid parameters | Wrong argument types or format | Verify parameter format matches the tool schema. Dates should be ISO 8601. |

### Debugging Steps

1. **Check server process:** Verify the MCP server process is running.
   ```bash
   ps aux | grep mcp-server
   ```

2. **Check logs:** Look for error messages in Claude Desktop logs and the MCP
   server's stderr output.

3. **Test the server manually:** Run the MCP server command directly in a
   terminal to see startup errors:
   ```bash
   PUBLIC_API_KEY=your-key npx -y @anthropic/public-mcp-server
   ```

4. **Verify network:** Ensure your firewall and proxy settings allow outbound
   HTTPS connections to the broker's API endpoints.

5. **Version check:** Ensure Node.js >= 18 and npm >= 9.
   ```bash
   node --version
   npm --version
   ```

---

## Cross-References

- Trading strategy skills: see `skills/` directory
- Risk management and kill switch: see `skills/risk-management.md`
- Options trading workflow: see `skills/options-strategy.md`

---

*Keep your MCP server packages updated. Broker APIs evolve, and outdated
servers may lose access to new tools or encounter breaking changes.*
