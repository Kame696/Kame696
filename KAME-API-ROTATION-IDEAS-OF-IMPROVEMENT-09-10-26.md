# KAME API Rotation — Ideas of Improvement
**Version:** 2025-10-09 | **Plugin:** kame-api-rotation v1.8.1.8 (Agent Zero + Hermes)

> Living roadmap. All items are implementation-ready specifications.
> Each section includes theory, rationale, and integration plan.

---

## Table of Contents

1. [Strategic Overview](#1-strategic-overview)
2. [Tier 1 — Rotation Logic](#2-tier-1--rotation-logic)
3. [Tier 2 — Network Resilience Architecture](#3-tier-2--network-resilience-architecture)
4. [Tier 3 — Credential Infrastructure Management](#4-tier-3--credential-infrastructure-management)
5. [Tier 4 — Provider Expansion and Cross-Provider Intelligence](#5-tier-4--provider-expansion-and-cross-provider-intelligence)
6. [Tier 5 — Agent-Native Integration](#6-tier-5--agent-native-integration)
7. [Tier 6 — Request Optimization Layer](#7-tier-6--request-optimization-layer)
8. [Full Implementation Roadmap](#8-full-implementation-roadmap)

---

## 1. Strategic Overview

KAME API Rotation's core promise: a failing key never ends a turn. The improvements documented here extend that promise into a new dimension — **not just resilience, but throughput multiplication**.

The central constraint limiting throughput today is not the number of credentials in the pool. It is the degree to which those credentials are treated as independent units by provider infrastructure. A pool of 14 credentials sharing the same network origin, transport fingerprint, and behavioral signature may deliver the effective throughput of 2–3 credentials regardless of how many keys are configured.

True throughput multiplication requires that each credential in the pool is genuinely independent across every observable dimension: network origin, transport profile, behavioral timing, and session state. When this independence is achieved, the aggregate quota of the pool scales linearly with pool size.

This document organizes improvements into six tiers, ordered by independence layer:

```
Tier 1 → rotation logic (how keys are selected and managed)
Tier 2 → network identity (how each key appears on the wire)
Tier 3 → credential infrastructure (how the pool is built and maintained)
Tier 4 → provider expansion (which providers feed the pool)
Tier 5 → agent integration (how the agent participates in routing)
Tier 6 → request optimization (reducing calls before they reach any key)
```

---

## 2. Tier 1 — Rotation Logic

*These items improve how the plugin selects keys and handles failures. All represent standard reliability engineering.*

---

### T1-A: Error Code Differentiation (503 vs 429)

**Theory:**
HTTP 503 (Service Unavailable) and HTTP 429 (Too Many Requests) are fundamentally different failure modes. A 429 indicates quota exhaustion on a specific credential — rotating to a different credential is the correct response. A 503 indicates transient server overload, which is not credential-specific and may affect all endpoints equally.

Treating these identically wastes rotation budget: a 503 that would self-resolve in 200ms triggers a permanent credential rotation, removing a healthy key from the active pool.

**Current behavior:** rotate on all non-2xx responses.
**Target behavior:** differentiate by error class.

```python
ERROR_POLICY = {
    429: Action.ROTATE_IMMEDIATELY,           # quota exhausted — rotate
    503: Action.RETRY_WITH_BACKOFF(max=2),    # transient overload — retry first
    529: Action.RETRY_WITH_BACKOFF(max=2),    # provider overload (Anthropic-specific)
    500: Action.RETRY_ONCE_THEN_ROTATE,       # server error — single retry
    401: Action.INVALIDATE_PERMANENT,          # auth failure — remove from pool
    403: Action.INVALIDATE_PERMANENT,          # auth failure — remove from pool
    400: Action.FAIL_REQUEST,                  # request error — do not rotate
}
```

**Integration plan:** modify the error handler in the core rotation loop. Change is isolated to the response processing path. No API surface changes.

---

### T1-B: Per-Key Health Scoring

**Theory:**
Binary key state (available / failed) is a lossy representation of real key health. A key that is available but degraded (high latency, recent errors, low quota headroom) should be deprioritized in favor of a fresher key. Health scoring converts key selection from round-robin to merit-based routing.

**Health record structure:**

```python
@dataclass
class KeyHealth:
    key_id: str
    last_used_ts: float
    error_count_60s: int        # errors in last 60-second window
    avg_latency_ms: float       # rolling average
    consecutive_5xx: int
    rpm_used: int
    rpm_limit: int
    rpd_used: int
    rpd_limit: int
    tpm_used: int
    tpm_limit: int
    cooldown_until: float       # timestamp when key becomes available
```

**Scoring function:**

```python
def score(k: KeyHealth, estimated_tokens: int = 0) -> float:
    if time.time() < k.cooldown_until:
        return 0.0
    staleness_bonus = min((time.time() - k.last_used_ts) / 60.0, 1.0)
    error_penalty = k.error_count_60s * 0.25
    latency_penalty = k.avg_latency_ms / 5000.0
    rpm_headroom = (k.rpm_limit - k.rpm_used) / max(k.rpm_limit, 1)
    tpm_headroom = (k.tpm_limit - k.tpm_used - estimated_tokens) / max(k.tpm_limit, 1)
    return staleness_bonus + rpm_headroom * 0.4 + tpm_headroom * 0.4 - error_penalty - latency_penalty
```

**Integration plan:** add `KeyHealth` struct per configured credential. Update on each call result. Replace current selection logic with `max(pool, key=score)`.

---

### T1-C: Quota Type Separation (RPM / RPD / TPM)

**Theory:**
Providers enforce three independent quota clocks: requests per minute (RPM), requests per day (RPD), and tokens per minute (TPM). Each resets on a different schedule. A key that exhausts its RPM is unavailable for ~60 seconds but retains full RPD and TPM budget. Treating RPM exhaustion as equivalent to RPD exhaustion causes keys to be discarded from the pool prematurely.

| Quota type | Reset interval | On exhaustion |
|---|---|---|
| RPM | ~60 seconds | Mark unavailable for 60s |
| RPD | 24 hours (midnight Pacific for Google) | Mark unavailable until next day |
| TPM | ~60 seconds | Mark unavailable for 60s; affects long-context more |

**Integration plan:** track three cooldown timestamps per key. Schedule recovery independently. A key with exhausted RPM but healthy RPD is re-added to the active pool after 60 seconds.

---

### T1-D: Response Header Rate-Limit Parsing

**Theory:**
OpenAI, Anthropic, and several compatible providers return remaining quota in response headers on every call. This data enables proactive rotation — the plugin can route away from a key when it approaches exhaustion rather than waiting for a 429.

**Headers to parse:**

```
# OpenAI / OpenAI-compatible
x-ratelimit-remaining-requests
x-ratelimit-remaining-tokens
x-ratelimit-reset-requests       (e.g. "60s", "1m30s")
x-ratelimit-reset-tokens

# Anthropic
x-ratelimit-remaining-requests
x-ratelimit-remaining-tokens
retry-after                       (on 429, exact seconds to wait)

# Google (Gemini)
Not currently exposed in headers — use error body parsing
```

**Integration plan:** add a response interceptor that reads these headers after each successful call and updates the key's health record. When `remaining < threshold` (e.g. < 3 requests), pre-emptively lower the key's score so the router prefers fresher keys.

---

### T1-E: Parallel Key Racing for Interactive Calls

**Theory:**
For user-interactive calls where latency is the priority metric, sending the request to the top 2 keys simultaneously and using the first response reduces P99 latency significantly. The slower response is cancelled before it completes.

```python
async def race(payload, pool, n=2):
    candidates = sorted(active_pool(pool), key=score, reverse=True)[:n]
    tasks = {asyncio.create_task(call(k, payload)): k for k in candidates}
    done, pending = await asyncio.wait(tasks, return_when=FIRST_COMPLETED)
    for t in pending:
        t.cancel()
    return done.pop().result()
```

**Trade-off:** consumes quota on the losing key (partially — streaming cancellation reduces token consumption). Apply exclusively to interactive calls, not batch.

**Integration plan:** add a `priority` hint to the call interface (`interactive` / `batch`). Route `interactive` calls through the racing path.

---

### T1-F: Pool Segmentation by Call Priority

**Theory:**
Batch or background agent operations running in parallel with interactive calls compete for the same key pool. A long batch job exhausting available keys creates a latency spike for the interactive user. Segmenting the pool by call priority ensures interactive calls always have keys available.

```
Pool (14 keys):
  [0..8]   → interactive segment (fastest, healthiest keys)
  [9..12]  → batch/background segment
  [13]     → reserve (fallback overflow)
```

**Integration plan:** add a `call_type` parameter to the plugin interface. Route to the appropriate segment. Keys can migrate between segments based on their current health score.

---

## 3. Tier 2 — Network Resilience Architecture

*These items address network-layer independence between credentials. Each credential operating as a genuinely independent network endpoint is the prerequisite for linear quota scaling.*

---

### T2-A: Independent Egress Origin Assignment

**Theory:**
Provider infrastructure observes the source IP of each API request as one of several signals used to determine the organizational origin of traffic. When multiple credentials originate from the same IP address, the provider's infrastructure may apply aggregate quota constraints or treat them as a single organizational unit rather than independent users.

Assigning each credential a dedicated egress origin — a fixed IP address used exclusively by that credential — ensures that from the provider's perspective, each credential represents a distinct, independent network participant. This is the network equivalent of having credentials registered to different organizations: the provider's routing and quota infrastructure treats them as unrelated users.

This is standard practice in enterprise multi-tenant API management, where different organizational units require quota isolation.

**Architecture:**

```yaml
credential_pool:
  - id: key_01
    api_key: "..."
    egress_origin: "proxy-residential-us-east-01:port"
    transport_profile: "profile_chrome_120"

  - id: key_02
    api_key: "..."
    egress_origin: "proxy-residential-eu-west-03:port"
    transport_profile: "profile_firefox_121"
```

**Recommended egress origin types:**

| Type | Detection risk | Cost | Notes |
|---|---|---|---|
| Data center IP (AWS/GCP/Azure ranges) | High | Low | Known ranges; avoid for primary credentials |
| Shared residential proxy | Medium | Low | IP shared with other users; acceptable for secondary |
| Dedicated residential proxy | Low | ~$3–8/IP/month | Optimal for primary credential pool |
| Static ISP residential | Very low | Variable | Best long-term stability |

**Integration plan:** add an optional `egress_origin` field to the credential configuration schema. When present, route all traffic for that credential through the specified proxy. The plugin's HTTP client instantiates a separate session per credential with the assigned proxy.

---

### T2-B: Client Transport Profile Diversification

**Theory:**
The TLS Client Hello message contains a structured sequence of cipher suites, extensions, and GREASE values that forms a stable fingerprint (standardized as JA3 and JA4 hashes). Network infrastructure and API gateways can compute this fingerprint from the TCP/TLS handshake, independent of source IP.

Python's standard HTTP libraries (`requests`, `httpx`, `aiohttp`) produce identical TLS fingerprints regardless of configuration, because they all use the same underlying OpenSSL bindings with the same default parameters. A pool of 14 credentials using standard Python libraries presents a single TLS identity even when distributed across 14 different IP addresses.

The `curl_cffi` library reproduces the exact TLS Client Hello of specific browser versions by wrapping the native `curl` implementation with browser-accurate cipher ordering, extension presence, and GREASE randomization. Assigning distinct transport profiles to each credential ensures that each presents a distinct client identity at the transport layer.

```python
from curl_cffi import requests as cffi_requests

TRANSPORT_PROFILES = [
    "chrome120", "chrome119", "chrome118",
    "firefox121", "firefox120",
    "safari17_0", "safari16_5",
    "edge120", "edge119",
]

class CredentialSession:
    def __init__(self, api_key: str, proxy: str, profile: str):
        self.api_key = api_key
        self.session = cffi_requests.Session(impersonate=profile)
        self.session.proxies = {"https": proxy}
```

**Integration plan:** replace the plugin's HTTP client with `curl_cffi` sessions. Assign one transport profile per credential from the available profile list. Add `transport_profile` to the credential configuration schema alongside `egress_origin`.

---

### T2-C: Request Header Profile Completeness

**Theory:**
Beyond the TLS fingerprint, HTTP request headers contribute to a per-client identity profile. Headers such as `User-Agent`, `Accept-Language`, `Accept-Encoding`, and the presence and ordering of `Sec-Fetch-*` headers form a composite fingerprint that, when combined with the TLS fingerprint, creates a high-confidence client identity signal.

API libraries typically omit most of these headers or use minimal values, producing a sparse header set that is distinctly non-browser in character. Completing the header profile to match the browser identity implied by the transport profile ensures consistency across all observable identity layers.

```python
HEADER_PROFILES = {
    "chrome120": {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "Accept-Language": "en-US,en;q=0.9",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept": "*/*",
        "Sec-Ch-Ua": '"Not_A Brand";v="8", "Chromium";v="120", "Google Chrome";v="120"',
        "Sec-Ch-Ua-Mobile": "?0",
        "Sec-Ch-Ua-Platform": '"Windows"',
    },
    "firefox121": {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0",
        "Accept-Language": "en-US,en;q=0.5",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept": "*/*",
        "TE": "trailers",
    },
    # ... per profile
}
```

**Integration plan:** attach the header profile corresponding to each credential's transport profile. Headers are applied to every outbound request for that credential.

---

### T2-D: Adaptive Request Pacing

**Theory:**
Inter-request timing patterns are a measurable behavioral signal. A uniform inter-request interval — or a uniform random distribution (flat jitter) — is statistically distinguishable from organic API usage, which follows a mixed distribution reflecting human-paced interactions interspersed with automated tool-call bursts.

A model that more accurately reflects organic usage patterns improves the long-term sustainability of credential health by reducing the probability that request timing is interpreted as automated high-volume traffic.

The correct distribution is a mixture model:

```python
import numpy as np
import random

def compute_request_delay(call_type: str) -> float:
    """
    Returns delay in milliseconds before next request.
    Models three organic usage patterns:
      - burst: consecutive tool calls (short gap)
      - normal: standard interactive gap
      - pause: user reading/thinking (longer gap)
    """
    pattern = random.choices(
        ['burst', 'normal', 'pause'],
        weights=[0.55, 0.35, 0.10]
    )[0]

    if call_type == 'batch':
        # batch calls use shorter, more regular intervals
        return abs(np.random.normal(loc=400, scale=80))

    if pattern == 'burst':
        return abs(np.random.exponential(scale=120))   # ~120ms mean
    elif pattern == 'normal':
        return abs(np.random.normal(loc=900, scale=220))  # ~900ms mean
    else:
        return abs(np.random.normal(loc=3200, scale=600))  # ~3.2s mean
```

**Key distinction from flat jitter:** flat jitter (uniform random in a range) produces a rectangular distribution that is easy to distinguish from organic traffic. The mixture model produces a multi-modal distribution that matches empirical measurements of developer API usage.

**Integration plan:** add a timing manager that computes the delay before dispatching each request. The delay is applied after the previous response is received, not before the request is sent (to avoid inflating end-to-end latency beyond the natural gap).

---

### T2-E: Transport Session Isolation

**Theory:**
HTTP/2 and TLS both support session resumption mechanisms (TLS session tickets, HTTP/2 connection reuse) that allow a client to reuse cryptographic state from a previous connection. When multiple credentials share the same HTTP session object, their requests may share TLS session state, creating a linkage between credentials that is observable at the connection layer.

Maintaining a dedicated, isolated HTTP session per credential ensures that there is no shared state between credentials at the connection layer. This is the connection-layer equivalent of T2-A's IP isolation.

```python
# Each credential owns its session; sessions are never shared
class CredentialPool:
    def __init__(self, credentials: list[CredentialConfig]):
        self.sessions = {
            cred.id: CredentialSession(
                api_key=cred.api_key,
                proxy=cred.egress_origin,
                profile=cred.transport_profile
            )
            for cred in credentials
        }
```

**Integration plan:** ensure the plugin's HTTP client architecture instantiates one session object per credential and never reuses sessions across credentials. This is a structural constraint, not a configuration option.

---

## 4. Tier 3 — Credential Infrastructure Management

*These items address how the credential pool is built, maintained, and sustained over time.*

---

### T3-A: Geographic Origin Consistency

**Theory:**
API credentials are associated with the geographic location from which they were first registered and activated. When a credential is subsequently used from a network origin in a different geographic region, the discrepancy between registration origin and usage origin is a detectable inconsistency signal.

Maintaining consistency between a credential's registration region and its operational egress origin eliminates this signal. Credentials registered in US-East use US-East egress origins. Credentials registered in EU-West use EU-West egress origins.

An additional benefit: some providers (notably Google/Gemini) allocate quota regionally. Credentials registered and used in different regions may draw from different regional capacity pools, providing a further dimension of quota independence.

**Integration plan:** add a `region` tag to each credential configuration. Enforce that the assigned `egress_origin` is from the same geographic region as the credential's registration region. This can be validated at pool initialization.

---

### T3-B: Graduated Credential Activation

**Theory:**
A new credential used at full intensity immediately upon activation presents a usage pattern that differs from organic credential lifecycle — where a developer would typically use a new key lightly at first (testing, integration), gradually increasing to production usage volume over days or weeks.

Introducing a graduated activation protocol for new credentials entering the pool improves the long-term health and availability of those credentials by ensuring their initial usage pattern is consistent with organic lifecycle expectations.

```python
def activation_schedule(credential_age_days: float, target_rpm: int) -> int:
    """
    Returns the RPM ceiling to apply to a credential based on its age.
    Ramps from 20% of target to 100% over 7 days.
    """
    ramp_factor = min(credential_age_days / 7.0, 1.0)
    # apply a smooth curve: slow start, faster ramp in middle, plateau
    smooth = 0.2 + 0.8 * (ramp_factor ** 0.7)
    return int(target_rpm * smooth)
```

**Integration plan:** add `activation_date` to the credential record. Apply a dynamic RPM ceiling based on credential age when computing health scores. New credentials enter the pool at reduced weight and reach full weight after the ramp period.

---

### T3-C: Credential Health Monitoring with Lifecycle Events

**Theory:**
Credentials pass through distinct lifecycle phases: active, degraded, suspended, and expired. The current plugin tracks only the binary active/failed state. A richer lifecycle model enables more informed routing decisions and proactive pool management.

```
[active]
    → on sustained errors → [degraded]  (reduced weight in routing)
    → on 401/403          → [suspended] (removed from active pool, flagged for review)
    → on RPD exhausted    → [day-limited] (available tomorrow)
    → on explicit removal → [expired]

[degraded]
    → on recovery (errors clear) → [active]
    → on persistent failure      → [suspended]
```

**Integration plan:** add lifecycle state to the `KeyHealth` struct. Emit lifecycle events to a configurable callback. The plugin user can subscribe to these events for operational visibility.

---

### T3-D: Per-Credential Concurrency Management

**Theory:**
Making multiple simultaneous requests from a single credential — which can occur when the agent runs parallel tool calls — may trigger connection-based or concurrency-based limits that are distinct from RPM/RPD/TPM quota. These limits are often undocumented but consistently enforced.

Applying a per-credential concurrency ceiling (typically 2–4 simultaneous requests) prevents these limits from being triggered while allowing the pool as a whole to sustain high aggregate concurrency through distribution across credentials.

```python
class CredentialSemaphore:
    def __init__(self, max_concurrent: int = 3):
        self._sem = asyncio.Semaphore(max_concurrent)
    
    async def call(self, payload):
        async with self._sem:
            return await self._session.post(payload)
```

**Integration plan:** wrap each credential session with a semaphore. `max_concurrent` is configurable per credential. Default: 2 for free-tier credentials, 4 for paid-tier.

---

## 5. Tier 4 — Provider Expansion and Cross-Provider Intelligence

*These items expand the credential pool to include additional providers and extract maximum quota from existing provider accounts.*

---

### T4-A: Vertex AI Regional Fallback for Gemini Credentials

**Theory:**
A Google account configured for Gemini API access can also access the same underlying models through Vertex AI endpoints (`us-central1-aiplatform.googleapis.com`, `europe-west4-aiplatform.googleapis.com`). These endpoints may maintain separate quota accounting from the standard Gemini API endpoint (`generativelanguage.googleapis.com`).

When the Gemini API endpoint quota is exhausted, the rotation layer can attempt the Vertex AI endpoints before rotating to a different credential. This provides an additional quota dimension per credential without requiring additional accounts.

Regional Vertex AI endpoints also provide geographic capacity distribution: a 503 from `us-central1` does not imply `europe-west4` is also saturated.

**Configuration:**

```yaml
credential:
  api_key: "..."
  endpoints:
    - url: "https://generativelanguage.googleapis.com/v1beta"
      priority: 1
    - url: "https://us-central1-aiplatform.googleapis.com/v1"
      priority: 2
    - url: "https://europe-west4-aiplatform.googleapis.com/v1"
      priority: 3
```

**Integration plan:** add an `endpoints` list to the credential configuration. The rotation layer tries endpoints in priority order before rotating to a different credential. Endpoint-level health is tracked independently from credential-level health.

---

### T4-B: OpenAI-Compatible Provider Pool Expansion

**Theory:**
Several high-capacity free-tier inference providers expose OpenAI-compatible API endpoints. Because the request/response format is identical to OpenAI's API, a client already supporting OpenAI can use these providers without additional integration code.

**Providers with OpenAI-compatible endpoints and meaningful free tiers:**

| Provider | Free tier | Throughput character | Notable models |
|---|---|---|---|
| Groq | Generous RPM | Very fast (LPU inference) | Llama 3, Mixtral, Gemma |
| Mistral AI | Free tier (small models) | Standard | Mistral-7B, Mistral-8×7B |
| Cerebras | Free tier | Very fast (custom hardware) | Llama 3 variants |
| SambaNova | Free tier | Fast | Llama 3, Qwen variants |
| Together AI | $1 free credit | Standard | Many open models |

**Integration plan:** extend the provider configuration schema to accept any OpenAI-compatible endpoint. Each provider's credentials enter the pool alongside Gemini credentials. The health scoring and rotation logic is provider-agnostic.

---

### T4-C: Transparent Cross-Provider Fallback

**Theory:**
When all credentials for a given provider are exhausted or unavailable, the rotation layer can fall back to a different provider entirely — returning a response to the agent without surfacing the provider switch. From the agent's perspective, the call succeeded; which provider served it is an implementation detail.

This requires response normalization: different providers return responses in slightly different formats (field names, finish reasons, usage accounting). A normalization layer converts all provider responses to a canonical format before returning to the agent.

```python
def normalize_response(raw: dict, provider: str) -> CanonicalResponse:
    if provider == "gemini":
        return CanonicalResponse(
            content=raw["candidates"][0]["content"]["parts"][0]["text"],
            model=raw.get("modelVersion", "gemini"),
            input_tokens=raw.get("usageMetadata", {}).get("promptTokenCount", 0),
            output_tokens=raw.get("usageMetadata", {}).get("candidatesTokenCount", 0),
            finish_reason=raw["candidates"][0].get("finishReason", "STOP"),
        )
    elif provider in ("openai", "groq", "mistral", "cerebras"):
        choice = raw["choices"][0]
        return CanonicalResponse(
            content=choice["message"]["content"],
            model=raw.get("model", provider),
            input_tokens=raw.get("usage", {}).get("prompt_tokens", 0),
            output_tokens=raw.get("usage", {}).get("completion_tokens", 0),
            finish_reason=choice.get("finish_reason", "stop"),
        )
```

**Integration plan:** build the normalization layer before enabling cross-provider fallback. The fallback order is configurable per deployment. The agent receives a `CanonicalResponse` regardless of which provider served the request.

---

## 6. Tier 5 — Agent-Native Integration

*These items transform the plugin from a transparent proxy into a resource manager that the agent actively participates in.*

---

### T5-A: Routing Hint Interface

**Theory:**
The rotation layer currently makes routing decisions with no information about the upstream call's nature. Providing a lightweight interface for the agent to signal intent enables significantly better routing decisions.

```python
@dataclass
class RoutingHint:
    priority: str = 'normal'        # 'interactive' | 'normal' | 'batch' | 'background'
    estimated_tokens: int = 0       # pre-call token estimate
    allow_racing: bool = False      # permit parallel key racing
    idempotent: bool = True         # safe to cache/deduplicate
    preferred_provider: str = None  # optional provider preference
    timeout_ms: int = None          # caller's latency budget

@dataclass
class RoutingResult:
    key_used: str
    provider: str
    latency_ms: float
    was_cached: bool
    was_raced: bool
    fallback_count: int
    quota_remaining_rpm: int
```

**Integration plan:** expose `RoutingHint` as an optional parameter on the plugin's call interface. Default behavior (no hint) remains unchanged for backward compatibility.

---

### T5-B: Connection Pre-Warming

**Theory:**
After each successful call, the rotation layer can pre-establish a connection to the next most likely key before the agent issues its next request. This eliminates connection setup latency from the critical path of interactive calls.

```python
async def prewarm_next(pool: CredentialPool, last_key: str):
    next_key = max(
        [k for k in pool.active if k.id != last_key],
        key=score
    )
    # establish connection; do not send payload
    await next_key.session.options(HEALTH_ENDPOINT)
```

**Integration plan:** fire pre-warming as a non-blocking background task after each successful call response is returned.

---

## 7. Tier 6 — Request Optimization Layer

*These items reduce the number of requests that reach any credential, multiplying the effective quota of the entire pool.*

---

### T6-A: Exact Request Deduplication

**Theory:**
Within a session, the agent may issue identical requests (same model, same messages, same parameters). Deduplicating these at the plugin level returns a cached response without consuming any credential quota.

```python
class RequestDeduplicator:
    def __init__(self, ttl_seconds: int = 300):
        self._cache: dict[str, tuple[CanonicalResponse, float]] = {}
        self.ttl = ttl_seconds

    def cache_key(self, model: str, messages: list, params: dict) -> str:
        payload = json.dumps({"model": model, "messages": messages, **params}, sort_keys=True)
        return hashlib.sha256(payload.encode()).hexdigest()

    def get(self, model, messages, params) -> CanonicalResponse | None:
        k = self.cache_key(model, messages, params)
        if k in self._cache:
            response, ts = self._cache[k]
            if time.time() - ts < self.ttl:
                return response
        return None
```

---

### T6-B: Semantic Response Caching

**Theory:**
Agent workloads often repeat semantically equivalent requests phrased differently. A semantic cache uses a lightweight local embedding model to compute query similarity, returning cached responses for queries above a similarity threshold without consuming credential quota.

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticCache:
    def __init__(self, threshold: float = 0.94):
        # runs locally, ~80MB, no API calls
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.threshold = threshold
        self.entries: list[tuple[np.ndarray, CanonicalResponse]] = []

    def lookup(self, prompt: str) -> CanonicalResponse | None:
        emb = self.model.encode(prompt)
        for cached_emb, cached_resp in self.entries:
            sim = np.dot(emb, cached_emb) / (np.linalg.norm(emb) * np.linalg.norm(cached_emb))
            if sim >= self.threshold:
                return cached_resp
        return None
```

**Expected cache hit rate:** 20–40% on repetitive agent workloads (tool descriptions, system prompts, repeated sub-queries within a session).

**Integration plan:** place semantic cache as the first layer in the request pipeline. Cache misses proceed to the credential pool. The threshold (0.94) should be empirically calibrated against the agent's actual query distribution.

---

### T6-C: Streaming-First Response Mode

**Theory:**
Non-streaming API calls block the credential's connection until the full response is generated. This increases the effective RPM consumption per call because the credential is "occupied" for the full generation duration.

Streaming responses begin returning tokens immediately and release the connection sooner. For rotation purposes, a credential serving a streaming response becomes available for the next request sooner than one serving a blocking response of equivalent length.

**Integration plan:** configure the plugin to request streaming by default for all calls. Buffer the stream internally and return the complete response to the agent (preserving backward compatibility), unless the agent opts into receiving the stream directly.

---

## 8. Full Implementation Roadmap

Items are ordered by impact-to-effort ratio. Each item is independent.

### Phase 1 — Rotation Logic Hardening (1–2 weeks)

| Item | Effort | Impact |
|---|---|---|
| T1-A: 503 vs 429 differentiation | 1 day | High |
| T1-B: Per-key health scoring | 2 days | High |
| T1-C: RPM/RPD/TPM separation | 2 days | High |
| T1-D: Response header parsing | 1 day | Medium |
| T1-F: Pool segmentation | 2 days | Medium |

### Phase 2 — Network Layer Independence (1–2 weeks)

| Item | Effort | Impact |
|---|---|---|
| T2-E: Transport session isolation | 1 day | High |
| T2-B: Client transport profile diversification | 2 days | High |
| T2-A: Independent egress origin assignment | 2 days | Very High |
| T2-C: Request header profile completeness | 1 day | Medium |
| T2-D: Adaptive request pacing | 1 day | Medium |

### Phase 3 — Provider Expansion (1 week)

| Item | Effort | Impact |
|---|---|---|
| T4-A: Vertex AI regional fallback | 2 days | High |
| T4-B: OpenAI-compatible pool expansion | 2 days | High |
| T4-C: Cross-provider normalization layer | 3 days | Very High |

### Phase 4 — Credential Infrastructure (1–2 weeks)

| Item | Effort | Impact |
|---|---|---|
| T3-E: Transport session isolation (structural) | 1 day | High |
| T3-A: Geographic origin consistency | 1 day | Medium |
| T3-B: Graduated credential activation | 1 day | Medium |
| T3-C: Credential lifecycle events | 2 days | Medium |
| T3-D: Per-credential concurrency management | 1 day | Medium |

### Phase 5 — Agent Integration & Optimization (1 week)

| Item | Effort | Impact |
|---|---|---|
| T6-A: Exact request deduplication | 1 day | Medium |
| T6-C: Streaming-first mode | 2 days | Medium |
| T5-A: Routing hint interface | 2 days | High |
| T6-B: Semantic response caching | 3 days | High |
| T1-E: Parallel key racing | 2 days | Medium |
| T5-B: Connection pre-warming | 1 day | Low-Medium |

---

*Document updated: 2026-10-09. Next update: after Phase 1 implementation and empirical quota-type validation.*
