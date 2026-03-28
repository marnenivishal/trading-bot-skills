---
name: chrome-extension-signal-bridge
description: "Use when building a Chrome Extension to scrape trading signals from web chatrooms, designing browser-to-backend signal bridges, or implementing client-side message queuing and offline retry"
---

# Chrome Extension Signal Bridge

A Chrome Extension that captures trading signals from web chatrooms is an unreliable transport layer — it runs in a browser that can crash, on a network that can drop, inside a service worker that Chrome can kill. The backend must function without it, and every signal must survive extension restarts and network failures.

---

## The Iron Law

> **THE EXTENSION IS AN UNRELIABLE TRANSPORT. THE BACKEND MUST FUNCTION WITHOUT IT. EVERY SIGNAL MUST SURVIVE CRASH, NETWORK FAILURE, AND BROWSER RESTART.**
>
> Origin: emabot went through 3 extension versions (v1, v2, v3). Signals were lost during network failures because v1 had no offline queue. Duplicate signals appeared when the extension reconnected and resent its queue without dedup. No heartbeat meant the backend couldn't tell if the extension was running or dead.

---

## Core Patterns

### Pattern 1: IndexedDB Offline Queue

Signals must be persisted client-side BEFORE sending. Only delete after confirmed delivery.

```javascript
// BAD: Fire-and-forget — signal lost on network failure
async function sendSignal(signal) {
  await fetch("/api/webhook", { method: "POST", body: JSON.stringify(signal) });
}

// GOOD: Queue in IndexedDB, send with retry, delete on success
async function queueSignal(signal) {
  const hash = await contentHash(signal);
  const record = { ...signal, hash, status: "pending", retries: 0 };
  await db.put("signals", record);  // Persists across crashes
  await processPendingQueue();
}

async function processPendingQueue() {
  const pending = await db.getAllFromIndex("signals", "status", "pending");
  for (const record of pending) {
    try {
      const resp = await fetch("/api/webhook", {
        method: "POST",
        body: JSON.stringify(record),
        signal: AbortSignal.timeout(5000),
      });
      if (resp.ok) {
        await db.delete("signals", record.hash);
      } else {
        await incrementRetry(record);
      }
    } catch (e) {
      await incrementRetry(record);  // Network error — retry later
    }
  }
}
```

### Pattern 2: Content-Based Dedup (Client-Side)

Prevent the same signal from being queued twice even if the DOM fires duplicate events.

```javascript
async function contentHash(signal) {
  const raw = `${signal.username}|${signal.ticker}|${signal.action}|${
    Math.floor(signal.timestamp / 300000)  // 5-min window bucket
  }`;
  const buf = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(raw));
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, "0")).join("");
}

async function queueIfNew(signal) {
  const hash = await contentHash(signal);
  const existing = await db.get("signals", hash);
  if (existing) return;  // Already queued or sent
  await queueSignal({ ...signal, hash });
}
```

### Pattern 3: MV3 Service Worker Lifecycle

Chrome Manifest V3 service workers are killed after ~30 seconds of inactivity. Use `chrome.alarms` for reliable retries.

```javascript
// BAD: setInterval in service worker — dies when worker is killed
setInterval(processPendingQueue, 30000);

// GOOD: chrome.alarms survives worker restarts
chrome.alarms.create("retry-queue", { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === "retry-queue") {
    processPendingQueue();
  }
});

// Also process on every new signal (opportunistic)
chrome.runtime.onMessage.addListener((msg) => {
  if (msg.type === "new-signal") {
    queueIfNew(msg.signal).then(processPendingQueue);
  }
});
```

### Pattern 4: Heartbeat Mechanism

The backend must know if the extension is alive. Send periodic heartbeats; alert if stale.

```javascript
// Extension: send heartbeat every 60 seconds
chrome.alarms.create("heartbeat", { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === "heartbeat") {
    fetch("/api/heartbeat", {
      method: "POST",
      body: JSON.stringify({ source: "extension", ts: Date.now() }),
    }).catch(() => {});  // Fire-and-forget — don't block on heartbeat failure
  }
});
```

```python
# Backend: track last heartbeat, alert if stale
async def check_extension_health():
    last_hb = await db.get_last_heartbeat("extension")
    if last_hb is None or (now_utc() - last_hb).seconds > 180:
        logger.warning("Extension heartbeat stale — switching to scraper fallback")
        await activate_scraper_fallback()
```

### Pattern 5: Webhook Payload Design

```python
# Webhook payload — every field needed for server-side processing
{
    "signal_hash": "a1b2c3...",       # Client-side content hash for dedup
    "timestamp": "2026-03-27T10:30:00Z",  # UTC ISO 8601
    "source": "extension",            # Identifies signal source
    "raw_text": "AAPL calls looking good",  # Original message
    "username": "TraderJoe",          # Normalized username
    "parsed": {                       # Client-side parsed fields (optional)
        "ticker": "AAPL",
        "action": "CALL",
        "confidence": "medium"
    }
}
```

### Pattern 6: Fallback Chain

```
Priority 1: Chrome Extension (live DOM scrape)  → Preferred: real-time, accurate
     ↓ (heartbeat stale > 3 min)
Priority 2: Playwright Scraper (headless)        → Backup: slower, fragile
     ↓ (circuit breaker open)
Priority 3: Webhook-only mode                    → Degraded: manual signals only
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| No offline queue (IndexedDB) | Signals lost on network failure | Queue before send, delete after 200 OK |
| `setInterval` in MV3 service worker | Worker killed after 30s idle — interval dies | `chrome.alarms` for persistent scheduling |
| No content-hash dedup on client | Duplicate DOM events → duplicate signals | Hash(user + ticker + action + 5min window) |
| No heartbeat to backend | Backend can't tell if extension is dead | 60s heartbeat + 180s staleness threshold |
| Sending raw DOM HTML to backend | Fragile, breaks on any chatroom UI change | Parse in extension, send structured payload |
| No retry with backoff | Single failure = permanent signal loss | Exponential backoff with max 5 retries |
| MV2 persistent background page | Deprecated — Chrome will force MV3 migration | Use MV3 service worker + chrome.alarms |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Server-side signal dedup | `trading-bot-skills:chat-signal-parsing-and-dedup` |
| Webhook receiver hardening | `trading-bot-skills:signal-source-integration` |
| Scraper fallback in Docker | `trading-bot-skills:docker-and-scraper-reliability` |
| Signal validation pipeline | `trading-bot-skills:strategy-signal-validation` |
| Async webhook processing | `trading-bot-skills:async-reliability` |
