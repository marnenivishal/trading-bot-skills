---
name: ibkr-api-precautions
description: Use when IBKR API orders are not submitting, sitting idle, or requiring manual TWS dialog confirmation — diagnoses TWS Precautionary Settings and IBC bypass config BEFORE debugging code
---

# IBKR API Precautions Screener

## Why This Skill Exists

The #1 reason IBKR API orders "don't work" is NOT a code bug — it's TWS/IB Gateway
**order precaution dialogs** blocking API submissions silently. Developers waste hours
debugging order logic when the fix is a single checkbox in TWS configuration.

This skill catches that class of problem FIRST.

---

## When to Activate

Trigger this skill when the user mentions ANY of:

- "Order not going through from API but works in TWS"
- "Paper/live API orders just sit there / not submitted"
- "Need to click a dialog in TWS to accept the order"
- "IBC_BypassOrderPrecautions" / "BypassOrderPrecautions"
- "TWS order precautions" / "API precautions" / "confirmation dialogs"
- "orderStatus callback never fires"
- "Order stays in PendingSubmit forever"

**First assumption: TWS's API Precautions or IBC config are blocking the orders.**

---

## TWS Precautions Model

TWS has **Precautionary Settings** and **API Precautions** that apply to ALL API orders.

If an API order violates these, TWS will:
- **Block it** and pop up a dialog requiring manual confirmation
- **Reject it** with a precautionary error in logs
- **Hold it** in PendingSubmit state indefinitely

### Key TWS Settings

Location: **Global Configuration > API > Precautions**

| Setting | What it does |
|---------|-------------|
| Bypass Order Precautions for API Orders | Skips ALL precaution dialogs for API-submitted orders |
| Bypass Redirect Order warning for Stock API Orders | Skips exchange redirect warnings |
| Size/value limit warnings | Triggers dialog if order exceeds configured limits |
| Bond/negative yield warnings | Asset-specific precaution dialogs |

**If "Bypass Order Precautions for API Orders" is NOT checked:**
API orders may trigger dialogs and never auto-submit unless someone clicks through in TWS.

---

## IBC / ib-controller Configuration

For users running IBC around TWS/IBG (including Docker setups):

IBC can auto-handle dialogs and configure TWS API Precautions via `config.ini`.

### Relevant config.ini settings:

```ini
# API Precautions section
BypassOrderPrecautions=yes
BypassRedirectOrderWarning=yes
```

Other `Bypass*` fields may exist depending on IBC version.

### Docker-specific notes:

When running `ghcr.io/extrange/ibkr:latest` or similar IBKR Docker images:
- TWS runs headlessly via noVNC
- IBC handles initial login and dialog acceptance
- API Precautions must be set either via IBC config OR manually through noVNC
- Environment variable `IBC_BypassOrderPrecautions=yes` may be supported depending on the image

If the user sets `IBC_BypassOrderPrecautions` or similar:
- This tells IBC to configure TWS's "Bypass Order Precautions for API Orders" at startup
- Missing or misconfigured flags mean TWS still blocks on API precautions

---

## Diagnostic Checklist

**Run this BEFORE debugging code when orders aren't submitting:**

### Step 1: Manual TWS Test

Ask the user:
> "If you place this exact order manually in TWS, do you see any warning dialogs
> (size, price, redirect, risk)?"

If yes: API orders are being blocked by the same precautions.

### Step 2: Check TWS API Precautions

Instruct the user to open:
**Global Configuration > API > Precautions**

Verify:
- [ ] "Bypass Order Precautions for API Orders" is checked
- [ ] "Bypass Redirect Order warning for Stock API Orders" is checked (if trading stocks)

### Step 3: For IBC/Docker Users

Ask:
> "Are you using IBC / ib-controller to run TWS/IBG headless (e.g., Docker)?"

If yes:
- Inspect `config.ini` for `BypassOrderPrecautions=yes`
- Check Docker environment variables for `IBC_BypassOrderPrecautions`
- For the `ghcr.io/extrange/ibkr` image, check if API precautions need to be set via noVNC

### Step 4: Check Logs for Precaution Errors

Look for in TWS/API logs:
- Warnings about exceeding default size/value limits
- Precautionary messages blocking execution
- "Order held" or "precaution" keywords
- Warning code 10268 (eTradeOnly/firmQuoteOnly — set both to False on Order objects)

Translate any such messages into:
> "This order violates TWS precaution X; either lower the size/risk or
> adjust/bypass that precaution in TWS/IBC."

---

## Resolution Paths

### Path A: Keep Precautions ON (safer)

- Lower order size to stay below precaution thresholds
- Adjust TWS presets for the specific asset class
- Accept that some orders require manual confirmation

### Path B: Bypass Precautions for API (recommended for bots)

1. Check "Bypass Order Precautions for API Orders" in TWS
2. Set `BypassOrderPrecautions=yes` in IBC config.ini
3. **BUT** only after verifying the bot has:
   - Max contract/share size limits per order
   - Max notional value limits
   - Daily loss limits with kill switch
   - Position count limits

### ibapi Order Object Checklist

When creating orders via the official `ibapi` library, always set:

```python
order = Order()
order.eTradeOnly = False      # Avoid warning 10268
order.firmQuoteOnly = False   # Avoid warning 10268
# ... rest of order fields
```

---

## Risk & Safety Guidance

**Bypassing precautions is powerful and dangerous.**

"Bypass Order Precautions for API Orders" tells TWS to stop protecting the user
and trust the bot completely.

Therefore, recommend enabling bypass ONLY if:
- The bot has strong internal risk checks (max size, max notional, daily loss limits)
- The user understands the risk of runaway orders
- Kill switch and circuit breaker are implemented and tested

**NEVER:**
- Tell users to blindly turn every bypass on
- Ignore risk — whenever suggesting bypass, also suggest specific bot-side validations
- Skip this diagnostic and jump straight to code debugging

---

## Common Misdiagnoses This Skill Prevents

| Symptom | Wrong Diagnosis | Actual Cause |
|---------|----------------|--------------|
| Order stays PendingSubmit | "ibapi bug" or "connection issue" | TWS precaution dialog waiting for click |
| orderStatus callback never fires | "Callback not registered" | TWS holding order pending precaution acceptance |
| Order works in TWS but not API | "Wrong order parameters" | API orders subject to precautions, manual orders bypass them |
| Orders work on paper, fail on live | "Live account permissions" | Live account has stricter precaution defaults |

---

## Integration with Other Skills

- **broker-api-integration**: This skill runs BEFORE broker debugging — precautions must be ruled out first
- **kill-switch-and-circuit-breakers**: Required before recommending bypass — bot must have internal safety
- **order-execution-integrity**: After precautions are resolved, use this for fill/cancel/partial fill issues
