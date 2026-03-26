---
name: market-sentiment-analyst
description: Use when processing news feeds for trading signals, analyzing social media sentiment, implementing NLP-based market mood indicators, or detecting institutional flow patterns
---

# Market Sentiment Analyst

## Iron Law

**SENTIMENT IS A SIGNAL INPUT, NEVER A SOLE DECISION DRIVER. IT MUST PASS THROUGH THE SAME VALIDATION GATES AS ANY OTHER SIGNAL.**

This skill fills the empty "Sentiment" indicator family referenced in strategy-signal-validation's confluence gates.

---

## Why Sentiment Matters (And Why It's Dangerous Alone)

Sentiment captures information not visible in price/volume: regulatory changes, executive departures, supply chain disruptions, coordinated retail interest. By the time these events hit the tape, the move is already underway.

But sentiment alone has approximately 52% accuracy -- barely above a coin flip. Combined with technical confluence (trend, momentum, volume families), accuracy rises to 65-70%. Sentiment proposes, technical confluence disposes.

### Case Study: The Twitter-Only Bot

A bot traded mid-cap equities based solely on Twitter sentiment. For two weeks it performed well. Then a coordinated pump-and-dump campaign hit three tracked tickers in the same week. The bot saw massive bullish sentiment spikes, entered aggressively, and rode the subsequent dumps down. Net result: -12% in one month. No technical filters, no manipulation detection, no sentiment decay model.

**Takeaway:** Sentiment is a signal input to the confluence gate, not a standalone strategy.

---

## News NLP Classification

Process headlines into bullish/bearish/neutral with a confidence score. Must handle negation ("not bullish" = bearish), sarcasm, and hedged language.

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Optional

class SentimentDirection(Enum):
    BULLISH = "bullish"
    BEARISH = "bearish"
    NEUTRAL = "neutral"

@dataclass
class SentimentScore:
    """Structured sentiment output for the confluence gate."""
    direction: SentimentDirection
    magnitude: float          # 0.0 to 1.0
    confidence: float         # 0.0 to 1.0
    source: str               # "news", "social", "fed", "institutional"
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    symbol: Optional[str] = None
    raw_text: Optional[str] = None

    @property
    def weighted_score(self) -> float:
        """Signed score: positive = bullish, negative = bearish."""
        if self.direction == SentimentDirection.NEUTRAL:
            return 0.0
        sign = 1.0 if self.direction == SentimentDirection.BULLISH else -1.0
        return sign * self.magnitude * self.confidence

BULLISH_KEYWORDS = {
    "upgrade": 0.7, "beat": 0.6, "raised guidance": 0.8,
    "buyback": 0.5, "record revenue": 0.7, "breakout": 0.5,
}
BEARISH_KEYWORDS = {
    "downgrade": 0.7, "missed": 0.6, "lowered guidance": 0.8,
    "layoffs": 0.5, "bankruptcy": 0.9, "SEC probe": 0.7, "fraud": 0.8,
}
NEGATION_WORDS = {"not", "no", "never", "neither", "barely", "hardly", "without"}

class NewsClassifier:
    """Keyword + negation-aware headline classifier."""
    def __init__(self, min_confidence: float = 0.3):
        self._min_confidence = min_confidence

    def classify(self, headline: str, symbol: Optional[str] = None) -> SentimentScore:
        tokens = headline.lower().split()
        text_lower = headline.lower()
        bull_score, bear_score = 0.0, 0.0

        for keyword, weight in BULLISH_KEYWORDS.items():
            if keyword in text_lower:
                if self._is_negated(tokens, keyword):
                    bear_score += weight * 0.8
                else:
                    bull_score += weight
        for keyword, weight in BEARISH_KEYWORDS.items():
            if keyword in text_lower:
                if self._is_negated(tokens, keyword):
                    bull_score += weight * 0.8
                else:
                    bear_score += weight

        total = bull_score + bear_score
        if total == 0:
            return SentimentScore(SentimentDirection.NEUTRAL, 0.0, 0.5, "news", symbol=symbol, raw_text=headline)

        magnitude = min(abs(bull_score - bear_score) / max(total, 1e-9), 1.0)
        confidence = min(total / 2.0, 1.0)
        if confidence < self._min_confidence:
            direction = SentimentDirection.NEUTRAL
        elif bull_score > bear_score:
            direction = SentimentDirection.BULLISH
        else:
            direction = SentimentDirection.BEARISH
        return SentimentScore(direction, magnitude, confidence, "news", symbol=symbol, raw_text=headline)

    def _is_negated(self, tokens: list[str], keyword: str) -> bool:
        first_kw_token = keyword.split()[0]
        for i, token in enumerate(tokens):
            if token == first_kw_token:
                preceding = tokens[max(0, i - 3):i]
                if any(neg in preceding for neg in NEGATION_WORDS):
                    return True
        return False
```

---

## Social Feed Processing

Social media is high volume, low signal. Rate limiting (respect API limits), deduplication (same news reposted 100x), and bot filtering (accounts < 30 days, suspicious posting patterns) are mandatory. Key principle: volume-weighted sentiment -- 1000 bearish tweets carry less weight than 1 institutional analyst downgrade.

```python
import hashlib
import time

@dataclass
class SocialPost:
    text: str
    author_id: str
    author_followers: int
    author_account_age_days: int
    timestamp: datetime
    platform: str             # "twitter", "reddit", "stocktwits"
    engagement: int = 0       # likes + retweets + replies

@dataclass
class BotFilter:
    """Filter likely bot accounts -- #1 source of sentiment manipulation."""
    min_account_age_days: int = 30
    min_followers: int = 10
    max_posts_per_minute: int = 5
    _post_counts: dict[str, list[float]] = field(default_factory=dict)

    def is_bot(self, post: SocialPost) -> bool:
        if post.author_account_age_days < self.min_account_age_days:
            return True
        if post.author_followers < self.min_followers:
            return True
        now = time.time()
        recent = [t for t in self._post_counts.get(post.author_id, []) if now - t < 60]
        recent.append(now)
        self._post_counts[post.author_id] = recent
        return len(recent) > self.max_posts_per_minute

class SocialFeedProcessor:
    """Process social feeds into credibility-weighted sentiment scores."""
    def __init__(self, bot_filter: BotFilter = None, classifier: NewsClassifier = None):
        self._bot_filter = bot_filter or BotFilter()
        self._classifier = classifier or NewsClassifier()
        self._seen_hashes: set[str] = set()

    def process_batch(self, posts: list[SocialPost], symbol: str) -> Optional[SentimentScore]:
        valid_scores: list[tuple[float, SentimentScore]] = []
        for post in posts:
            if self._bot_filter.is_bot(post):
                continue
            content_hash = hashlib.sha256(post.text.encode()).hexdigest()[:16]
            if content_hash in self._seen_hashes:
                continue
            self._seen_hashes.add(content_hash)
            score = self._classifier.classify(post.text, symbol=symbol)
            weight = min(post.author_followers / 10000, 5) + min(post.engagement / 100, 3)
            valid_scores.append((weight, score))

        if not valid_scores:
            return None
        total_weight = sum(w for w, _ in valid_scores)
        net = sum(w * s.weighted_score for w, s in valid_scores) / max(total_weight, 1e-9)
        direction = (SentimentDirection.BULLISH if net > 0.05
                     else SentimentDirection.BEARISH if net < -0.05
                     else SentimentDirection.NEUTRAL)
        return SentimentScore(direction, min(abs(net), 1.0), min(len(valid_scores) / 50, 1.0), "social", symbol=symbol)
```

---

## Fed Transcript Analysis

Hawkish/dovish keyword scoring. Key phrases: "persistent inflation" (hawkish), "labor market cooling" (dovish), "data-dependent" (neutral). FOMC dot plot interpretation and forward guidance parsing drive rate expectations.

```python
HAWKISH_PHRASES: dict[str, float] = {
    "persistent inflation": 0.8, "further tightening": 0.9, "restrictive stance": 0.7,
    "above target": 0.6, "wage pressures": 0.5, "overheating": 0.7, "rate increase": 0.8,
}
DOVISH_PHRASES: dict[str, float] = {
    "labor market cooling": 0.7, "rate cut": 0.8, "accommodative": 0.7,
    "downside risks": 0.6, "disinflation": 0.7, "economic slowdown": 0.6, "pause": 0.5,
}

@dataclass
class FedSentimentResult:
    hawk_score: float
    dove_score: float
    net_stance: float         # positive = hawkish, negative = dovish
    key_phrases: list[str]
    direction: SentimentDirection = SentimentDirection.NEUTRAL

    def __post_init__(self):
        if self.net_stance > 0.15:
            self.direction = SentimentDirection.BEARISH   # Hawkish = bearish for equities
        elif self.net_stance < -0.15:
            self.direction = SentimentDirection.BULLISH    # Dovish = bullish for equities

class FedTranscriptAnalyzer:
    """Analyze FOMC statements and map hawk/dove language to equity sentiment."""
    def __init__(self, hawkish: dict[str, float] = None, dovish: dict[str, float] = None):
        self._hawkish = hawkish or HAWKISH_PHRASES
        self._dovish = dovish or DOVISH_PHRASES

    def analyze(self, transcript: str) -> FedSentimentResult:
        text_lower = transcript.lower()
        hawk_total, dove_total, key_phrases = 0.0, 0.0, []
        for phrase, weight in self._hawkish.items():
            count = text_lower.count(phrase)
            if count > 0:
                hawk_total += weight * count
                key_phrases.append(f"HAWK: '{phrase}' x{count}")
        for phrase, weight in self._dovish.items():
            count = text_lower.count(phrase)
            if count > 0:
                dove_total += weight * count
                key_phrases.append(f"DOVE: '{phrase}' x{count}")
        total = hawk_total + dove_total
        if total == 0:
            return FedSentimentResult(0.0, 0.0, 0.0, [])
        return FedSentimentResult(hawk_total / total, dove_total / total, (hawk_total - dove_total) / total, key_phrases)

    def to_sentiment_score(self, result: FedSentimentResult) -> SentimentScore:
        return SentimentScore(result.direction, min(abs(result.net_stance), 1.0), min((result.hawk_score + result.dove_score) * 2, 1.0), "fed")
```

---

## Institutional Flow Detection

Unusual options activity (put/call ratio spikes), dark pool prints (large blocks at VWAP), sector rotation signals, and 13F filing analysis. Institutional signals carry higher confidence because institutions risk real capital in size.

```python
@dataclass
class OptionsFlowSignal:
    symbol: str
    put_call_ratio: float
    historical_avg_pc_ratio: float
    total_premium: float
    largest_single_trade: float
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

@dataclass
class DarkPoolPrint:
    symbol: str
    size: int
    price: float
    vwap: float
    avg_daily_volume: int
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

class InstitutionalFlowDetector:
    """Detect anomalous institutional flow patterns."""
    def __init__(self, pc_ratio_std_threshold: float = 2.0, dark_pool_size_threshold: float = 0.05):
        self._pc_threshold = pc_ratio_std_threshold
        self._dp_threshold = dark_pool_size_threshold

    def analyze_options_flow(self, signal: OptionsFlowSignal) -> Optional[SentimentScore]:
        ratio_deviation = abs(signal.put_call_ratio - signal.historical_avg_pc_ratio)
        if ratio_deviation < self._pc_threshold * 0.1:
            return None
        if signal.put_call_ratio > signal.historical_avg_pc_ratio * 1.5:
            direction = SentimentDirection.BEARISH
        elif signal.put_call_ratio < signal.historical_avg_pc_ratio * 0.6:
            direction = SentimentDirection.BULLISH
        else:
            return None
        return SentimentScore(direction, min(ratio_deviation / signal.historical_avg_pc_ratio, 1.0), 0.7, "institutional", symbol=signal.symbol)

    def analyze_dark_pool(self, print_: DarkPoolPrint) -> Optional[SentimentScore]:
        size_ratio = print_.size / max(print_.avg_daily_volume, 1)
        if size_ratio < self._dp_threshold:
            return None
        price_vs_vwap = (print_.price - print_.vwap) / max(print_.vwap, 1e-9)
        if price_vs_vwap > 0.001:
            direction = SentimentDirection.BULLISH
        elif price_vs_vwap < -0.001:
            direction = SentimentDirection.BEARISH
        else:
            direction = SentimentDirection.NEUTRAL
        return SentimentScore(direction, min(size_ratio / 0.1, 1.0), 0.8, "institutional", symbol=print_.symbol)
```

---

## Sentiment Decay Half-Life

Sentiment is perishable. Social media sentiment: 15-30 minute half-life. News sentiment: 2-4 hour half-life. Institutional flow signals: 1-3 day half-life. Stale sentiment is worse than no sentiment because it creates false confidence in outdated information.

```python
import math

HALF_LIVES: dict[str, float] = {
    "social": 1200.0,         # 20 minutes
    "news": 10800.0,          # 3 hours
    "fed": 43200.0,           # 12 hours
    "institutional": 129600.0, # 1.5 days
}

@dataclass
class SentimentDecayModel:
    """Exponential decay: weight = 0.5 ^ (age_seconds / half_life)."""
    half_lives: dict[str, float] = None
    min_weight: float = 0.1

    def __post_init__(self):
        if self.half_lives is None:
            self.half_lives = HALF_LIVES.copy()

    def decay_weight(self, score: SentimentScore) -> float:
        age_seconds = (datetime.now(timezone.utc) - score.timestamp).total_seconds()
        if age_seconds < 0:
            return 0.0
        half_life = self.half_lives.get(score.source, 3600.0)
        weight = math.pow(0.5, age_seconds / half_life)
        return weight if weight >= self.min_weight else 0.0

    def apply_decay(self, score: SentimentScore) -> SentimentScore:
        weight = self.decay_weight(score)
        if weight == 0.0:
            return SentimentScore(SentimentDirection.NEUTRAL, 0.0, 0.0, score.source, timestamp=score.timestamp, symbol=score.symbol)
        return SentimentScore(score.direction, score.magnitude * weight, score.confidence * weight, score.source, timestamp=score.timestamp, symbol=score.symbol)

    def is_expired(self, score: SentimentScore) -> bool:
        return self.decay_weight(score) == 0.0
```

---

## Anti-Manipulation Detection

Coordinated pump-and-dump pattern: sudden volume spike in social mentions + low market cap + no fundamental news = likely manipulation. This is the single biggest threat to sentiment-based trading.

```python
@dataclass
class ManipulationIndicators:
    mention_spike_ratio: float    # Current rate vs 7-day average
    unique_author_ratio: float    # Unique authors / total mentions (low = coordinated)
    avg_account_age_days: float
    market_cap_usd: float
    has_fundamental_news: bool
    price_volume_divergence: bool

@dataclass
class ManipulationResult:
    is_suspicious: bool
    score: float              # 0.0 to 1.0
    reasons: list[str]
    recommendation: str       # "proceed", "reduce_weight", "reject"

class ManipulationDetector:
    """
    Scoring heuristics (each contributes to a 0-1 score):
    - Mention spike > 5x average: +0.25
    - Low unique author ratio (< 0.3): +0.25
    - Low avg account age (< 60 days): +0.20
    - Low market cap (< $500M): +0.15
    - No fundamental news catalyst: +0.15
    """
    def __init__(self, mention_spike_threshold: float = 5.0, unique_author_floor: float = 0.3,
                 account_age_floor_days: float = 60.0, market_cap_floor_usd: float = 500_000_000.0):
        self._mention_threshold = mention_spike_threshold
        self._author_floor = unique_author_floor
        self._age_floor = account_age_floor_days
        self._cap_floor = market_cap_floor_usd

    def evaluate(self, indicators: ManipulationIndicators) -> ManipulationResult:
        score, reasons = 0.0, []
        if indicators.mention_spike_ratio > self._mention_threshold:
            score += 0.25
            reasons.append(f"Mention spike {indicators.mention_spike_ratio:.1f}x (threshold: {self._mention_threshold:.1f}x)")
        if indicators.unique_author_ratio < self._author_floor:
            score += 0.25
            reasons.append(f"Low unique author ratio {indicators.unique_author_ratio:.2f} (floor: {self._author_floor:.2f})")
        if indicators.avg_account_age_days < self._age_floor:
            score += 0.20
            reasons.append(f"Young accounts avg {indicators.avg_account_age_days:.0f}d (floor: {self._age_floor:.0f}d)")
        if indicators.market_cap_usd < self._cap_floor:
            score += 0.15
            reasons.append(f"Low market cap ${indicators.market_cap_usd:,.0f} (floor: ${self._cap_floor:,.0f})")
        if not indicators.has_fundamental_news:
            score += 0.15
            reasons.append("No fundamental news catalyst")
        recommendation = "reject" if score >= 0.6 else "reduce_weight" if score >= 0.35 else "proceed"
        return ManipulationResult(is_suspicious=score >= 0.35, score=score, reasons=reasons, recommendation=recommendation)
```

---

## Red Flags

| Red Flag | Why It Matters |
|---|---|
| Sentiment score used as sole entry trigger | 52% accuracy is a coin flip. Must pass confluence gate with 3+ families |
| No bot filtering on social feeds | Bot networks flip sentiment in minutes, triggering false entries |
| No sentiment decay model | Stale sentiment creates false confidence. A 2-hour-old tweet is priced in |
| No manipulation detection | Pump-and-dump campaigns specifically target sentiment-based bots |
| All social posts weighted equally | 1 institutional analyst > 10,000 anonymous tweets. Weight by credibility |
| No deduplication on social feeds | Same news reposted 100x inflates sentiment scores artificially |
| Fed transcript treated as bullish/bearish directly | Hawkish Fed = bearish equities. The mapping must be explicit |
| Sentiment confidence score ignored | Low-confidence sentiment must be discounted or excluded |
| No rate limiting on feed ingestion | API rate limits will get your keys revoked |
| Social sentiment half-life > 1 hour | Social moves are priced in within 15-30 minutes |

---

## Integration

- **signal-source-integration** -- Sentiment feeds (news APIs, social APIs, Fed transcripts) are external signal sources. They must pass through the same authentication, rate-limiting, and normalization pipeline as any other external signal before reaching the sentiment classifiers.
- **confidence-thresholds** -- Sentiment scores carry a confidence value that feeds into the overall signal confidence calculation. Low-confidence sentiment (few data points, ambiguous language) is weighted down or excluded from confluence.
- **strategy-signal-validation** -- Sentiment fills the "Sentiment" family in the confluence gate's indicator family classification. A bullish sentiment score satisfies one of the required 3+ non-correlated indicator families, but it cannot substitute for trend, momentum, or volume confirmation.
- **risk-management-gates** -- When manipulation is detected (score >= 0.35), the risk gate must reduce position size or reject the trade entirely. Manipulation detection output feeds directly into the pre-trade risk check pipeline as an additional gate.
