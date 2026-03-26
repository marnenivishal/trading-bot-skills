---
name: trading-config-management
description: Use when implementing trading bot configuration, managing settings across environments, or when configuration exists in multiple places causing drift
---

# Trading Config Management

## Iron Law

**ONE CONFIG SOURCE. VALIDATED AT STARTUP. EVERY SETTING HAS A TYPE, DEFAULT, AND RANGE. NO MAGIC NUMBERS IN CODE.**

Configuration drift is silent. You will not know your paper trading config diverged
from live until a live bug hits. You will not know a magic number in the code overrides
your config file until the "config change" you made has no effect.

## Prevents

- **Configuration drift (#7):** Settings scattered across config files, environment
  variables, hardcoded constants, and command-line arguments. Changes to one location
  have no effect because another location takes priority.
- **Magic numbers (#10):** Constants buried in code (`VIX_THRESHOLD = 99.0`,
  `MAX_POSITIONS = 5`) that shadow config file settings and cannot be changed
  without a code deploy.

---

## Single Source of Truth

All configuration lives in ONE file. No exceptions.

- **Config file:** All trading parameters, thresholds, timeframes, strategy settings.
- **Environment variables:** ONLY for secrets (API keys, passwords, webhook URLs).
- **Command line:** ONLY for overriding the config file path or environment name.
- **Code:** ZERO magic numbers. Every constant comes from config.

```
# What goes where:

config.toml          -> All trading parameters
                        ema_fast_period = 8
                        max_positions = 10
                        risk_per_trade = 0.01
                        vix_max = 80.0

Environment vars     -> ONLY secrets
                        BROKER_API_KEY=xxx
                        TELEGRAM_BOT_TOKEN=xxx

Command line         -> ONLY meta-config
                        --config config.paper.toml
                        --env paper
```

---

## Config Schema (TOML)

```toml
# config.paper.toml
# ==========================================
# PAPER TRADING CONFIGURATION
# ==========================================
# This is the SINGLE SOURCE OF TRUTH for all trading parameters.
# DO NOT hardcode any of these values in the code.

[environment]
name = "paper"                    # "paper" or "live"
broker_endpoint = "https://paper-api.broker.com"

[risk]
max_positions = 10                # Maximum concurrent positions
max_portfolio_risk_pct = 0.05     # 5% max total portfolio risk
risk_per_trade_pct = 0.01         # 1% risk per trade
max_daily_loss_pct = 0.03         # 3% daily loss -> halt trading
max_single_loss_pct = 0.02        # 2% max loss on single position
max_correlated_positions = 3      # Max positions in same sector

[strategy.ema_crossover]
enabled = true
fast_period = 8
slow_period = 21
confirmation_bars = 3
min_spread_atr = 0.5
timeframe = "5min"
higher_tf = "1h"

[strategy.ema_crossover.curl]
enabled = true
lookback = 12
atr_threshold = 0.15
smoothing = 3

[indicators]
atr_period = 14
rsi_period = 14
vwap_reset = "session"            # Reset VWAP each session
bollinger_period = 20
bollinger_std = 2.0

[trailing_stop]
atr_multiplier = 2.0
min_distance_pct = 0.005          # Minimum 0.5% distance
ratchet_only = true               # MUST be true -- stops never move backward

[vix_gate]
enabled = true
max_staleness_seconds = 30
min_vix = 9.0
max_vix = 80.0
halt_on_unavailable = true        # If VIX feed dies, halt entries

[data]
primary_source = "websocket"
backup_source = "rest_poll"
max_staleness_seconds = 5
poll_interval_seconds = 1
gap_detection_threshold_seconds = 10

[alerts]
telegram_enabled = true
critical_channel = "trading-critical"
warning_channel = "trading-warnings"
info_channel = "trading-audit"

[logging]
level = "INFO"
format = "json"
file = "logs/trading.log"
max_size_mb = 100
rotation = "daily"
```

---

## Startup Validation

Load and validate EVERY field at startup. If ANY validation fails, DO NOT START.

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional
import tomllib
import sys


class ConfigValidationError(Exception):
    """Raised when config validation fails. Bot must not start."""
    pass


@dataclass(frozen=True)
class RiskConfig:
    max_positions: int
    max_portfolio_risk_pct: float
    risk_per_trade_pct: float
    max_daily_loss_pct: float
    max_single_loss_pct: float
    max_correlated_positions: int

    def __post_init__(self):
        errors = []
        if not (1 <= self.max_positions <= 50):
            errors.append(f"max_positions={self.max_positions} not in [1, 50]")
        if not (0.001 <= self.risk_per_trade_pct <= 0.05):
            errors.append(
                f"risk_per_trade_pct={self.risk_per_trade_pct} not in [0.1%, 5%]"
            )
        if not (0.01 <= self.max_daily_loss_pct <= 0.10):
            errors.append(
                f"max_daily_loss_pct={self.max_daily_loss_pct} not in [1%, 10%]"
            )
        if self.risk_per_trade_pct > self.max_single_loss_pct:
            errors.append(
                f"risk_per_trade ({self.risk_per_trade_pct}) > "
                f"max_single_loss ({self.max_single_loss_pct})"
            )
        if self.risk_per_trade_pct * self.max_positions > self.max_portfolio_risk_pct * 2:
            errors.append(
                f"max_positions * risk_per_trade "
                f"({self.max_positions * self.risk_per_trade_pct:.2%}) "
                f"exceeds 2x max_portfolio_risk ({self.max_portfolio_risk_pct:.2%})"
            )
        if errors:
            raise ConfigValidationError(
                f"Risk config validation failed:\n" +
                "\n".join(f"  - {e}" for e in errors)
            )


@dataclass(frozen=True)
class EMAStrategyConfig:
    enabled: bool
    fast_period: int
    slow_period: int
    confirmation_bars: int
    min_spread_atr: float
    timeframe: str
    higher_tf: str

    def __post_init__(self):
        errors = []
        if self.fast_period >= self.slow_period:
            errors.append(
                f"fast_period ({self.fast_period}) must be < "
                f"slow_period ({self.slow_period})"
            )
        if not (1 <= self.fast_period <= 200):
            errors.append(f"fast_period={self.fast_period} not in [1, 200]")
        if not (1 <= self.slow_period <= 500):
            errors.append(f"slow_period={self.slow_period} not in [1, 500]")
        if self.confirmation_bars < 1:
            errors.append(f"confirmation_bars must be >= 1, got {self.confirmation_bars}")
        if self.min_spread_atr <= 0:
            errors.append(f"min_spread_atr must be > 0, got {self.min_spread_atr}")
        valid_timeframes = {"1min", "5min", "15min", "1h", "4h", "1d"}
        if self.timeframe not in valid_timeframes:
            errors.append(f"timeframe '{self.timeframe}' not in {valid_timeframes}")
        if self.higher_tf not in valid_timeframes:
            errors.append(f"higher_tf '{self.higher_tf}' not in {valid_timeframes}")
        if errors:
            raise ConfigValidationError(
                f"EMA strategy config validation failed:\n" +
                "\n".join(f"  - {e}" for e in errors)
            )


@dataclass(frozen=True)
class TrailingStopConfig:
    atr_multiplier: float
    min_distance_pct: float
    ratchet_only: bool

    def __post_init__(self):
        errors = []
        if not (0.5 <= self.atr_multiplier <= 10.0):
            errors.append(
                f"atr_multiplier={self.atr_multiplier} not in [0.5, 10.0]"
            )
        if not self.ratchet_only:
            errors.append(
                "ratchet_only MUST be true. "
                "A non-ratcheting stop moves backward and increases risk."
            )
        if errors:
            raise ConfigValidationError(
                f"Trailing stop config validation failed:\n" +
                "\n".join(f"  - {e}" for e in errors)
            )


@dataclass(frozen=True)
class VIXGateConfig:
    enabled: bool
    max_staleness_seconds: int
    min_vix: float
    max_vix: float
    halt_on_unavailable: bool

    def __post_init__(self):
        errors = []
        if self.enabled and not self.halt_on_unavailable:
            errors.append(
                "VIX gate is enabled but halt_on_unavailable is false. "
                "If VIX is unavailable, entries MUST be halted."
            )
        if self.min_vix >= self.max_vix:
            errors.append(
                f"min_vix ({self.min_vix}) >= max_vix ({self.max_vix})"
            )
        if self.max_staleness_seconds < 5:
            errors.append(
                f"max_staleness_seconds={self.max_staleness_seconds} too low (min 5)"
            )
        if errors:
            raise ConfigValidationError(
                f"VIX gate config validation failed:\n" +
                "\n".join(f"  - {e}" for e in errors)
            )


@dataclass(frozen=True)
class EnvironmentConfig:
    name: str
    broker_endpoint: str

    def __post_init__(self):
        errors = []
        if self.name not in ("paper", "live"):
            errors.append(f"environment name must be 'paper' or 'live', got '{self.name}'")
        # Cross-check: paper env must use paper endpoint
        if self.name == "paper" and "paper" not in self.broker_endpoint.lower():
            errors.append(
                f"Environment is 'paper' but broker endpoint "
                f"'{self.broker_endpoint}' does not contain 'paper'. "
                f"Are you pointing paper config at live broker?"
            )
        if self.name == "live" and "paper" in self.broker_endpoint.lower():
            errors.append(
                f"Environment is 'live' but broker endpoint "
                f"'{self.broker_endpoint}' contains 'paper'. "
                f"Are you pointing live config at paper broker?"
            )
        if errors:
            raise ConfigValidationError(
                f"Environment config validation failed:\n" +
                "\n".join(f"  - {e}" for e in errors)
            )


@dataclass(frozen=True)
class TradingConfig:
    """
    Top-level config. Immutable after construction.

    ALL settings come from here. Nothing is hardcoded in the trading logic.
    """
    environment: EnvironmentConfig
    risk: RiskConfig
    ema_strategy: EMAStrategyConfig
    trailing_stop: TrailingStopConfig
    vix_gate: VIXGateConfig

    @classmethod
    def load(cls, config_path: str) -> "TradingConfig":
        """
        Load, parse, and validate config from TOML file.

        If ANY validation fails, raises ConfigValidationError.
        The bot MUST NOT start with invalid config.
        """
        path = Path(config_path)
        if not path.exists():
            raise ConfigValidationError(f"Config file not found: {config_path}")

        with open(path, "rb") as f:
            raw = tomllib.load(f)

        try:
            config = cls(
                environment=EnvironmentConfig(**raw["environment"]),
                risk=RiskConfig(**raw["risk"]),
                ema_strategy=EMAStrategyConfig(**raw["strategy"]["ema_crossover"]),
                trailing_stop=TrailingStopConfig(**raw["trailing_stop"]),
                vix_gate=VIXGateConfig(**raw["vix_gate"]),
            )
        except KeyError as e:
            raise ConfigValidationError(f"Missing required config key: {e}")
        except TypeError as e:
            raise ConfigValidationError(f"Invalid config structure: {e}")

        return config
```

---

## Application Startup

```python
import os
import sys


def main():
    config_path = os.environ.get("TRADING_CONFIG", "config.paper.toml")

    try:
        config = TradingConfig.load(config_path)
    except ConfigValidationError as e:
        print(f"FATAL: Config validation failed. Bot will NOT start.\n{e}", file=sys.stderr)
        sys.exit(1)

    print(f"Config loaded: environment={config.environment.name}")
    print(f"  Risk per trade: {config.risk.risk_per_trade_pct:.2%}")
    print(f"  Max positions: {config.risk.max_positions}")
    print(f"  EMA strategy: {config.ema_strategy.fast_period}/{config.ema_strategy.slow_period}")

    # Now start the bot with validated config
    bot = TradingBot(config)
    bot.run()
```

---

## Hot Reload with Validation

Sometimes you need to change config without restarting. The rule:
**Validate completely before applying. Invalid config -> keep old config, alert.**

```python
import asyncio
from datetime import datetime, timezone


class ConfigManager:
    """
    Manages config with hot reload support.

    CRITICAL: New config is validated COMPLETELY before replacing old config.
    If validation fails, old config is kept and an alert is sent.
    """

    def __init__(self, config_path: str):
        self._config_path = config_path
        self._current: TradingConfig = TradingConfig.load(config_path)
        self._last_loaded: datetime = datetime.now(timezone.utc)
        self._reload_count: int = 0

    @property
    def config(self) -> TradingConfig:
        return self._current

    async def try_reload(self) -> tuple[bool, str]:
        """
        Attempt to reload config from file.

        Returns (success, message).
        On failure: keeps old config, returns error message.
        On success: replaces config atomically, returns success message.
        """
        try:
            new_config = TradingConfig.load(self._config_path)
        except ConfigValidationError as e:
            msg = f"Config reload REJECTED: validation failed.\n{e}\nKeeping previous config."
            await alert_warning(msg)
            return False, msg

        # Validate no dangerous changes
        danger_check = self._check_dangerous_changes(self._current, new_config)
        if danger_check:
            msg = f"Config reload REJECTED: dangerous change detected.\n{danger_check}"
            await alert_critical(msg)
            return False, msg

        # All checks passed -- swap atomically
        old_env = self._current.environment.name
        self._current = new_config
        self._last_loaded = datetime.now(timezone.utc)
        self._reload_count += 1

        msg = (
            f"Config reloaded successfully (reload #{self._reload_count}). "
            f"Environment: {new_config.environment.name}"
        )
        await alert_info(msg)
        return True, msg

    def _check_dangerous_changes(
        self, old: TradingConfig, new: TradingConfig
    ) -> Optional[str]:
        """Reject changes that could cause immediate harm."""
        # Cannot switch environments via hot reload
        if old.environment.name != new.environment.name:
            return (
                f"Cannot switch environment via hot reload: "
                f"{old.environment.name} -> {new.environment.name}. "
                f"Restart the bot to change environment."
            )
        # Cannot disable VIX gate via hot reload
        if old.vix_gate.enabled and not new.vix_gate.enabled:
            return "Cannot disable VIX gate via hot reload."
        # Cannot more than double risk per trade
        if new.risk.risk_per_trade_pct > old.risk.risk_per_trade_pct * 2:
            return (
                f"Risk per trade increase too large: "
                f"{old.risk.risk_per_trade_pct:.2%} -> {new.risk.risk_per_trade_pct:.2%}"
            )
        return None
```

---

## No Magic Numbers

Every constant in the codebase must come from config. No exceptions.

```python
# BAD: Magic numbers everywhere
def check_vix(vix_value):
    if vix_value > 80:      # Where did 80 come from?
        return "too_high"
    if vix_value < 9:       # Why 9?
        return "too_low"
    return "ok"

def calculate_stop(price, atr):
    return price - (2.0 * atr)   # Why 2.0?

MAX_POSITIONS = 5   # Hardcoded at module level -- invisible to config

# GOOD: Everything from config
def check_vix(vix_value: float, config: VIXGateConfig) -> str:
    if vix_value > config.max_vix:
        return "too_high"
    if vix_value < config.min_vix:
        return "too_low"
    return "ok"

def calculate_stop(price: float, atr: float, config: TrailingStopConfig) -> float:
    return price - (config.atr_multiplier * atr)

# max_positions comes from config.risk.max_positions -- no module-level constant
```

---

## Environment Separation

```
config/
    config.paper.toml    # Paper trading settings
    config.live.toml     # Live trading settings (higher thresholds)
```

### Critical Cross-Check

The environment name in the config MUST match the broker endpoint:

```python
# This check is in EnvironmentConfig.__post_init__ above, but to emphasize:
#
# config.paper.toml with broker_endpoint = "https://api.broker.com"  -> REJECTED
# config.live.toml with broker_endpoint = "https://paper-api.broker.com"  -> REJECTED
#
# This prevents the most catastrophic config error: running live logic
# against a paper endpoint (annoying) or paper logic against a live
# endpoint (potentially very expensive).
```

### Live vs Paper Differences

```toml
# config.paper.toml
[risk]
risk_per_trade_pct = 0.02    # More aggressive for testing
max_daily_loss_pct = 0.10    # Higher tolerance in paper

# config.live.toml
[risk]
risk_per_trade_pct = 0.01    # Conservative in live
max_daily_loss_pct = 0.03    # Tight daily loss limit in live
```

---

## Testing Config Validation

```python
import pytest


class TestConfigValidation:
    def test_valid_config_loads(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(VALID_CONFIG_TOML)
        config = TradingConfig.load(str(config_file))
        assert config.environment.name == "paper"

    def test_missing_required_field_fails(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(CONFIG_MISSING_RISK_TOML)
        with pytest.raises(ConfigValidationError, match="Missing required config key"):
            TradingConfig.load(str(config_file))

    def test_fast_ema_greater_than_slow_fails(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(CONFIG_BAD_EMA_TOML)  # fast=50, slow=20
        with pytest.raises(ConfigValidationError, match="fast_period.*must be <.*slow_period"):
            TradingConfig.load(str(config_file))

    def test_paper_env_with_live_endpoint_fails(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(CONFIG_PAPER_LIVE_ENDPOINT_TOML)
        with pytest.raises(ConfigValidationError, match="paper.*does not contain"):
            TradingConfig.load(str(config_file))

    def test_non_ratcheting_stop_fails(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(CONFIG_NO_RATCHET_TOML)
        with pytest.raises(ConfigValidationError, match="ratchet_only MUST be true"):
            TradingConfig.load(str(config_file))

    def test_risk_per_trade_exceeds_max(self, tmp_path):
        config_file = tmp_path / "config.toml"
        config_file.write_text(CONFIG_EXCESSIVE_RISK_TOML)  # risk_per_trade = 0.10
        with pytest.raises(ConfigValidationError, match="risk_per_trade_pct"):
            TradingConfig.load(str(config_file))


class TestHotReload:
    def test_valid_reload_succeeds(self):
        manager = ConfigManager("config.paper.toml")
        # Modify file with valid changes, then:
        success, msg = asyncio.run(manager.try_reload())
        assert success

    def test_invalid_reload_keeps_old_config(self):
        manager = ConfigManager("config.paper.toml")
        old_risk = manager.config.risk.risk_per_trade_pct
        # Write invalid config to file
        success, msg = asyncio.run(manager.try_reload())
        assert not success
        assert manager.config.risk.risk_per_trade_pct == old_risk

    def test_environment_switch_blocked(self):
        manager = ConfigManager("config.paper.toml")
        # Try to change environment to "live" via hot reload
        # Should be rejected
        success, msg = asyncio.run(manager.try_reload())
        assert not success
        assert "Cannot switch environment" in msg
```

---

## Emabot Config Drift Case Study

### What Happened

Emabot's configuration was spread across:
1. `config.yaml` -- main settings
2. `constants.py` -- hardcoded module-level constants
3. Environment variables -- some overrides
4. Command-line arguments -- more overrides
5. Database -- runtime-modified settings

When a developer changed `VIX_THRESHOLD` in `config.yaml` from 30 to 25, nothing
happened. Why? Because `constants.py` had `VIX_THRESHOLD = 30` which was imported
directly by the risk module. The config file value was loaded into a different variable
that was never read.

The developer spent hours debugging "why didn't my config change take effect?" before
discovering the shadowing constant.

### The Fix

1. Delete `constants.py` entirely
2. All values come from the validated config object
3. Config object is passed as dependency, never imported as module global
4. Startup validation catches missing fields

---

## Red Flags

Stop and fix immediately if you see ANY of these:

- **Settings in 3+ places.** Config file, constants module, env vars, CLI args, database.
  Pick ONE source of truth.
- **Magic numbers in code.** `if vix > 80`, `stop = price - 2.0 * atr`,
  `MAX_POSITIONS = 5`. All must come from config.
- **No startup validation.** Bot starts with any config, even nonsensical values.
  Invalid config should prevent startup.
- **`from constants import *`.** This is how config drift hides.
- **Different config formats per environment.** Paper uses YAML, live uses JSON,
  staging uses env vars. Use ONE format everywhere.
- **Mutable config object.** Config should be frozen/immutable after validation.
  Runtime changes go through the hot reload gate with full validation.

---

## Integration

- **trading-bot-skills:trading-bot-architecture** -- Config management is a core
  architectural concern. The config object flows through the entire system.
- **trading-bot-skills:risk-management-gates** -- Risk parameters are the most
  critical config values. They must be validated with extra strictness and
  cross-checked for consistency.
