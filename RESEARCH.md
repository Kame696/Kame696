# KAME API Rotation — Master Research & Roadmap

> Living document. Grows with the project. Last updated: 2026-10-09.
> English only. Written for a professional development context.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
3. [Rotation Logic — Performance Layer](#3-rotation-logic--performance-layer)
4. [Network Identity Architecture](#4-network-identity-architecture)
5. [Credential Pool Strategy](#5-credential-pool-strategy)
6. [Provider Intelligence — Quota Models](#6-provider-intelligence--quota-models)
7. [Request Optimization & Caching](#7-request-optimization--caching)
8. [Agent-Native Integration (The 50/50 Architecture)](#8-agent-native-integration-the-5050-architecture)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Open Research Items](#10-open-research-items)

---

## 1. Executive Summary

KAME API Rotation solves one concrete problem: **a failing key should never end a turn.** The current implementation achieves this with a fail-forward rotation model across a pool of credentials.

The research documented here identifies the next generation of improvements — organized into eight distinct improvement vectors. Each vector is independently implementable. Taken together, they transform KAME from a resilience tool into a **throughput multiplier** that allows an AI agent to treat the aggregate of all free-tier providers as a single, effectively unlimited resource.

The central thesis: **14 API keys used naively may deliver no more throughput than 2-3, because they share identity signals at the network and account level.** True isolation — network, fingerprint, and credential — is what converts 14 keys into 14× throughput.

A secondary thesis: **the agent itself should be a participant in rotation decisions, not a passive consumer.** A rotation layer that receives priority signals and token estimates from the agent upstream of each call can make dramatically better routing decisions than one operating blind.

---

## 2. Current Architecture Analysis

### What is known from public documentation

| Property | Value |
|---|---|
| Current version | 1.8.1.8 (both ports) |
| Supported hosts | Agent Zero, Hermes |
| Core mechanism | Fail-forward key rotation |
| Provider allowlist | None — evidence-based routing only |
| Test coverage | 24 offline suites + remote native checks (Agent Zero); 3,132–3,136 tests per job (Hermes) |

### Core mechanism

```
[agent call]
    → [rotation layer picks key from pool]
    → [API call with selected key]
    → [success] → return response
    → [failure: 429/503/etc.] → mark key, pick next → retry
```

### Identified limitations

1. **Reactive only** — rotation fires after failure, not before
2. **Binary key state** — keys are either "good" or "failed"; no health gradient
3. **No quota-type awareness** — RPM, RPD, and TPM treated identically
4. **No network identity separation** — all keys likely share source IP and TLS fingerprint
5. **No request deduplication or caching** — every call hits the API
6. **Agent-opaque** — the agent has no channel to signal priority or token estimates
7. **503 treated as 429** — transient server overload handled same as quota exhaustion

---

## 3. Rotation Logic — Performance Layer

### 3.1 Health-Scored Predictive Routing

**Current behavior:** wait for failure, then rotate.  
**Target behavior:** score every key continuously; route to the highest-scoring key before a failure occurs.

Each key maintains a live health record:

```python
@dataclass
class KeyHealth:
    key_id: str
    last_used_ts: float           # unix timestamp
    error_window_count: int       # errors in last 60s
    avg_latency_ms: float         # rolling average
    consecutive_5xx: int          # consecutive server errors
    rpm_used: int                 # requests in current minute window
    rpm_limit: int                # known RPM ceiling
    rpd_used: int                 # requests today
    rpd_limit: int                # known RPD ceiling
    tpm_used: int                 # tokens in current minute
    tpm_limit: int                # known TPM ceiling
    last_quota_type_hit: str      # 'rpm' | 'rpd' | 'tpm' | None
    estimated_recovery_ts: float  # when quota window reopens
```

Scoring function:

```python
def health_score(k: KeyHealth, prompt_tokens: int = 0) -> float:
    now = time.time()

    # if in cooldown, score is zero
    if now < k.estimated_recovery_ts:
        return 0.0

    # staleness bonus — keys recover quota over time
    staleness = min((now - k.last_used_ts) / 60.0, 1.0)

    # error penalty with decay
    error_penalty = k.error_window_count * 0.3

    # latency penalty
    latency_penalty = k.avg_latency_ms / 5000.0

    # quota headroom (fraction remaining)
    rpm_headroom = max(0, (k.rpm_limit - k.rpm_used) / max(k.rpm_limit, 1))
    tpm_headroom = max(0, (k.tpm_limit - (k.tpm_used + prompt_tokens)) / max(k.tpm_limit, 1))

    return staleness + rpm_headroom * 0.4 + tpm_headroom * 0.4 - error_penalty - latency_penalty
```

### 3.2 Error Code Differentiation

Not all errors mean the same thing. Treating them uniformly wastes rotations.

| HTTP Code | Meaning | Correct Action |
|---|---|---|
| 429 | Quota exhausted (RPM/RPD/TPM) | Mark key cooldown, rotate immediately |
| 503 | Server overload (transient) | Retry same key 1–2× with 200–500ms backoff; rotate only if persists |
| 529 | Anthropic-specific overload | Same as 503 |
| 500 | Server error | Single retry; if repeats, rotate |
| 401/403 | Auth failure | Key is invalid/revoked; remove from pool permanently |
| 400 | Bad request | Do not rotate; error is in the payload, not the key |

The critical distinction: **a 503 on key A does not mean key B is better.** Both keys hit the same overloaded server. Rotating on every 503 burns the key rotation budget for zero throughput gain.

```python
def handle_error(response_code: int, key: KeyHealth, attempt: int) -> Action:
    if response_code in (401, 403):
        return Action.INVALIDATE_KEY
    if response_code == 429:
        return Action.ROTATE_IMMEDIATELY
    if response_code in (500, 503, 529):
        if attempt < 2:
            return Action.RETRY_SAME_KEY_WITH_BACKOFF
        return Action.ROTATE
    if response_code == 400:
        return Action.FAIL_REQUEST  # not a key issue
    return Action.ROTATE
```

### 3.3 Parallel Key Racing

For interactive, latency-sensitive calls: send to the top N keys simultaneously, use the first response, cancel the rest.

```python
async def race_keys(prompt: str, keys: list[KeyHealth], n: int = 2) -> Response:
    top_keys = sorted(keys, key=health_score, reverse=True)[:n]
    tasks = {asyncio.create_task(call_api(k, prompt)): k for k in top_keys}
    
    done, pending = await asyncio.wait(
        tasks.keys(), 
        return_when=asyncio.FIRST_COMPLETED
    )
    
    for task in pending:
        task.cancel()
    
    winning_task = done.pop()
    losing_keys = [tasks[t] for t in pending]
    
    # cancel the in-flight requests to avoid wasting quota
    for k in losing_keys:
        await cancel_inflight_request(k)
    
    return winning_task.result()
```

**Trade-off:** burns quota on losing keys. Use exclusively for interactive calls where P99 latency matters. Batch/background calls use single-key routing.

### 3.4 Quota-Type Awareness

RPM, RPD, and TPM are independent clocks with different reset periods.

| Quota type | Resets every | On exhaustion |
|---|---|---|
| RPM | 60 seconds | Mark key unavailable for ~60s |
| RPD | 24 hours (midnight Pacific for Google) | Mark key unavailable until next day |
| TPM | 60 seconds | Mark key unavailable for ~60s; disproportionately affected by long context |

A key that hits RPM is recoverable in 60 seconds. Do not discard it from the pool — schedule recovery. A key that hits RPD is gone for the day.

Token-aware routing: route long-context requests to keys with high TPM headroom.

```python
def route_by_token_budget(prompt_tokens: int, keys: list[KeyHealth]) -> KeyHealth:
    if prompt_tokens > 8000:
        # long context: prioritize TPM headroom
        viable = [k for k in keys if k.tpm_limit - k.tpm_used > prompt_tokens * 1.2]
        return max(viable, key=lambda k: k.tpm_limit - k.tpm_used) if viable else None
    return max(keys, key=lambda k: health_score(k, prompt_tokens))
```

---

## 4. Network Identity Architecture

### 4.1 Why Multiple Accounts on the Same IP Underperform

API providers — particularly Google (Gemini), Anthropic, and OpenAI — implement multi-signal identity clustering. When multiple accounts share the same originating IP, they are recognized as belonging to the same organizational origin, and their quotas may be evaluated in aggregate or individually but with reduced total capacity.

**Identity signals observed or documented:**

| Signal | Detection method | Independence strategy |
|---|---|---|
| Source IP address | Direct header inspection | Dedicated egress IP per credential |
| TLS client fingerprint (JA3/JA4) | TLS handshake analysis | Per-credential TLS profile diversification |
| HTTP/2 SETTINGS frames | Connection-level fingerprint | Per-credential HTTP/2 configuration |
| User-Agent string | Request header | Per-credential UA rotation (stable per key) |
| Account creation origin IP | Registration-time record | Provision each account from a distinct network origin |
| Request timing distribution | Statistical analysis | Organic cadence simulation (see §4.4) |

### 4.2 Dedicated Network Origin Assignment

**Architecture:** each credential in the pool is assigned a static, dedicated egress IP. The credential always exits through its assigned IP regardless of which machine originates the request.

```
credential_pool:
  key_01:
    api_key: "..."
    proxy_assignment: "residential_us_east_01"
    tls_profile: "chrome_120"
    user_agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..."
  key_02:
    api_key: "..."
    proxy_assignment: "residential_eu_west_03"
    tls_profile: "firefox_121"
    user_agent: "Mozilla/5.0 (X11; Linux x86_64; rv:121.0) Gecko/20100101..."
  ...
```

**Proxy type selection:**

| Proxy type | Detection risk | Cost | Recommendation |
|---|---|---|---|
| Data center (AWS/GCP/Azure IP ranges) | High — well-known ranges | Low | Avoid for primary keys |
| Shared residential | Medium | Low | Acceptable for lower-priority keys |
| Dedicated residential | Low | Medium (~$3–8/IP/month) | Recommended for primary pool |
| ISP/static residential | Very low | Higher | Optimal for long-term stability |
| Mobile (4G/5G rotating) | Very low | Variable | Useful for account provisioning |

**Implementation via `curl_cffi` (recommended over `requests`/`aiohttp`):**

```python
from curl_cffi import requests as cffi_requests

TLS_PROFILES = [
    "chrome120", "chrome119", "firefox121", "safari17_0",
    "chrome118", "firefox120", "edge120"
]

class CredentialClient:
    def __init__(self, api_key: str, proxy: str, tls_profile: str):
        self.api_key = api_key
        self.session = cffi_requests.Session(impersonate=tls_profile)
        self.session.proxies = {"https": proxy, "http": proxy}
    
    async def call(self, payload: dict) -> dict:
        return self.session.post(
            ENDPOINT,
            json=payload,
            headers={"Authorization": f"Bearer {self.api_key}"}
        ).json()
```

### 4.3 TLS Fingerprint Diversification

The TLS Client Hello message contains ordered cipher suites, extension list, and GREASE values that form a stable fingerprint (JA3/JA4 hash). Python's standard `requests` and `aiohttp` produce identical JA3 hashes — a fleet of 14 API clients using these libraries presents a single TLS identity regardless of IP diversity.

**`curl_cffi` solves this** by reproducing the exact TLS Client Hello of specific browser versions, including correct cipher ordering, extension presence, and GREASE randomization.

Assign a distinct `impersonate` target to each credential so the TLS fingerprint pool matches the IP pool in diversity.

### 4.4 Organic Request Cadence

Inter-request timing analysis can distinguish automated clients from organic API usage. A flat request interval (no jitter) is the clearest automation signal. The jitter implementation you already have is the correct response — but the implementation details matter.

**Naïve jitter** (uniform random delay): detectable — the distribution is too flat.

**Organic cadence simulation**: model actual developer API usage patterns.

```python
import random
import numpy as np

def organic_delay(base_ms: int = 0) -> float:
    """
    Simulate organic inter-request timing.
    Models: fast burst (agent tool calls), medium pause (thinking),
    occasional long pause (user reading output).
    """
    profile = random.choices(
        ['burst', 'normal', 'pause'],
        weights=[0.6, 0.3, 0.1]
    )[0]
    
    if profile == 'burst':
        # fast consecutive calls — agent tool chaining
        return base_ms + np.random.exponential(scale=150)
    elif profile == 'normal':
        # normal inter-call gap
        return base_ms + np.random.normal(loc=800, scale=200)
    else:
        # longer pause — simulates user reading
        return base_ms + np.random.normal(loc=3000, scale=500)
```

---

## 5. Credential Pool Strategy

### 5.1 Independent Credential Provisioning

For maximum quota independence, each credential in the pool should be provisioned under conditions that establish it as an independent organizational unit from the provider's perspective.

**Provisioning independence checklist:**

| Factor | Target | Notes |
|---|---|---|
| Email address | Unique per credential | Catch-all domain or dedicated mailbox per account |
| Provisioning IP | Unique per credential | Mobile/residential proxy at account creation time |
| SMS verification | Unique phone number per account | VoIP providers with programmable numbers |
| Browser fingerprint at signup | Unique per credential | Dedicated browser profile, distinct UA and TLS |
| Provisioning date distribution | Staggered over days/weeks | Avoid cluster signatures from batch creation |
| Geographic region | Varied | Distributes accounts across provider regions |

**Credential lifecycle management:**

```
[provision] → [verify] → [activate] → [pool entry] → [active rotation]
                                                              ↓
                                                    [health monitoring]
                                                              ↓
                                              [degraded] → [cooldown] → [recovery]
                                                              ↓
                                                    [expired/banned] → [replacement]
```

### 5.2 Credential Pool Sizing

Calculating optimal pool size for a target throughput:

```
target_rpm: desired requests per minute (e.g., 60)
provider_rpm_per_key: provider's RPM limit per account (e.g., 15 for Gemini free tier)
safety_factor: 1.3 (30% buffer for rotation overhead and cooldown overlap)

minimum_pool_size = ceil((target_rpm / provider_rpm_per_key) * safety_factor)

# Example: 60 RPM target, 15 RPM/key
minimum_pool_size = ceil((60 / 15) * 1.3) = ceil(5.2) = 6 keys
```

For effectively unlimited throughput (agent use cases where calls arrive in bursts):

```
burst_size: maximum concurrent calls in a burst (e.g., 10)
pool_size: burst_size * 3  # ensures at minimum 1 healthy key per burst slot after overhead
```

### 5.3 Pool Segmentation by Call Type

Segment the pool to prevent high-volume batch operations from exhausting keys needed for interactive calls.

```python
class SegmentedPool:
    def __init__(self, keys: list[CredentialClient]):
        # 60% of pool for interactive calls (racing enabled)
        # 30% for batch/background operations
        # 10% reserved (emergency overflow)
        n = len(keys)
        self.interactive = keys[:int(n * 0.6)]
        self.batch = keys[int(n * 0.6):int(n * 0.9)]
        self.reserve = keys[int(n * 0.9):]
    
    def get_key(self, call_type: str = 'interactive') -> CredentialClient:
        pool = {
            'interactive': self.interactive,
            'batch': self.batch,
            'reserve': self.reserve
        }[call_type]
        return max(pool, key=lambda k: health_score(k))
```

---

## 6. Provider Intelligence — Quota Models

### 6.1 Google Gemini

**Free tier limits (2025, subject to change):**

| Model | RPM | RPD | TPM |
|---|---|---|---|
| Gemini 2.0 Flash | 15 | 1,500 | 1,000,000 |
| Gemini 1.5 Flash | 15 | 1,500 | 1,000,000 |
| Gemini 1.5 Pro | 2 | 50 | 32,000 |

**Quota reset:** RPM resets every 60 seconds; RPD resets at midnight Pacific Time.

**Key insight — dual quota pools per account:**

Google accounts can access Gemini through two independent endpoints with separate quota accounting:

| Endpoint | Access | Quota pool |
|---|---|---|
| `generativelanguage.googleapis.com` | Gemini API key | Free tier quota |
| `us-central1-aiplatform.googleapis.com` | Vertex AI | Separate quota (free tier on Vertex has different limits) |
| `europe-west4-aiplatform.googleapis.com` | Vertex AI EU | Regional capacity — may differ from US during peak |

A single Google account can be configured for both access paths. During Gemini API quota exhaustion, the rotation layer can fall back to Vertex AI endpoints without switching credentials. This effectively doubles the available quota per account.

**Regional capacity:** Gemini 503 errors are often region-specific. `us-central1` capacity saturation does not necessarily imply `europe-west4` saturation. A rotation layer aware of this can include regional fallback as a dimension of its routing logic.

### 6.2 Anthropic (Claude)

**Free tier:** not available as of 2025 (requires paid credits).  
**Relevant for:** rotation logic applied to paid keys across multiple accounts or team seats.

**Rate limit structure:** Tier-based (Tier 1–4), scales with spend history.

| Tier | RPM | TPM | Approximate spend threshold |
|---|---|---|---|
| Tier 1 | 50 | 50,000 | $0+ (new accounts) |
| Tier 2 | 1,000 | 100,000 | $100 |
| Tier 3 | 2,000 | 200,000 | $500 |
| Tier 4 | 4,000 | 400,000 | $5,000 |

**Rotation benefit:** multiple Tier 1 accounts in rotation can collectively match Tier 3+ throughput at zero incremental spend beyond the minimum top-up required to activate each account.

**Key signal:** Anthropic returns `retry-after` in the 429 response header. The rotation layer should parse this value and schedule the key's recovery at `now + retry_after` rather than using a fixed cooldown estimate.

### 6.3 OpenAI

**Free tier:** $5 credit on new accounts (historically; may vary by region).

**Tier structure:** similar to Anthropic — increases with verified spend.

**Key signal:** `x-ratelimit-remaining-requests` and `x-ratelimit-remaining-tokens` are returned in every response header. A rotation layer that reads these values can perform proactive rotation before hitting zero rather than waiting for a 429.

```python
def update_key_from_response_headers(key: KeyHealth, headers: dict):
    rpm_remaining = int(headers.get('x-ratelimit-remaining-requests', key.rpm_limit))
    tpm_remaining = int(headers.get('x-ratelimit-remaining-tokens', key.tpm_limit))
    reset_requests = headers.get('x-ratelimit-reset-requests', '60s')
    reset_tokens = headers.get('x-ratelimit-reset-tokens', '60s')
    
    key.rpm_used = key.rpm_limit - rpm_remaining
    key.tpm_used = key.tpm_limit - tpm_remaining
    
    # parse reset times and schedule recovery
    if rpm_remaining < 3:
        key.estimated_recovery_ts = now() + parse_reset_duration(reset_requests)
```

### 6.4 Other Free-Tier Providers Worth Including in Rotation

| Provider | Free tier | Notable limits | API compatibility |
|---|---|---|---|
| Google AI Studio (Gemini) | Yes — see above | See §6.1 | Native |
| Groq | Yes — generous RPM | Fast inference (LPU), low latency | OpenAI-compatible |
| Together AI | $1 free credit | Many open models | OpenAI-compatible |
| Mistral AI | Free tier on small models | Mistral-7B family | Native / OpenAI-compatible |
| Cohere | Free tier (trial) | Command-R family | Native |
| Hugging Face Inference API | Free (rate limited) | Many open models | Native |
| Cerebras | Free tier | Very fast inference | OpenAI-compatible |
| SambaNova | Free tier | Fast inference | OpenAI-compatible |

**Strategic insight:** because several of these providers expose OpenAI-compatible endpoints, a rotation layer with a single OpenAI-compatible client can treat them as interchangeable key sources. The pool is not just "14 Gemini keys" — it becomes "14 Gemini + 10 Groq + 8 Mistral + ..." without per-provider client code.

---

## 7. Request Optimization & Caching

### 7.1 Exact Request Deduplication

Within a short time window, identical requests (same model, same messages, same parameters) should return cached responses rather than consuming quota.

```python
import hashlib
import time

class RequestCache:
    def __init__(self, ttl_seconds: int = 300):
        self._cache: dict[str, tuple[dict, float]] = {}
        self.ttl = ttl_seconds
    
    def _key(self, model: str, messages: list, params: dict) -> str:
        payload = json.dumps({"model": model, "messages": messages, **params}, sort_keys=True)
        return hashlib.sha256(payload.encode()).hexdigest()
    
    def get(self, model, messages, params) -> dict | None:
        k = self._key(model, messages, params)
        if k in self._cache:
            response, ts = self._cache[k]
            if time.time() - ts < self.ttl:
                return response
        return None
    
    def set(self, model, messages, params, response):
        k = self._key(model, messages, params)
        self._cache[k] = (response, time.time())
```

### 7.2 Semantic Caching

For agent workloads where the same question is phrased differently across turns, semantic similarity matching avoids redundant API calls.

A lightweight local embedding model (`all-MiniLM-L6-v2`, ~80MB) runs entirely offline and adds no latency overhead beyond the embedding computation.

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticCache:
    def __init__(self, model_name: str = 'all-MiniLM-L6-v2', threshold: float = 0.94):
        self.model = SentenceTransformer(model_name)
        self.threshold = threshold
        self.entries: list[tuple[np.ndarray, str, dict]] = []  # (embedding, prompt, response)
    
    def lookup(self, prompt: str) -> dict | None:
        if not self.entries:
            return None
        embedding = self.model.encode(prompt)
        for cached_embedding, cached_prompt, cached_response in self.entries:
            similarity = np.dot(embedding, cached_embedding) / (
                np.linalg.norm(embedding) * np.linalg.norm(cached_embedding)
            )
            if similarity >= self.threshold:
                return cached_response
        return None
    
    def store(self, prompt: str, response: dict):
        embedding = self.model.encode(prompt)
        self.entries.append((embedding, prompt, response))
```

**Expected cache hit rate:** 20–40% on repetitive agent workloads (tool descriptions, system prompts, repeated sub-queries).

### 7.3 Request Batching

Some providers support batching multiple prompts into a single API call. Where available:

- Reduces per-request overhead (connection, auth, HTTP round-trip)
- Consolidates quota consumption into fewer high-value calls
- Enables the rotation layer to pack multiple agent sub-calls into a single API unit

### 7.4 Streaming Response Optimization

Streaming responses release per-minute quota faster than waiting for full completion. A 1000-token response in streaming mode begins returning tokens within ~100ms and releases the RPM slot earlier than a non-streaming call that blocks for 2-3 seconds.

For rotation purposes: **prefer streaming for interactive calls, non-streaming for cached/batch.**

---

## 8. Agent-Native Integration (The 50/50 Architecture)

### 8.1 The Core Insight

Current state: the rotation layer is a transparent proxy. The agent calls it the same way it calls an API directly. The rotation layer has no information about:

- Whether the call is interactive or batch
- How many tokens the prompt contains (before encoding)
- Whether the agent would prefer speed vs. reliability
- Whether the agent is in a retry loop (redundant rotation)

Target state: **the agent is a first-class participant in routing decisions.**

```
agent
  → [pre-call signal: priority=interactive, estimated_tokens=2400, can_race=true]
  → [rotation layer routes to fastest available key, optionally races 2]
  → [response + routing metadata: key_used, latency_ms, quota_remaining]
  → agent uses routing metadata for next call decisions
```

### 8.2 Agent-to-Rotation Signal Interface

```python
@dataclass
class RoutingHint:
    priority: str = 'normal'       # 'interactive' | 'normal' | 'batch' | 'background'
    estimated_tokens: int = 0      # pre-estimate of prompt tokens
    allow_racing: bool = False     # permit parallel key racing for this call
    retry_context: bool = False    # agent is already in a retry loop
    idempotent: bool = True        # safe to deduplicate/cache
    preferred_provider: str = None # optional provider preference
    max_latency_ms: int = None     # timeout hint

@dataclass 
class RoutingResult:
    key_used: str
    provider: str
    latency_ms: float
    quota_remaining_rpm: int
    quota_remaining_rpd: int
    was_cached: bool
    was_raced: bool
    fallback_count: int
```

### 8.3 The 50/50 Model

```
┌─────────────────────────────────────────────────────┐
│                   AI AGENT                          │
│                                                     │
│  "I need to call an LLM."                          │
│                                                     │
│  [produces RoutingHint]                             │
└──────────────────────┬──────────────────────────────┘
                       │ hint + payload
                       ▼
┌─────────────────────────────────────────────────────┐
│              KAME ROTATION LAYER                    │
│                                                     │
│  ┌─────────────────┐    ┌─────────────────────┐    │
│  │  Credential     │    │  Semantic Cache      │    │
│  │  Pool           │    │  (hit → skip API)    │    │
│  │                 │    └─────────────────────┘    │
│  │  key_01 → IP_01 │                               │
│  │  key_02 → IP_02 │    ┌─────────────────────┐    │
│  │  key_03 → IP_03 │    │  Health Scorer       │    │
│  │  ...            │    │  (pick best key)     │    │
│  └─────────────────┘    └─────────────────────┘    │
│                                                     │
│  50% of job: managing the pool                      │
└──────────────────────┬──────────────────────────────┘
                       │ optimal key + routing
                       ▼
┌─────────────────────────────────────────────────────┐
│           FREE-TIER PROVIDER FLEET                  │
│                                                     │
│  Gemini × N  │  Groq × N  │  Mistral × N  │  ...   │
│                                                     │
│  50% of job: unlimited free AI inference            │
└─────────────────────────────────────────────────────┘
```

The agent does not manage keys. The rotation layer does not understand agent intent. Together, they produce a system where the agent treats the entire free-tier provider fleet as a single, unlimited, reliable LLM endpoint.

### 8.4 Pre-Warming

For agents with predictable call patterns (tool-use chains, fixed pipelines), the rotation layer can pre-warm a connection to the best available key before the next call arrives.

```python
async def prewarm_next_key(pool: SegmentedPool, expected_tokens: int = 1000):
    """
    After each successful call, pre-establish a connection to the
    next best key so the following call has no connection setup latency.
    """
    next_key = pool.get_key('interactive')
    await next_key.session.get(HEALTH_CHECK_ENDPOINT)
```

---

## 9. Implementation Roadmap

Items are ordered by **impact-to-effort ratio.** Each item is independently implementable.

### Phase 1 — Rotation Logic (High impact, low effort)

| # | Item | Effort | Impact |
|---|---|---|---|
| 1.1 | Differentiate 503 vs 429 handling | 1 day | High |
| 1.2 | Health score struct per key | 2 days | High |
| 1.3 | RPM/RPD/TPM tracked separately | 2 days | High |
| 1.4 | Proactive rotation before quota hit | 1 day | Medium |
| 1.5 | Parse provider retry-after / rate-limit headers | 1 day | Medium |

### Phase 2 — Network Identity (High impact, medium effort)

| # | Item | Effort | Impact |
|---|---|---|---|
| 2.1 | Per-key proxy assignment configuration | 2 days | Very High |
| 2.2 | `curl_cffi` integration for TLS diversity | 1 day | High |
| 2.3 | Per-key User-Agent assignment | 1 day | Medium |
| 2.4 | Organic cadence simulation (upgraded jitter) | 1 day | Medium |

### Phase 3 — Provider Expansion (Medium impact, low effort per provider)

| # | Item | Effort | Impact |
|---|---|---|---|
| 3.1 | Groq integration (OpenAI-compatible) | 1 day | High |
| 3.2 | Mistral AI integration | 1 day | Medium |
| 3.3 | Cerebras/SambaNova integration | 1 day | Medium |
| 3.4 | Vertex AI regional fallback for Gemini keys | 2 days | High |

### Phase 4 — Caching Layer (Medium impact, medium effort)

| # | Item | Effort | Impact |
|---|---|---|---|
| 4.1 | Exact request deduplication | 1 day | Medium |
| 4.2 | Semantic cache with local embedding model | 3 days | High |
| 4.3 | Streaming-first response mode | 2 days | Medium |

### Phase 5 — Agent-Native Interface (High impact, medium effort)

| # | Item | Effort | Impact |
|---|---|---|---|
| 5.1 | RoutingHint / RoutingResult interface | 2 days | High |
| 5.2 | Priority-aware pool segmentation | 2 days | High |
| 5.3 | Parallel key racing for interactive calls | 2 days | Medium |
| 5.4 | Connection pre-warming | 1 day | Medium |

---

## 10. Open Research Items

Items requiring further investigation before design decisions.

### 10.1 Gemini Vertex AI Free Tier Exact Limits

**Question:** Do Vertex AI endpoints for Gemini carry their own independent free quota, or do they share the `generativelanguage.googleapis.com` quota bucket at the account level?

**Why it matters:** if independent, each Google account effectively has two quota pools. If shared, regional fallback helps with 503s but not with 429s.

**Research approach:** provision a test account, saturate the `generativelanguage.googleapis.com` quota, then measure response from `us-central1-aiplatform.googleapis.com` with the same key.

### 10.2 JA3/JA4 Fingerprint Verification

**Question:** do providers actually block or cluster based on TLS fingerprint in practice, or is IP the dominant signal?

**Research approach:** use the same API key from two machines — one with standard `requests`, one with `curl_cffi` impersonating a browser — both behind the same residential IP. Compare quota consumption patterns.

### 10.3 Account Creation Automation — Detection Signals

**Question:** what is Google's current account creation detection threshold? Specifically: are accounts created with a virtual phone number (VoIP) reliably accepted, or are they blocked at higher rates than mobile numbers?

**Research approach:** test account provisioning success rates across VoIP providers. Document per-provider acceptance rates.

### 10.4 Groq Rate Limit Behavior Under Rotation

**Question:** Groq's free tier is documented as "generous" but exact limits are not published. What is the actual RPM/RPD/TPM ceiling, and does it cluster at the account or IP level?

**Research approach:** empirical testing with isolated accounts.

### 10.5 Semantic Cache Threshold Calibration

**Question:** at what cosine similarity threshold does a semantic cache match produce a response that the agent would have accepted from a live call? The 0.94 value in §7.2 is a starting estimate.

**Research approach:** collect a corpus of agent prompts from real KAME usage logs, compute pairwise similarities, manually label acceptable matches, calibrate threshold to maximize recall at acceptable precision.

---

*This document is updated as research progresses and new findings are incorporated.*  
*Next update: after Phase 1 implementation and empirical quota-type validation.*
