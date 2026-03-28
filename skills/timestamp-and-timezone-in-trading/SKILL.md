---
name: timestamp-and-timezone-in-trading
description: "Use when handling dates, times, or timestamps in trading systems, implementing market hours checks, storing timestamps in databases, or debugging timezone-related bugs including DST transitions"
---

# Timestamp and Timezone Handling in Trading

Trading systems span multiple timezones (UTC storage, ET trading logic, user display timezone), multiple systems (Docker containers, browsers, webhooks, databases), and multiple DST transitions. A single naive datetime comparison can cause trades outside market hours, wrong DTE calculations, or duplicate signal processing.

---

## The Iron Law

> **STORE UTC. TRADE IN ET. DISPLAY IN USER TZ. NEVER USE NAIVE DATETIMES.**
>
> Origin: emabot had 25+ commits fixing timezone bugs. DST transitions caused market hours misclassification. `datetime.now()` (naive) compared to `datetime.now(timezone.utc)` (aware) raised TypeError. Hardcoded UTC-5 offsets broke during EDT. Streamlit `@st.cache_data` deserialized datetime objects as strings, crashing `.hour` access.

---

## Core Patterns

### Pattern 1: Always Use ZoneInfo (Never pytz, Never Hardcoded Offsets)

```python
from datetime import datetime, timezone, time
from zoneinfo import ZoneInfo

ET = ZoneInfo("America/New_York")  # Handles EST/EDT automatically

# BAD: Hardcoded offset — wrong 6 months of the year
now_et = datetime.now(timezone.utc) - timedelta(hours=5)  # EST only!

# BAD: pytz requires .localize() and has DST edge cases
import pytz
eastern = pytz.timezone("US/Eastern")
now_et = eastern.localize(datetime.now())  # Fragile near DST boundaries

# GOOD: ZoneInfo handles DST automatically
now_et = datetime.now(ET)

# GOOD: Convert UTC to ET
utc_time = datetime.now(timezone.utc)
et_time = utc_time.astimezone(ET)
```

### Pattern 2: Market Hours Check (Fail-Closed)

```python
MARKET_OPEN = time(9, 30)
MARKET_CLOSE = time(16, 0)

def is_market_open() -> bool:
    """Check if US equity market is open. Fail-closed on error."""
    try:
        now_et = datetime.now(ET)
        # Weekend check
        if now_et.weekday() >= 5:  # Saturday=5, Sunday=6
            return False
        current_time = now_et.time()
        return MARKET_OPEN <= current_time < MARKET_CLOSE
    except Exception:
        return False  # FAIL-CLOSED: if timezone fails, assume market closed

# BAD: Fails for 10:00-10:29 because minute < 30
if now.hour >= 9 and now.minute >= 30:  # WRONG at 10:15!

# GOOD: Use time() comparison
if now_et.time() >= time(9, 30):
```

### Pattern 3: Database Storage (Always UTC)

```python
# BAD: Store local time — ambiguous during DST transition
cursor.execute(
    "INSERT INTO trades (opened_at) VALUES (%s)",
    (datetime.now(),)  # Naive! Which timezone? Unknown.
)

# GOOD: Store UTC with timezone info
cursor.execute(
    "INSERT INTO trades (opened_at) VALUES (%s)",
    (datetime.now(timezone.utc),)
)

# Schema: always use TIMESTAMPTZ, never TIMESTAMP
# BAD:  opened_at TIMESTAMP
# GOOD: opened_at TIMESTAMPTZ
```

### Pattern 4: DTE Calculation (Date-Only, No Time Component)

```python
from datetime import date

# BAD: Using datetime subtraction — timezone affects date boundary
dte = (expiry_datetime - datetime.now()).days  # Could be off by 1 near midnight

# GOOD: Use date objects for DTE — no timezone ambiguity
today_et = datetime.now(ET).date()
expiry_date = expiry_datetime.astimezone(ET).date()
dte = (expiry_date - today_et).days
```

### Pattern 5: Pandas .dt Accessor Safety

```python
# BAD: .dt accessor crashes on non-datetime columns or all-NaT
df["hour"] = df["timestamp"].dt.hour  # AttributeError if column is string!

# GOOD: Ensure dtype before .dt access
if pd.api.types.is_datetime64_any_dtype(df["timestamp"]):
    df["hour"] = df["timestamp"].dt.hour
else:
    df["timestamp"] = pd.to_datetime(df["timestamp"], utc=True, errors="coerce")
    df["hour"] = df["timestamp"].dt.hour
```

### Pattern 6: Streamlit Cache Datetime Deserialization

```python
# BAD: Streamlit @st.cache_data can return strings instead of datetimes
@st.cache_data(ttl=60)
def get_trades():
    return fetch_trades_from_db()  # Returns datetime objects

# After cache hit, datetime fields may be ISO strings!
trade = get_trades()[0]
hour = trade["opened_at"].hour  # AttributeError: 'str' has no attr 'hour'

# GOOD: Always verify type after cache retrieval
def safe_datetime(val) -> datetime:
    if isinstance(val, str):
        return datetime.fromisoformat(val)
    if isinstance(val, datetime):
        return val
    raise TypeError(f"Expected datetime, got {type(val)}")

hour = safe_datetime(trade["opened_at"]).hour
```

### Pattern 7: ZoneInfo Fallback (Fail-Closed)

```python
# BAD: Fall back to hardcoded offset
try:
    et = ZoneInfo("America/New_York")
except Exception:
    et = timezone(timedelta(hours=-5))  # WRONG during EDT!

# GOOD: Fall back to UTC and fail-closed
try:
    ET = ZoneInfo("America/New_York")
except Exception:
    ET = timezone.utc  # UTC is ALWAYS correct — but market hours will fail-closed
    logger.critical("ZoneInfo unavailable — market hours check will block all trades")
```

---

## The Three-Layer Timezone Architecture

```
Layer 1: STORAGE (Database)     → Always UTC (TIMESTAMPTZ columns)
Layer 2: TRADING LOGIC (Bot)    → Always ET (market hours, DTE, entry cutoffs)
Layer 3: DISPLAY (Dashboard)    → User's local timezone (or ET for US markets)

Conversions happen at boundaries:
  DB → Bot:  utc_time.astimezone(ET)
  Bot → DB:  et_time.astimezone(timezone.utc)
  DB → UI:   utc_time.astimezone(user_tz)
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| `datetime.now()` without timezone | Creates naive datetime — can't compare with aware datetimes | `datetime.now(timezone.utc)` or `datetime.now(ET)` |
| `timedelta(hours=-5)` or `timedelta(hours=-4)` | Hardcoded offset — wrong 6 months/year during DST | `ZoneInfo("America/New_York")` handles DST |
| `import pytz` | pytz requires `.localize()`, has subtle DST edge cases | `from zoneinfo import ZoneInfo` (stdlib since 3.9) |
| `TIMESTAMP` column type (no TZ) | Ambiguous timezone — depends on server/session settings | `TIMESTAMPTZ` always stores UTC |
| `df["col"].dt.hour` without dtype check | Crashes on string columns or all-NaT columns | Check `is_datetime64_any_dtype()` first |
| `now.hour >= 9 and now.minute >= 30` | Wrong for 10:00-10:29 (minute < 30 but after open!) | `now.time() >= time(9, 30)` |
| ZoneInfo fallback to hardcoded offset | Falls back to wrong offset during DST | Fall back to UTC + fail-closed behavior |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Timestamps in database queries | `trading-bot-skills:database-safety-for-trading` |
| Market data freshness checks | `trading-bot-skills:market-data-pipeline` |
| Options DTE calculation | `trading-bot-skills:options-trading-safety` |
| EOD flatten timing | `trading-bot-skills:eod-liquidation` |
| Streamlit dashboard display | `trading-bot-skills:streamlit-dashboard-patterns` |
| Dedup timestamp windows | `trading-bot-skills:chat-signal-parsing-and-dedup` |
