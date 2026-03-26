---
name: mcp-broker-integration
description: Use when integrating Claude with brokerage APIs via Model Context Protocol, configuring MCP servers for trading, implementing deterministic trade execution loops, or setting up human-in-the-loop approval for live trades
---

# MCP Broker Integration

## The Iron Law

**EVERY MCP TRADE EXECUTION FOLLOWS THE DETERMINISTIC LOOP: DISCOVERY, CONTEXT, PREFLIGHT, CONFIRMATION, EXECUTION. NO STEPS SKIPPED.**

No shortcuts. No "the user seems confident so skip confirmation." No "preflight is slow so
skip it." The loop exists because markets are unforgiving and MCP tool calls are the ONLY
source of truth. Skip a step and you are guessing. Guessing with real money is gambling.

---

## The NxM Problem and MCP Solution

### Why MCP Exists

Without MCP, connecting AI models to brokers requires custom integrations for every
combination:

```
Traditional approach: N models × M brokers = N×M integrations

  Claude ──── Alpaca         (custom integration)
  Claude ──── Public.com     (custom integration)
  Claude ──── IBKR           (custom integration)
  GPT ────── Alpaca          (custom integration)
  GPT ────── Public.com      (custom integration)
  GPT ────── IBKR            (custom integration)
  ...

  3 models × 3 brokers = 9 integrations
  10 models × 10 brokers = 100 integrations
```

MCP standardizes this to N+M:

```
MCP approach: N models + M brokers = N+M implementations

  Claude ─┐
  GPT ────┤── MCP Protocol ──┤── Alpaca MCP Server
  Gemini ─┘                  ├── Public.com MCP Server
                             └── IBKR MCP Server

  3 models + 3 brokers = 6 implementations
  10 models + 10 brokers = 20 implementations
```

### MCP Architecture

```
┌──────────────────────────────────────────────────────────┐
│  HOST (Claude Desktop / Claude Code / Custom App)        │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                    │
│  │  MCP Client  │    │  MCP Client  │   (one per server) │
│  └──────┬───────┘    └──────┬───────┘                    │
└─────────┼───────────────────┼────────────────────────────┘
          │ STDIO/SSE         │ STDIO/SSE
          ▼                   ▼
   ┌──────────────┐    ┌──────────────┐
   │  Public.com  │    │   Alpaca     │
   │  MCP Server  │    │  MCP Server  │
   └──────┬───────┘    └──────┬───────┘
          │ HTTPS             │ HTTPS
          ▼                   ▼
   ┌──────────────┐    ┌──────────────┐
   │  Public.com  │    │   Alpaca     │
   │  REST API    │    │  REST API    │
   └──────────────┘    └──────────────┘
```

### Role Comparison

| Component      | Role                                   | Example                          |
|----------------|----------------------------------------|----------------------------------|
| **Host**       | Application housing the AI model       | Claude Desktop, Claude Code      |
| **Client**     | Maintains 1:1 connection to a server   | MCP Client instance              |
| **Server**     | Exposes tools, resources, prompts      | Public.com MCP Server            |
| **Transport**  | Communication channel                  | STDIO (local), SSE (remote)      |
| **Tool**       | Discrete function the model can call   | `get_option_chain`, `place_order`|
| **Resource**   | Data the model can read                | Account balance, positions       |

### Key Distinction: STDIO vs SSE

- **STDIO**: Server runs locally on your machine. API keys never leave your box. Lower
  latency. Used by Public.com MCP server.
- **SSE (Server-Sent Events)**: Server runs remotely. Requires network access. Used for
  cloud-hosted MCP servers.

For trading, **STDIO is strongly preferred** because API credentials remain local.

---

## The 5-Step Deterministic Execution Loop

This is the core of the skill. Every MCP-driven trade follows these exact steps.

```
  ┌─────────────┐
  │ 1. DISCOVERY│ ── Identify intent, enumerate tools
  └──────┬──────┘
         ▼
  ┌──────────────────┐
  │ 2. CONTEXT       │ ── Gather market data via MCP tools
  └──────┬───────────┘
         ▼
  ┌──────────────────┐
  │ 3. PREFLIGHT     │ ── Estimate cost, validate feasibility
  └──────┬───────────┘
         ▼
  ┌──────────────────┐
  │ 4. CONFIRMATION  │ ── Human approves or rejects
  └──────┬───────────┘
         ▼
  ┌──────────────────┐
  │ 5. EXECUTION     │ ── Submit order, verify fill
  └──────────────────┘
```

### Implementation

```python
import time
import logging
from dataclasses import dataclass, field
from typing import Optional, Dict, Any, List
from enum import Enum

logger = logging.getLogger(__name__)


class LoopStep(Enum):
    DISCOVERY = "discovery"
    CONTEXT = "context"
    PREFLIGHT = "preflight"
    CONFIRMATION = "confirmation"
    EXECUTION = "execution"


class LoopResult(Enum):
    SUCCESS = "success"
    ABORTED_BY_USER = "aborted_by_user"
    FAILED_PREFLIGHT = "failed_preflight"
    FAILED_EXECUTION = "failed_execution"
    FAILED_CONTEXT = "failed_context"
    TOOL_NOT_FOUND = "tool_not_found"
    TIMEOUT = "timeout"


@dataclass
class ExecutionState:
    """Tracks state through all 5 steps. Nothing is optional after its step completes."""
    step: LoopStep = LoopStep.DISCOVERY
    available_tools: List[str] = field(default_factory=list)
    context_data: Dict[str, Any] = field(default_factory=dict)
    preflight_result: Optional[Dict[str, Any]] = None
    user_approved: Optional[bool] = None
    execution_result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    timestamps: Dict[str, float] = field(default_factory=dict)

    def record_step(self, step: LoopStep):
        self.step = step
        self.timestamps[step.value] = time.time()


class MCPExecutionLoop:
    """Implements the 5-step deterministic execution loop.

    NEVER skip a step. NEVER reorder steps.
    Each step validates the output of the previous step before proceeding.
    """

    def __init__(self, mcp_client, approval_gate, safety_wrapper):
        self.mcp = mcp_client
        self.approval_gate = approval_gate
        self.safety = safety_wrapper

    async def execute(self, trade_intent: Dict[str, Any]) -> ExecutionState:
        state = ExecutionState()

        # Step 1: Discovery
        state.record_step(LoopStep.DISCOVERY)
        try:
            tools = await self.safety.call(
                self.mcp.list_tools, timeout=10.0
            )
            state.available_tools = [t["name"] for t in tools]
            required = self._required_tools_for(trade_intent)
            missing = [t for t in required if t not in state.available_tools]
            if missing:
                state.error = f"Missing required tools: {missing}"
                logger.error("Discovery failed: %s", state.error)
                return state
        except Exception as e:
            state.error = f"Discovery failed: {e}"
            logger.error(state.error)
            return state

        # Step 2: Context Gathering
        state.record_step(LoopStep.CONTEXT)
        try:
            state.context_data = await self._gather_context(trade_intent)
            if not self._validate_context(state.context_data):
                state.error = "Context validation failed: incomplete data"
                logger.error(state.error)
                return state
        except Exception as e:
            state.error = f"Context gathering failed: {e}"
            logger.error(state.error)
            return state

        # Step 3: Preflight Analysis
        state.record_step(LoopStep.PREFLIGHT)
        try:
            state.preflight_result = await self.safety.call(
                self.mcp.call_tool,
                tool_name="preflight_multileg",
                arguments=self._build_preflight_args(trade_intent, state.context_data),
                timeout=15.0,
            )
            if not self._preflight_passes(state.preflight_result):
                state.error = f"Preflight rejected: {state.preflight_result}"
                logger.warning(state.error)
                return state
        except Exception as e:
            state.error = f"Preflight failed: {e}"
            logger.error(state.error)
            return state

        # Step 4: User Confirmation
        state.record_step(LoopStep.CONFIRMATION)
        try:
            notional_value = state.preflight_result.get("estimated_cost", 0)
            approval = await self.approval_gate.request_approval(
                trade_intent=trade_intent,
                preflight=state.preflight_result,
                context=state.context_data,
                notional_value=notional_value,
            )
            state.user_approved = approval.approved
            if not approval.approved:
                state.error = f"User rejected: {approval.reason}"
                logger.info(state.error)
                return state
        except Exception as e:
            state.error = f"Confirmation failed: {e}"
            logger.error(state.error)
            return state

        # Step 5: Execution
        state.record_step(LoopStep.EXECUTION)
        try:
            state.execution_result = await self.safety.call(
                self.mcp.call_tool,
                tool_name="place_multileg_order",
                arguments=self._build_order_args(trade_intent, state.context_data),
                timeout=30.0,
            )
            logger.info(
                "Order executed: %s", state.execution_result.get("order_id", "unknown")
            )
        except Exception as e:
            state.error = f"Execution failed: {e}"
            logger.error(state.error)
            return state

        return state

    def _required_tools_for(self, trade_intent: Dict) -> List[str]:
        """Determine which MCP tools are required for this trade type."""
        base = ["get_option_chain", "preflight_multileg", "place_multileg_order"]
        if trade_intent.get("needs_greeks", False):
            base.append("get_option_greeks")
        return base

    async def _gather_context(self, trade_intent: Dict) -> Dict[str, Any]:
        """Gather all market data needed for the trade."""
        context = {}

        # Get option chain
        chain = await self.safety.call(
            self.mcp.call_tool,
            tool_name="get_option_chain",
            arguments={
                "symbol": trade_intent["symbol"],
                "expiration": trade_intent.get("expiration"),
            },
            timeout=10.0,
        )
        context["option_chain"] = chain

        # Get Greeks if needed
        if trade_intent.get("needs_greeks", False):
            greeks = await self.safety.call(
                self.mcp.call_tool,
                tool_name="get_option_greeks",
                arguments={
                    "symbol": trade_intent["symbol"],
                    "strikes": trade_intent.get("strikes", []),
                    "expiration": trade_intent.get("expiration"),
                },
                timeout=10.0,
            )
            context["greeks"] = greeks

        # Get account info
        account = await self.safety.call(
            self.mcp.call_tool,
            tool_name="get_account",
            arguments={},
            timeout=10.0,
        )
        context["account"] = account

        return context

    def _validate_context(self, context: Dict) -> bool:
        """Validate that all required context data is present and non-stale."""
        if "option_chain" not in context:
            return False
        if "account" not in context:
            return False
        chain = context["option_chain"]
        if not chain or not chain.get("strikes"):
            return False
        return True

    def _preflight_passes(self, preflight: Dict) -> bool:
        """Check preflight result for deal-breakers."""
        if preflight.get("rejected"):
            return False
        if preflight.get("estimated_cost", 0) <= 0:
            return False
        return True

    def _build_preflight_args(self, intent: Dict, context: Dict) -> Dict:
        """Build arguments for preflight_multileg tool call."""
        return {
            "symbol": intent["symbol"],
            "legs": intent.get("legs", []),
            "order_type": intent.get("order_type", "limit"),
        }

    def _build_order_args(self, intent: Dict, context: Dict) -> Dict:
        """Build arguments for place_multileg_order tool call."""
        return {
            "symbol": intent["symbol"],
            "legs": intent.get("legs", []),
            "order_type": intent.get("order_type", "limit"),
            "time_in_force": intent.get("time_in_force", "day"),
        }
```

---

## Public.com MCP Server

Public.com provides a local STDIO-based MCP server for options trading. API keys stay on
your machine. No third-party data transit.

### Available Tools

| Tool                   | Purpose                                    | Returns                              |
|------------------------|--------------------------------------------|--------------------------------------|
| `get_option_chain`     | Live strikes and expirations for a symbol  | Strikes, bids, asks, OI, volume      |
| `get_option_greeks`    | Greeks for specific contracts              | Delta, Gamma, Theta, Vega, IV        |
| `preflight_multileg`   | Cost estimation before execution           | Est. cost, fees, buying power impact  |
| `place_multileg_order` | Execute a multileg options order           | Order ID, status, fill details        |

### Tool Call Examples

**Getting an option chain:**

```python
chain = await mcp.call_tool(
    tool_name="get_option_chain",
    arguments={
        "symbol": "SPY",
        "expiration": "2026-03-27",    # 0DTE Friday
    },
)
# Returns: {"strikes": [580, 581, 582, ...], "calls": [...], "puts": [...]}
# Each entry has: strike, bid, ask, last, volume, open_interest
```

**Getting Greeks for specific strikes:**

```python
greeks = await mcp.call_tool(
    tool_name="get_option_greeks",
    arguments={
        "symbol": "SPY",
        "strikes": [580, 582, 585],
        "expiration": "2026-03-27",
        "option_type": "call",
    },
)
# Returns per strike: delta, gamma, theta, vega, implied_volatility
# These are LIVE values, not approximations from training data
```

**Preflight a vertical spread:**

```python
preflight = await mcp.call_tool(
    tool_name="preflight_multileg",
    arguments={
        "symbol": "SPY",
        "legs": [
            {"strike": 580, "type": "call", "side": "buy", "quantity": 1},
            {"strike": 585, "type": "call", "side": "sell", "quantity": 1},
        ],
        "order_type": "limit",
        "limit_price": 2.50,
    },
)
# Returns: estimated_cost, max_loss, max_profit, fees, buying_power_impact
# This is a DRY RUN. No order is placed.
```

**Placing the order:**

```python
result = await mcp.call_tool(
    tool_name="place_multileg_order",
    arguments={
        "symbol": "SPY",
        "legs": [
            {"strike": 580, "type": "call", "side": "buy", "quantity": 1},
            {"strike": 585, "type": "call", "side": "sell", "quantity": 1},
        ],
        "order_type": "limit",
        "limit_price": 2.50,
        "time_in_force": "day",
    },
)
# Returns: order_id, status, filled_qty, average_price
```

### Security Model

- MCP server runs **locally** via STDIO -- no network intermediary
- API keys stored in `claude_desktop_config.json` on your machine
- All broker communication is direct: your machine → Public.com API
- No third-party proxy, no data aggregator, no analytics pipeline

---

## Alpaca MCP Server

Alpaca provides both paper and live trading through its MCP server. Key difference from
Public.com: Alpaca supports equities and options, with separate API endpoints for paper
vs live.

### Key Tools

| Tool                  | Purpose                              | Notes                          |
|-----------------------|--------------------------------------|--------------------------------|
| `get_account`         | Account balance, buying power, PDT   | Works for paper and live       |
| `get_positions`       | Current open positions               | Includes unrealized P&L        |
| `place_order`         | Submit equity or options order       | Supports all order types       |
| `get_market_data`     | Quotes, bars, snapshots              | REST-based, not streaming      |
| `cancel_order`        | Cancel a pending order               | Requires order_id              |

### Paper vs Live Switching

```python
# Paper trading (default -- safe for development)
# Base URL: https://paper-api.alpaca.markets

# Live trading (real money -- requires explicit opt-in)
# Base URL: https://api.alpaca.markets
```

The MCP server configuration determines which environment is active. **Never configure
live credentials in a development environment.** See `mcp-configuration-reference.md`
for config examples.

### REST vs WebSocket

- **REST (via MCP tools)**: Request-response. Used for order placement, account queries.
  Every call returns a complete response. Suitable for the deterministic execution loop.
- **WebSocket (outside MCP)**: Streaming. Used for real-time price updates, order status
  events. Not part of the MCP tool interface -- handled separately by your bot's data
  pipeline.

The execution loop uses REST exclusively. Streaming data feeds are handled by the
`market-data-pipeline` skill.

---

## Human-in-the-Loop Patterns

Not every trade needs a confirmation dialog. Paper trades should flow freely. Small live
trades should log but not block. Large live trades must pause for human review.

### Approval Tiers

| Tier | Condition                    | Action                                     |
|------|------------------------------|--------------------------------------------|
| 0    | Paper trading                | Auto-approve, log only                     |
| 1    | Live, notional < $500        | Auto-approve with full logging             |
| 2    | Live, notional $500 - $5000  | Require explicit user confirmation         |
| 3    | Live, notional > $5000       | Require confirmation + 10-second delay     |

### Implementation

```python
import asyncio
import time
import logging
from dataclasses import dataclass
from typing import Optional, Dict, Any
from enum import Enum

logger = logging.getLogger(__name__)


class TradingMode(Enum):
    PAPER = "paper"
    LIVE = "live"


@dataclass
class ApprovalResult:
    approved: bool
    tier: int
    reason: Optional[str] = None
    delay_applied: float = 0.0
    timestamp: float = 0.0


class ApprovalGate:
    """Tier-based approval gate for trade execution.

    Paper trades auto-approve. Live trades escalate based on notional value.
    The tiers are NOT configurable at runtime -- they are set at initialization
    and logged. Changing tiers requires a restart and an audit log entry.
    """

    def __init__(
        self,
        mode: TradingMode,
        tier1_threshold: float = 500.0,
        tier2_threshold: float = 5000.0,
        tier3_delay: float = 10.0,
    ):
        self.mode = mode
        self.tier1_threshold = tier1_threshold
        self.tier2_threshold = tier2_threshold
        self.tier3_delay = tier3_delay
        logger.info(
            "ApprovalGate initialized: mode=%s, tier1=%.0f, tier2=%.0f, delay=%.1fs",
            mode.value, tier1_threshold, tier2_threshold, tier3_delay,
        )

    async def request_approval(
        self,
        trade_intent: Dict[str, Any],
        preflight: Dict[str, Any],
        context: Dict[str, Any],
        notional_value: float,
    ) -> ApprovalResult:
        """Route to appropriate approval tier based on mode and notional value."""

        # Tier 0: Paper trading -- auto-approve everything
        if self.mode == TradingMode.PAPER:
            logger.info("Tier 0 auto-approve (paper): %s", trade_intent.get("symbol"))
            return ApprovalResult(
                approved=True, tier=0, reason="paper_mode", timestamp=time.time()
            )

        # Tier 1: Live, small notional -- auto-approve with logging
        if notional_value < self.tier1_threshold:
            logger.info(
                "Tier 1 auto-approve (live, $%.2f < $%.2f): %s",
                notional_value, self.tier1_threshold, trade_intent.get("symbol"),
            )
            return ApprovalResult(
                approved=True, tier=1, reason="below_tier1_threshold",
                timestamp=time.time(),
            )

        # Tier 2: Live, medium notional -- require confirmation
        if notional_value < self.tier2_threshold:
            logger.warning(
                "Tier 2 confirmation required (live, $%.2f): %s",
                notional_value, trade_intent.get("symbol"),
            )
            approved = await self._prompt_user(trade_intent, preflight, notional_value)
            return ApprovalResult(
                approved=approved, tier=2,
                reason="user_confirmed" if approved else "user_rejected",
                timestamp=time.time(),
            )

        # Tier 3: Live, large notional -- confirm + enforced delay
        logger.warning(
            "Tier 3 confirmation + delay required (live, $%.2f): %s",
            notional_value, trade_intent.get("symbol"),
        )
        approved = await self._prompt_user(trade_intent, preflight, notional_value)
        if approved:
            logger.info("Tier 3 enforced delay: %.1f seconds", self.tier3_delay)
            await asyncio.sleep(self.tier3_delay)
        return ApprovalResult(
            approved=approved, tier=3,
            reason="user_confirmed_with_delay" if approved else "user_rejected",
            delay_applied=self.tier3_delay if approved else 0.0,
            timestamp=time.time(),
        )

    async def _prompt_user(
        self, intent: Dict, preflight: Dict, notional: float
    ) -> bool:
        """Display trade details and ask for confirmation.

        In a CLI context, this prints to stdout and reads from stdin.
        In a GUI context, this would show a dialog.
        Override this method for your specific UI.
        """
        print("\n" + "=" * 60)
        print("TRADE CONFIRMATION REQUIRED")
        print("=" * 60)
        print(f"  Symbol:    {intent.get('symbol', 'N/A')}")
        print(f"  Strategy:  {intent.get('strategy', 'N/A')}")
        print(f"  Notional:  ${notional:,.2f}")
        print(f"  Est. Cost: ${preflight.get('estimated_cost', 0):,.2f}")
        print(f"  Max Loss:  ${preflight.get('max_loss', 0):,.2f}")
        print(f"  Fees:      ${preflight.get('fees', 0):,.2f}")
        if intent.get("legs"):
            print("  Legs:")
            for leg in intent["legs"]:
                print(f"    {leg.get('side', '?')} {leg.get('quantity', 0)}x "
                      f"{leg.get('strike', '?')} {leg.get('type', '?')}")
        print("=" * 60)
        response = input("Approve? (yes/no): ").strip().lower()
        return response in ("yes", "y")
```

---

## MCP Error Handling

Every MCP tool call can fail. Networks drop, servers crash, APIs rate-limit. Every call
must be wrapped with timeout, retry, and fallback logic.

### Failure Modes

| Failure              | Cause                              | Handling                                |
|----------------------|------------------------------------|-----------------------------------------|
| Tool not found       | Server misconfigured or outdated   | Abort loop at Discovery step            |
| Server timeout       | Network issue or overloaded server | Retry with backoff, max 3 attempts      |
| Partial response     | Server returned incomplete data    | Validate response schema, reject if bad |
| Connection drop      | STDIO pipe broken, process crashed | Detect, log, do NOT retry blind         |
| Auth failure         | Expired or invalid API key         | Abort immediately, alert operator       |
| Rate limit           | Too many requests                  | Back off, respect Retry-After header    |

### Safety Wrapper

```python
import asyncio
import time
import logging
from typing import Callable, Any, Optional
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class MCPCallResult:
    success: bool
    data: Any = None
    error: Optional[str] = None
    attempts: int = 0
    elapsed_ms: float = 0.0


class MCPSafetyWrapper:
    """Wraps every MCP tool call with timeout, retry, and validation.

    EVERY MCP call goes through this wrapper. No direct mcp.call_tool() calls
    anywhere in the codebase. If you see a raw call_tool without the wrapper,
    that is a bug.
    """

    def __init__(self, max_retries: int = 3, base_delay: float = 1.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.call_count = 0
        self.failure_count = 0

    async def call(
        self,
        func: Callable,
        timeout: float = 10.0,
        **kwargs,
    ) -> Any:
        """Call an MCP function with timeout and retry.

        Args:
            func: The MCP function to call (e.g., mcp.call_tool)
            timeout: Maximum seconds to wait for response
            **kwargs: Arguments passed to func

        Returns:
            The function result on success

        Raises:
            MCPTimeoutError: If all retries exhausted
            MCPConnectionError: If connection is broken (no retry)
            MCPAuthError: If authentication failed (no retry)
        """
        start = time.time()
        last_error = None

        for attempt in range(1, self.max_retries + 1):
            self.call_count += 1
            try:
                result = await asyncio.wait_for(
                    func(**kwargs), timeout=timeout
                )
                elapsed = (time.time() - start) * 1000
                logger.debug(
                    "MCP call succeeded: attempt=%d, elapsed=%.1fms", attempt, elapsed
                )
                return result

            except asyncio.TimeoutError:
                last_error = f"Timeout after {timeout}s (attempt {attempt})"
                logger.warning(last_error)

            except ConnectionError as e:
                # Connection broken -- do NOT retry, pipe is dead
                self.failure_count += 1
                raise MCPConnectionError(f"Connection lost: {e}") from e

            except PermissionError as e:
                # Auth failure -- do NOT retry, credentials are bad
                self.failure_count += 1
                raise MCPAuthError(f"Auth failed: {e}") from e

            except Exception as e:
                last_error = f"MCP call error (attempt {attempt}): {e}"
                logger.warning(last_error)

            # Exponential backoff before retry
            if attempt < self.max_retries:
                delay = self.base_delay * (2 ** (attempt - 1))
                logger.info("Retrying in %.1fs...", delay)
                await asyncio.sleep(delay)

        self.failure_count += 1
        raise MCPTimeoutError(
            f"All {self.max_retries} attempts failed. Last error: {last_error}"
        )


class MCPTimeoutError(Exception):
    """All retry attempts exhausted."""
    pass


class MCPConnectionError(Exception):
    """MCP server connection lost. Do not retry."""
    pass


class MCPAuthError(Exception):
    """Authentication failed. Do not retry."""
    pass
```

---

## Why MCP Eliminates Hallucination Risk

This is the fundamental reason MCP matters for trading.

### The Problem with LLM-Generated Market Data

Without MCP, when you ask an AI "What is the delta of the SPY 580 call expiring Friday?",
the model can only:

1. **Recall from training data** -- stale, possibly months or years old
2. **Estimate from patterns** -- a guess based on typical delta curves
3. **Refuse to answer** -- safest but unhelpful

All three are unacceptable for live trading. Training data deltas are wrong. Estimated
deltas are wrong. And refusing to answer blocks the workflow.

### The MCP Solution

With MCP, the model **never guesses**. It calls a deterministic tool:

```
Model thinks: "User wants delta for SPY 580C expiring Friday"
Model calls:  get_option_greeks(symbol="SPY", strikes=[580], expiration="2026-03-27")
Server returns: {"580": {"delta": 0.45, "gamma": 0.03, "theta": -0.15, "vega": 0.12}}
Model reports: "The delta is 0.45"
```

The model is acting as a **router**, not a **calculator**. It decides WHICH tool to call
and HOW to present the result. It does not compute the delta. It does not approximate
the delta. The delta comes from the broker's pricing engine via a deterministic API call.

### Comparison

| Aspect             | Without MCP (Training Data)    | With MCP (Live Tool Calls)     |
|--------------------|--------------------------------|--------------------------------|
| Price data         | Stale, from training cutoff    | Real-time, from broker         |
| Greeks             | Estimated or hallucinated      | Computed by pricing engine     |
| Account balance    | Unknown                        | Exact, current                 |
| Position state     | Unknown                        | Exact, current                 |
| Strike availability| May reference delisted strikes | Only live, tradeable strikes   |
| Expiration dates   | May reference past dates       | Only valid future expirations  |

---

## Red Flags

| Red Flag                                      | Why It Is Dangerous                                                |
|-----------------------------------------------|--------------------------------------------------------------------|
| Skipping preflight before execution            | You do not know the cost, max loss, or buying power impact         |
| Using training data for prices or Greeks       | Stale data leads to wrong strike selection and mispriced spreads    |
| No timeout on MCP tool calls                   | A hung call blocks the entire execution loop indefinitely          |
| Auto-approving large live trades               | A bug in signal logic can drain the account in one order           |
| Direct `call_tool` without safety wrapper      | No retry, no timeout, no error classification                      |
| Same config for paper and live                 | One wrong env var sends paper-intended orders to live               |
| Retrying after auth failure                    | Wastes time and may trigger account lockout                        |
| Retrying after connection drop                 | The pipe is dead; retrying the same pipe is pointless              |
| Not validating MCP response schema             | Partial or malformed responses treated as valid data               |
| Hardcoding broker-specific logic in the loop   | The execution loop should be broker-agnostic via MCP abstraction   |

---

## Integration Points

This skill connects directly to:

- **`broker-api-integration`** -- MCP is the transport layer; broker-api-integration
  covers the circuit breaker, retry, and idempotency patterns that operate underneath
  MCP tool calls.

- **`order-execution-integrity`** -- After MCP `place_multileg_order` returns, the
  order lifecycle tracking (fills, partials, cancellations) follows
  order-execution-integrity patterns.

- **`risk-management-gates`** -- The preflight step in the execution loop feeds into
  risk gates. If preflight shows max loss exceeding the daily risk budget, the gate
  blocks execution before confirmation.

- **`pre-trade-validation`** -- Pre-trade checks (market open, symbol valid, liquidity
  sufficient) run BEFORE the MCP execution loop begins. The loop assumes pre-trade
  validation has already passed.

- **`confidence-thresholds`** -- The approval gate tiers can incorporate signal
  confidence. A low-confidence signal on a high-notional trade may require stricter
  approval even if the notional alone would not.
