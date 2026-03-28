---
name: streamlit-dashboard-patterns
description: "Use when building Streamlit dashboards for trading systems, managing session state vs cache vs database persistence, fixing stale widget data, or implementing live-updating trading dashboards with real-time P&L"
---

# Streamlit Dashboard Patterns for Trading

Streamlit reruns the entire script on every user interaction. This execution model creates unique challenges for trading dashboards: cached position data goes stale in seconds, widget state disappears on page navigation, datetime objects deserialize as strings after caching, and settings applied via forms don't persist without explicit DB writes.

---

## The Iron Law

> **STREAMLIT RERUNS EVERYTHING ON EVERY INTERACTION. STATE THAT MUST SURVIVE RERUNS GOES IN `st.session_state`. STATE THAT MUST SURVIVE RESTARTS GOES IN THE DATABASE. CACHE IS FOR EXPENSIVE COMPUTATIONS, NOT TRADING STATE.**
>
> Origin: emabot's dashboard had 15+ UI bugs. Session state lost on page navigation. Widget state not clearing when "Apply" was clicked — old values displayed despite DB update. `@st.cache_data` returned stale position data. Datetime objects deserialized as strings after cache, crashing `.hour` access. Commits 9d611cb, 320b37e, 4eeffaf.

---

## Core Patterns

### Pattern 1: Three-Tier State Model

```python
# TIER 1: st.session_state — survives reruns within a session
# Use for: UI selections, form inputs, active tab, temporary filters
st.session_state.setdefault("selected_tab", "overview")
st.session_state.setdefault("filter_engine", "all")

# TIER 2: @st.cache_data — expensive computations with TTL
# Use for: historical analytics, static reference data, aggregations
# DO NOT use for: live positions, current P&L, open orders
@st.cache_data(ttl=300)  # 5-min TTL for historical data
def get_monthly_analytics(month: str) -> pd.DataFrame:
    return db.fetch_monthly_stats(month)

# TIER 3: Database — persistent truth
# Use for: settings, trade records, runtime config, anything that
# must survive process restart
def save_settings(settings: dict):
    db.upsert_runtime_settings(settings)  # Persists to DB
    st.cache_data.clear()  # Bust cache so next read gets fresh data
```

### Pattern 2: Live Data — Never Cache

```python
# BAD: Cached position data — stale within seconds
@st.cache_data(ttl=60)
def get_open_positions():
    return db.fetch_open_positions()
# Problem: User sees positions that closed 59 seconds ago

# GOOD: Direct DB query for live trading data (no cache)
def get_open_positions():
    """Live data — always fresh. No caching."""
    return db.fetch_open_positions()

# For the "hero P&L" — always query broker, never cache
def get_current_pnl():
    return broker.get_account().equity - broker.get_account().last_equity
```

### Pattern 3: Widget State Clearing After Form Submit

```python
# BAD: Settings saved to DB but old values still in widget state
if st.button("Apply Settings"):
    save_settings(new_settings)
    st.success("Saved!")
    # BUT widgets still show old values until next full rerun!

# GOOD: Clear session state keys that widgets depend on, then rerun
if st.button("Apply Settings"):
    save_settings(new_settings)
    # Clear widget state so widgets re-initialize from DB
    for key in ["stop_loss_pct", "max_positions", "risk_level"]:
        if key in st.session_state:
            del st.session_state[key]
    st.cache_data.clear()  # Also bust any cached settings
    st.rerun()  # Force complete rerun with fresh state
```

### Pattern 4: Datetime Deserialization After Cache

```python
from datetime import datetime

# BAD: @st.cache_data can return datetimes as ISO strings
@st.cache_data(ttl=60)
def get_trades():
    return db.fetch_recent_trades()

trade = get_trades()[0]
hour = trade["opened_at"].hour  # AttributeError: 'str' has no attribute 'hour'

# GOOD: Type-safe datetime access
def safe_datetime(val) -> datetime | None:
    """Handle datetime deserialization from Streamlit cache."""
    if val is None:
        return None
    if isinstance(val, datetime):
        return val
    if isinstance(val, str):
        try:
            return datetime.fromisoformat(val)
        except ValueError:
            return None
    return None

trade = get_trades()[0]
opened_at = safe_datetime(trade["opened_at"])
if opened_at:
    hour = opened_at.hour
```

### Pattern 5: URL State Synchronization

```python
# Allow sharing dashboard links with state (e.g., selected date, tab)
def sync_url_state():
    """Read state from URL params and apply to session state."""
    params = st.query_params

    if "tab" in params:
        st.session_state["selected_tab"] = params["tab"]
    if "date" in params:
        st.session_state["selected_date"] = params["date"]

def update_url_state():
    """Push session state to URL for bookmarking/sharing."""
    st.query_params.update({
        "tab": st.session_state.get("selected_tab", "overview"),
        "date": st.session_state.get("selected_date", ""),
    })

# Call at top of app
sync_url_state()
# Call after state changes
update_url_state()
```

### Pattern 6: Unique Widget Keys

```python
# BAD: Duplicate widget keys across pages or loops
for trade in trades:
    st.button("Close", key="close_btn")  # DuplicateWidgetID error!

# GOOD: Unique keys using IDs
for trade in trades:
    st.button("Close", key=f"close_btn_{trade['id']}")

# Also unique across pages — prefix with page name
st.selectbox("Engine", engines, key="overview_engine_filter")  # Not just "engine_filter"
```

### Pattern 7: Testing Dashboard Logic Without Streamlit

```python
# BAD: All logic lives inside Streamlit callbacks — untestable
if st.button("Execute"):
    trades = db.fetch_trades()
    filtered = [t for t in trades if t["pnl"] > 0]
    metrics = calculate_metrics(filtered)
    st.metric("Win Rate", f"{metrics['win_rate']:.1%}")

# GOOD: Extract logic into pure functions — test those
# logic.py (no Streamlit imports)
def filter_winning_trades(trades: list[dict]) -> list[dict]:
    return [t for t in trades if t["pnl"] is not None and t["pnl"] > 0]

def calculate_metrics(trades: list[dict]) -> dict:
    total = len(trades)
    winners = len(filter_winning_trades(trades))
    return {"win_rate": winners / total if total > 0 else 0.0}

# app.py (Streamlit UI only)
if st.button("Execute"):
    trades = db.fetch_trades()
    metrics = calculate_metrics(trades)
    st.metric("Win Rate", f"{metrics['win_rate']:.1%}")

# test_logic.py (no Streamlit dependency)
def test_calculate_metrics():
    trades = [{"pnl": 100}, {"pnl": -50}, {"pnl": 0}]
    result = calculate_metrics(trades)
    assert result["win_rate"] == pytest.approx(1/3)
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| `@st.cache_data` for live positions/P&L | Shows stale data — positions closed seconds ago still display | Direct DB query, no cache for live trading data |
| Widget state not cleared after "Apply" | Old values persist in UI despite DB save | Delete session_state keys + `st.rerun()` |
| `trade["opened_at"].hour` after cache | Cache deserializes datetime as string → AttributeError | `safe_datetime()` wrapper before `.hour`/`.minute` |
| Same widget key across loop iterations | `DuplicateWidgetID` error crashes page | `key=f"widget_{unique_id}"` |
| Logic embedded in Streamlit callbacks | Untestable — requires running Streamlit | Extract pure functions, test independently |
| No cache invalidation after trade actions | Dashboard shows old state after manual close/entry | `st.cache_data.clear()` after any write operation |
| Settings in `st.session_state` only | Lost on process restart — settings revert silently | Save to DB, read from DB on startup |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| Datetime handling in UI | `trading-bot-skills:timestamp-and-timezone-in-trading` |
| P&L display accuracy | `trading-bot-skills:pnl-calculation-and-reconciliation` |
| Health monitoring dashboard | `trading-bot-skills:trading-monitoring-and-alerts` |
| Settings persistence | `trading-bot-skills:trading-config-management` |
| Runtime config management | `trading-bot-skills:database-safety-for-trading` |
