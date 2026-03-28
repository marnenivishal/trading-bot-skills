---
name: llm-integration-for-trading-bots
description: "Use when integrating LLMs (Gemini, OpenAI, Claude) into trading systems, handling streaming response parsing, implementing provider fallback chains, managing API rate limits and costs, or extracting structured JSON from LLM responses"
---

# LLM Integration for Trading Bots

LLM responses in trading systems are untrusted input — they can be truncated mid-JSON, return unexpected formats when model versions change, and fail silently due to rate limits. A single unparseable response must never block a trade decision or crash the trading loop.

---

## The Iron Law

> **AN LLM RESPONSE IS UNTRUSTED INPUT. PARSE IT DEFENSIVELY. NEVER LET A PARSING FAILURE BLOCK A TRADE DECISION.**
>
> Origin: emabot had 129 Gemini streaming response parsing failures from truncated JSON. Model version drift (gemini-1.5-pro → gemini-2.0-flash) silently changed output format. Cost/quota exhaustion crashed the learning system. Commits 541a0e9, fe0b699, 57e4ae4 fixed these across 20+ files.

---

## Core Patterns

### Pattern 1: Accumulate Full Response Before Parsing

```python
# BAD: Parse streaming chunks incrementally — truncated JSON crashes
async for chunk in response.stream():
    data = json.loads(chunk.text)  # Crashes on partial JSON!
    process(data)

# GOOD: Accumulate full response, then parse once
async def call_llm_safe(client, prompt: str) -> dict | None:
    """Accumulate full response before parsing. Returns None on failure."""
    try:
        response = await client.generate_content(prompt)
        full_text = response.text  # Full response, not streaming
        return extract_json(full_text)
    except Exception as e:
        logger.warning(f"LLM call failed: {e}")
        return None  # Never crash — return None for caller to handle
```

### Pattern 2: Safe JSON Extraction

LLMs wrap JSON in markdown fences, add commentary, or return truncated payloads.

```python
import json
import re

def extract_json(text: str) -> dict | None:
    """Extract JSON from LLM response text. Handles markdown fences,
    commentary, and truncated payloads.
    """
    if not text:
        return None

    # Step 1: Try direct parse (ideal case)
    try:
        return json.loads(text.strip())
    except json.JSONDecodeError:
        pass

    # Step 2: Strip markdown code fences
    fenced = re.search(r'```(?:json)?\s*\n?(.*?)\n?```', text, re.DOTALL)
    if fenced:
        try:
            return json.loads(fenced.group(1).strip())
        except json.JSONDecodeError:
            pass

    # Step 3: Find JSON object boundaries
    start = text.find('{')
    end = text.rfind('}')
    if start != -1 and end != -1 and end > start:
        try:
            return json.loads(text[start:end + 1])
        except json.JSONDecodeError:
            pass

    # Step 4: Find JSON array
    start = text.find('[')
    end = text.rfind(']')
    if start != -1 and end != -1 and end > start:
        try:
            return json.loads(text[start:end + 1])
        except json.JSONDecodeError:
            pass

    logger.warning(f"Failed to extract JSON from LLM response ({len(text)} chars)")
    return None
```

### Pattern 3: Provider Fallback Chain

```python
from dataclasses import dataclass, field

@dataclass
class LLMProvider:
    name: str
    model: str
    client: object
    priority: int
    is_healthy: bool = True
    failure_count: int = 0

class LLMFallbackChain:
    """Try providers in priority order. Fall back on failure."""

    def __init__(self, providers: list[LLMProvider]):
        self.providers = sorted(providers, key=lambda p: p.priority)

    async def generate(self, prompt: str) -> dict | None:
        for provider in self.providers:
            if not provider.is_healthy:
                continue

            try:
                response = await provider.client.generate(
                    model=provider.model,
                    prompt=prompt,
                )
                result = extract_json(response.text)
                if result is not None:
                    provider.failure_count = 0
                    return result
                else:
                    provider.failure_count += 1
            except Exception as e:
                provider.failure_count += 1
                logger.warning(f"{provider.name} failed: {e}")

            # Circuit breaker: disable after 5 consecutive failures
            if provider.failure_count >= 5:
                provider.is_healthy = False
                logger.error(f"{provider.name} disabled — 5 consecutive failures")

        logger.error("ALL LLM providers failed — returning None")
        return None  # All providers failed — caller uses fallback logic

# Usage
chain = LLMFallbackChain([
    LLMProvider("gemini", "gemini-2.5-flash", gemini_client, priority=1),
    LLMProvider("openai", "gpt-4.1-nano", openai_client, priority=2),
    LLMProvider("claude", "claude-haiku-4-5-20251001", claude_client, priority=3),
])
```

### Pattern 4: Model Version Pinning

```python
# BAD: Unpinned model — behavior changes without warning
response = client.generate(model="gemini-pro")  # Which version? Changes over time!

# GOOD: Pin exact model version
GEMINI_MODEL = "gemini-2.5-flash"  # Pinned in config, not inline
response = client.generate(model=GEMINI_MODEL)

# Store model version with every LLM interaction for debugging
await db.execute(
    "INSERT INTO ai_interactions (model, prompt_tokens, response_tokens, "
    "latency_ms, success, cost_usd) VALUES ($1, $2, $3, $4, $5, $6)",
    GEMINI_MODEL, prompt_tokens, response_tokens, latency_ms, success, cost,
)
```

### Pattern 5: Cost Tracking and Budget Circuit Breaker

```python
@dataclass
class LLMBudget:
    daily_limit_usd: float = 5.00
    spent_today_usd: float = 0.0
    last_reset: str = ""  # YYYY-MM-DD

    def record_cost(self, cost_usd: float):
        today = date.today().isoformat()
        if self.last_reset != today:
            self.spent_today_usd = 0.0
            self.last_reset = today
        self.spent_today_usd += cost_usd

    def can_spend(self) -> bool:
        today = date.today().isoformat()
        if self.last_reset != today:
            return True  # New day
        return self.spent_today_usd < self.daily_limit_usd

    @property
    def remaining(self) -> float:
        return max(0, self.daily_limit_usd - self.spent_today_usd)

# Gate LLM calls behind budget check
async def call_with_budget(chain: LLMFallbackChain, budget: LLMBudget, prompt: str):
    if not budget.can_spend():
        logger.warning(f"LLM budget exhausted (${budget.spent_today_usd:.2f})")
        return None

    result = await chain.generate(prompt)
    # Estimate cost (provider-specific)
    estimated_cost = estimate_cost(prompt, result)
    budget.record_cost(estimated_cost)
    return result
```

### Pattern 6: Response Schema Validation

```python
def validate_trade_suggestion(data: dict | None) -> dict | None:
    """Validate LLM trade suggestion matches expected schema.
    Reject and return None if schema doesn't match.
    """
    if data is None:
        return None

    required_fields = ["ticker", "direction", "confidence", "reasoning"]
    for field in required_fields:
        if field not in data:
            logger.warning(f"LLM response missing required field: {field}")
            return None

    # Type checks
    if not isinstance(data["ticker"], str) or len(data["ticker"]) > 5:
        return None
    if data["direction"] not in ("CALL", "PUT", "LONG", "SHORT"):
        return None
    if not isinstance(data["confidence"], (int, float)) or not 0 <= data["confidence"] <= 1:
        return None

    return data  # Schema valid

# LLM output NEVER drives trades directly — always validate
suggestion = await chain.generate(analysis_prompt)
validated = validate_trade_suggestion(suggestion)
if validated and validated["confidence"] >= 0.7:
    await signal_engine.process(validated)  # Goes through normal risk gates
```

---

## Red Flags

| Red Flag | Why It's Dangerous | Correct Pattern |
|----------|-------------------|-----------------|
| Parsing streaming chunks as JSON | Truncated response = crash | Accumulate full response, then parse |
| `model="gemini-pro"` (unpinned) | Model behavior changes without notice | Pin exact version in config: `"gemini-2.5-flash"` |
| No fallback provider | Single API outage = total LLM failure | Fallback chain: Gemini → OpenAI → Claude |
| LLM output directly triggers trade | Hallucinated ticker, wrong direction | Schema validation + normal risk gates |
| No cost tracking | Budget exhaustion surprises, API quota exceeded | Daily budget with circuit breaker |
| `json.loads(response.text)` without try/except | Any non-JSON response crashes the bot | `extract_json()` with 4-step fallback |
| No AI interaction logging | Can't debug why LLM made a bad suggestion | Log model, tokens, latency, success, cost per call |

---

## Integration

| Scenario | Related Skill |
|----------|--------------|
| LLM confidence in signal validation | `trading-bot-skills:confidence-thresholds` |
| Async LLM calls with timeout | `trading-bot-skills:async-reliability` |
| Numeric parsing from LLM output | `trading-bot-skills:falsy-zero-and-sentinel-values` |
| LLM-driven self-tuning | `trading-bot-skills:self-tuning-and-learning-systems` |
| News sentiment analysis | `trading-bot-skills:market-sentiment-analyst` |
| Cost tracking in config | `trading-bot-skills:trading-config-management` |
