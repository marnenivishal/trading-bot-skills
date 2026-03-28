---
name: chat-signal-parsing-and-dedup
description: "Use when parsing trading signals from chat messages, implementing multi-layer deduplication, routing signals by priority tier, or normalizing usernames and ticker mentions from Discord, Telegram, or web chatrooms"
---

# Chat Signal Parsing and Deduplication

Trading chatrooms generate noisy, duplicated, ambiguous messages. The same signal can arrive 3+ times from retweets, echoes, and multi-user mentions. Without multi-layer dedup and signal routing, the bot processes duplicates as new entries — emabot entered SPY 11 times from the same signal.

---

## The Iron Law

> **DEDUP AT EVERY LAYER. A SIGNAL MUST PASS BROWSER DEDUP, WEBHOOK DEDUP, AND DATABASE DEDUP. ANY SINGLE LAYER CAN FAIL.**
>
> Origin: emabot had 10+ dedup commits. Same signal processed 3x because only one dedup layer existed. Content-based hashing was added after username normalization failures caused "TraderJoe" and "traderjoe" to be treated as different sources.

---

## Core Patterns

### Pattern 1: Three-Layer Dedup Architecture

```
Layer 1: BROWSER (IndexedDB)
  → Hash(user + ticker + action + 5min_window) before queueing
  → Catches: DOM re-renders, rapid duplicate events

Layer 2: WEBHOOK (Server memory / Redis)
  → Check content_hash in recent-signals cache (TTL: 10 min)
  → Catches: Extension reconnect replays, network retries

Layer 3: DATABASE (PostgreSQL)
  → INSERT ... ON CONFLICT (content_hash) DO NOTHING
  → Catches: Multi-source duplicates (extension + scraper + webhook)
```

```python
# Layer 2: Server-side webhook dedup
async def handle_signal_webhook(payload: dict) -> dict:
    content_hash = payload.get("signal_hash")
    if not content_hash:
        content_hash = compute_content_hash(payload)

    # Check recent cache (Redis or in-memory with TTL)
    if await redis.exists(f"signal:{content_hash}"):
        return {"status": "duplicate", "hash": content_hash}

    # Set with TTL — auto-expires after window
    await redis.setex(f"signal:{content_hash}", 600, "1")  # 10 min TTL

    # Layer 3: DB atomic insert
    inserted = await db.execute("""
        INSERT INTO signals (content_hash, ticker, action, source, created_at)
        VALUES ($1, $2, $3, $4, NOW())
        ON CONFLICT (content_hash) DO NOTHING
        RETURNING id
    """, content_hash, payload["ticker"], payload["action"], payload["source"])

    if not inserted:
        return {"status": "duplicate", "hash": content_hash}

    await signal_engine.process(payload)
    return {"status": "accepted", "hash": content_hash}
```

### Pattern 2: Content-Based Hash Design

The hash must be stable across retries, normalized across format variations, and windowed to prevent stale matches.

```python
import hashlib

def compute_content_hash(signal: dict) -> str:
    """Stable hash for dedup. Same signal within 5-min window = same hash."""
    username = normalize_username(signal.get("username", ""))
    ticker = signal.get("ticker", "").upper().strip()
    action = signal.get("action", "").upper().strip()
    # Floor timestamp to 5-minute boundary
    ts = signal.get("timestamp", 0)
    window_key = str(int(ts) // 300)  # 300 seconds = 5 minutes

    raw = f"{username}|{ticker}|{action}|{window_key}"
    return hashlib.sha256(raw.encode()).hexdigest()[:16]
```

### Pattern 3: Username Normalization

```python
import re

def normalize_username(raw: str) -> str:
    """Normalize chat usernames for consistent dedup and tracking.

    Handles: case, whitespace, display names vs handles, unicode.
    """
    # BAD: Direct comparison — "TraderJoe" != "traderjoe" != "Trader Joe"

    # GOOD: Normalize to canonical form
    normalized = raw.strip().lower()
    normalized = re.sub(r'\s+', '', normalized)      # Remove all whitespace
    normalized = re.sub(r'[^\w]', '', normalized)     # Remove special chars
    return normalized

# "TraderJoe" → "traderjoe"
# "Trader Joe" → "traderjoe"
# "  TRADER_JOE  " → "trader_joe" → "traderjoe"
```

### Pattern 4: Signal Tier Routing

Not all signals are equal. CRITICAL signals (CLOSE ALL, STOP LOSS) must bypass slow validation queues.

```python
from enum import Enum

class SignalTier(Enum):
    CRITICAL = "critical"  # CLOSE ALL, STOP LOSS — immediate action
    HIGH = "high"          # New entry from Tier-1 trader — fast queue
    NORMAL = "normal"      # Standard signal — full validation
    LOW = "low"            # Commentary, updates — log only

def classify_signal(signal: dict) -> SignalTier:
    text = signal.get("raw_text", "").upper()

    # CRITICAL: emergency actions
    if any(kw in text for kw in ["CLOSE ALL", "STOP LOSS", "KILL", "EXIT ALL"]):
        return SignalTier.CRITICAL

    # HIGH: trusted traders with actionable signals
    if signal.get("user_tier") == "TIER_1" and signal.get("ticker"):
        return SignalTier.HIGH

    # NORMAL: has ticker and direction
    if signal.get("ticker") and signal.get("action"):
        return SignalTier.NORMAL

    return SignalTier.LOW

async def route_signal(signal: dict):
    tier = classify_signal(signal)

    if tier == SignalTier.CRITICAL:
        await execute_immediately(signal)  # Skip validation queue
    elif tier == SignalTier.HIGH:
        await fast_validation_queue.put(signal)  # Abbreviated checks
    elif tier == SignalTier.NORMAL:
        await standard_queue.put(signal)  # Full 5-gate validation
    else:
        await log_signal(signal)  # Log only, no action
```

### Pattern 5: Ticker Extraction from Noisy Chat

```python
import re

# Known non-ticker words that look like tickers
FALSE_TICKERS = {"I", "A", "AM", "PM", "CEO", "IPO", "ATH", "ATL",
                 "IMO", "FOMO", "FYI", "LOL", "USD", "EUR", "BTC",
                 "ETH", "NFT", "AI", "IT", "OK", "US", "UK", "EU"}

def extract_tickers(text: str) -> list[str]:
    """Extract stock tickers from chat message text."""
    # Match $TICKER or standalone 1-5 letter uppercase words
    dollar_tickers = re.findall(r'\$([A-Z]{1,5})\b', text.upper())
    bare_tickers = re.findall(r'\b([A-Z]{1,5})\b', text)

    # Dollar-prefixed tickers have high confidence
    tickers = list(set(dollar_tickers))

    # Bare tickers need filtering
    for t in bare_tickers:
        if t not in FALSE_TICKERS and t not in tickers:
            tickers.append(t)

    return tickers
```

### Pattern 6: Per-Ticker Cooldown Across All Entry Paths

```python
import time
import threading

class UnifiedCooldown:
    """Global cooldown — shared across ALL entry paths (chat, webhook, cloud, flow)."""

    def __init__(self, window_seconds: int = 300):
        self._cooldowns: dict[str, float] = {}
        self._lock = threading.Lock()
        self._window = window_seconds

    def check_and_set(self, ticker: str, direction: str) -> bool:
        """Returns True if allowed (not in cooldown). Sets cooldown if allowed."""
        key = f"{ticker.upper()}:{direction.upper()}"
        now = time.time()

        with self._lock:
            # Prune stale entries to prevent memory leak
            if len(self._cooldowns) > 1000:
                cutoff = now - self._window
                self._cooldowns = {k: v for k, v in self._cooldowns.items() if v > cutoff}

            last = self._cooldowns.get(key)
            if last and (now - last) < self._window:
                return False  # In cooldown — reject

            self._cooldowns[key] = now
            return True  # Allowed — cooldown set
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| Single-layer dedup (DB only) | Network retries bypass Redis/memory check | 3-layer: browser → server cache → DB |
| Username comparison without normalization | "TraderJoe" ≠ "traderjoe" → duplicate signals | `normalize_username()` before hashing |
| No 5-minute window in hash | Same signal at 10:01 and 10:04 has different hash | Floor timestamp to 5-min boundary in hash |
| Per-engine cooldown dicts | Each engine has independent cooldown → duplicates | Single `UnifiedCooldown` shared by all paths |
| `setInterval` queue processing | Missed signals between intervals, races on concurrent calls | Event-driven: process on arrival + periodic sweep |
| No tier routing (all signals same queue) | CRITICAL "CLOSE ALL" waits behind 50 normal signals | Classify and route by tier immediately |
| Unbounded cooldown dict | Memory leak — dict grows forever | Prune entries older than window when size > 1000 |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Browser-side signal capture | `trading-bot-skills:chrome-extension-signal-bridge` |
| Server-side webhook handling | `trading-bot-skills:signal-source-integration` |
| Order dedup after signal accepted | `trading-bot-skills:order-execution-integrity` |
| Multi-engine duplicate prevention | `trading-bot-skills:multi-engine-coordination` |
| Database atomic insert | `trading-bot-skills:database-safety-for-trading` |
| Signal confidence scoring | `trading-bot-skills:confidence-thresholds` |
