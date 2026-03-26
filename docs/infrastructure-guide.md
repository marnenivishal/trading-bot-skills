# Infrastructure Guide for Trading Bots

---

## Why Infrastructure Matters

A trading bot is only as good as its execution environment. The fastest strategy in the world is useless if your order arrives 500ms late.

| Environment | Typical Latency to Broker | Reliability | Cost |
|---|---|---|---|
| Home PC (Wi-Fi) | 50-300ms | Low (ISP outages, power loss, Windows updates) | $0/mo |
| Home PC (Ethernet) | 20-100ms | Medium (still ISP-dependent) | $0/mo |
| Shared VPS (US-East) | 5-20ms | Medium-High (shared CPU contention) | $20-50/mo |
| Dedicated VPS (US-East) | 1-10ms | High (dedicated resources) | $50-200/mo |
| Co-location (NY4/NY5) | <1ms | Very High (enterprise-grade) | $500+/mo |

During high-volatility events (market open, FOMC announcements, earnings releases), latency determines fill quality. A 200ms delay at market open can mean the difference between getting filled at your target price and getting filled 0.5% worse on a fast-moving stock.

---

## VPS Requirements for Trading Bots

### Minimum Requirements

| Resource | Minimum | Recommended | Why |
|---|---|---|---|
| CPU | 2 dedicated cores | 4+ dedicated cores | Strategy computation, data processing, concurrent order management |
| RAM | 8 GB | 16 GB+ | Market data buffers, order book depth, historical data caching |
| Storage | 50 GB SSD | 100 GB+ NVMe SSD | Database I/O for trade logging, fast reads for historical data |
| Network | 100 Mbps | 1 Gbps | Market data streams, especially with multiple symbols or Level 2 data |
| Location | US-East | US-East (Virginia/New Jersey) | Proximity to Alpaca (Virginia), IBKR (Connecticut), exchanges (New Jersey) |
| Uptime SLA | 99.9% | 99.99% | 99.9% = ~8.7 hours downtime/year; 99.99% = ~52 minutes/year |

### Critical: Dedicated vs Shared CPU

**Shared CPU** means your VPS runs on a physical server alongside other tenants. The hypervisor allocates CPU time on demand. This works fine at 2 AM, but at 9:30 AM when everyone's bot is processing market open data, you compete for CPU cycles.

**Dedicated CPU** means your cores are reserved exclusively for your VPS. No contention. Your bot gets the same performance at market open as it does at midnight.

For trading bots, **always choose dedicated CPU**. The $20-30/month premium over shared CPU pays for itself in consistent execution timing.

---

## Co-location

### What It Is

Co-location means placing your server in the same data center as the exchange or broker's matching engine. This minimizes network hops and reduces latency to microseconds.

### Major Data Centers

| Data Center | Location | Primary Use |
|---|---|---|
| NY4 / NY5 (Equinix) | Secaucus, NJ | US equities (NYSE, Nasdaq) |
| CME Aurora | Aurora, IL | Futures (CME, CBOT, NYMEX) |
| TY3 (Equinix) | Tokyo | Japanese equities (TSE) |
| LD4 (Equinix) | London | European equities (LSE) |

### Do You Need Co-location?

**Almost certainly not**, if you are:
- Trading on timeframes of minutes or longer
- Using retail brokers (Alpaca, Tradier, Schwab)
- Running strategies based on technical indicators, not order book microstructure
- Not competing with HFT firms for queue priority

Co-location is relevant for:
- Market making strategies where queue position matters
- Statistical arbitrage with sub-second holding periods
- HFT strategies competing on latency

**For day trading bots**: a dedicated VPS in the same AWS region (us-east-1) or data center region as your broker is sufficient. You are competing with other retail traders, not HFT firms.

---

## For Day Trading Bots: Practical Setup

### Recommended Architecture

```
┌─────────────────────────────────────────────────┐
│  Primary VPS (US-East, Dedicated CPU)           │
│                                                 │
│  ┌─────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Trading │  │ Database │  │  Monitoring   │  │
│  │   Bot   │──│ (SQLite/ │  │  (Prometheus/ │  │
│  │         │  │  Postgres)│  │   Grafana)    │  │
│  └─────────┘  └──────────┘  └───────────────┘  │
│                                                 │
│  Network: 1Gbps to broker API                   │
└─────────────────────────────────────────────────┘
         │
         │ Failover (DNS or health-check based)
         ▼
┌─────────────────────────────────────────────────┐
│  Backup VPS (Different Data Center / Region)    │
│                                                 │
│  Same configuration, standby mode               │
│  Activates if primary health check fails        │
└─────────────────────────────────────────────────┘
```

### VPS Providers Commonly Used for Trading

| Provider | Dedicated CPU Plans | US-East Locations | Notes |
|---|---|---|---|
| Hetzner | Yes | Ashburn, VA | Good price-to-performance ratio |
| DigitalOcean | Yes (Premium CPU) | NYC1, NYC3 | Easy setup, good API |
| Vultr | Yes (Bare Metal available) | New Jersey | Low-latency options |
| AWS EC2 | Yes (c5/c6i instances) | us-east-1 (Virginia) | Enterprise-grade, Alpaca runs on AWS |
| Linode (Akamai) | Yes (Dedicated CPU) | Newark, NJ | Consistent performance |

---

## Redundancy and Failover

### Why Redundancy Matters

A single VPS means a single point of failure. If the data center has a network issue, your bot goes dark. During a volatile market, "dark" means positions without management.

### Redundancy Strategy

1. **Backup VPS in a different data center**: same configuration, database replica, ready to activate
2. **Health check monitoring**: external service (UptimeRobot, Pingdom, or custom) pings primary every 30 seconds
3. **Failover DNS**: if primary fails health check, DNS points to backup VPS (or use a load balancer)
4. **Auto-restart with systemd**: if the bot process crashes, systemd restarts it within seconds

```ini
# /etc/systemd/system/trading-bot.service
[Unit]
Description=Trading Bot
After=network.target

[Service]
Type=simple
User=trading
WorkingDirectory=/opt/trading-bot
ExecStart=/opt/trading-bot/venv/bin/python -m trading_bot
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

# Resource limits
MemoryMax=8G
CPUQuota=300%

[Install]
WantedBy=multi-user.target
```

5. **Database replication**: if using PostgreSQL, set up streaming replication to the backup VPS. If using SQLite, schedule periodic copies (every 5 minutes) to the backup.

### What to Monitor

| Metric | Alert Threshold | Action |
|---|---|---|
| Bot process alive | Process not running for 30s | Auto-restart via systemd, alert via SMS/Slack |
| WebSocket connected | Disconnected for 60s | Bot should auto-reconnect; alert if reconnect fails 3x |
| CPU usage | >80% sustained for 5 min | Investigate; may need larger instance |
| Memory usage | >85% of available | Investigate memory leak; restart if necessary |
| Disk usage | >80% | Clean old logs/data; expand disk |
| Order latency | >500ms average | Network issue or broker degradation; investigate |
| Last heartbeat to broker | >120s since last | Connection likely dead; force reconnect |

---

## CPU Contention: The Hidden Risk

### The Problem

At 9:30 AM Eastern, market opens. Every trading bot on every shared VPS in every data center wakes up simultaneously. CPU usage spikes across the physical server. Your "4 vCPU" shared VPS is suddenly competing with 20 other tenants for the same 64 physical cores.

Result: your strategy computation that takes 5ms at 2 AM takes 50ms at 9:30 AM. Your order that should arrive at the broker in 10ms takes 100ms. On a fast-moving stock at open, this is the difference between a fill and a miss.

### The Solution

Dedicated CPU plans guarantee your cores are not shared. The performance at 9:30 AM is the same as 2 AM. For a trading bot, this consistency is worth the premium.

---

## Cost Comparison

| Setup | Monthly Cost | Latency | Reliability | Best For |
|---|---|---|---|---|
| Home PC | $0 (electricity only) | 50-300ms | Low | Development, paper trading |
| Shared VPS | $20-50/mo | 5-20ms (variable) | Medium | Low-frequency strategies, testing |
| Dedicated VPS | $50-200/mo | 1-10ms (consistent) | High | Day trading bots, production |
| Co-location | $500-5,000+/mo | <1ms | Very High | HFT, market making |

### Cost-Benefit Analysis

For a bot trading a $50,000 account:
- A dedicated VPS at $100/month costs $1,200/year
- If better execution saves just 0.05% per trade (half a basis point) across 1,000 trades on $500 average size, that is $250 in saved slippage
- The reliability benefit (avoiding one bad outage that costs a few hundred dollars in unmanaged positions) likely covers the rest

For accounts over $25,000 trading daily, a dedicated VPS is a baseline requirement, not a luxury.
