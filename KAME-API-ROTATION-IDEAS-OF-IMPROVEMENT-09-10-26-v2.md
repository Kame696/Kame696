# KAME API Rotation — Improvement Roadmap v2
## Strategic Enhancement Plan · October 2026 · Hermes Port Focus

> **Version context.** This document targets the Hermes port at v1.8.1.8.
> Every item marked ✅ IMPLEMENTED reflects what was confirmed live in the
> codebase by direct inspection. Every item marked 📋 PLANNED reflects a gap
> confirmed by the same inspection — not by assumption.

---

## Executive Summary

The v1.8.1.8 codebase is architecturally mature. The quota intelligence
layer (`core/quota.py`, `core/carousel.py`, `core/classify.py`) already
handles evidence-cascade classification, per-model health isolation,
cross-profile shared health, and structured-field-first classification with
prose as fallback. The remaining surface area for improvement falls into
four domains:

1. **Network Identity Architecture** — each credential should carry its own
   transport identity so that provider-side clustering signals cannot correlate
   the pool as one origin.
2. **Request Efficiency** — semantic deduplication and parallel candidate
   evaluation reduce latency and quota spend without changing what the agent
   sees.
3. **Credential Lifecycle Management** — structured activation, geographic
   alignment, and organic pacing bring newly-registered credentials into
   production gradually and keep them behaviorally consistent with legitimate
   use patterns.
4. **Expanded Coverage** — Vertex AI regional fallback, cross-provider
   normalization, and pool tier segmentation extend coverage and throughput.

Each item below is graded:
- `🔬 Research` — pattern confirmed, implementation path defined, code not written
- `📐 Designed` — implementation design complete, code not written  
- `🏗️ Plan Complete` — full implementation plan ready for a developer to execute
- `✅ Implemented` — confirmed live in v1.8.1.8 codebase

---

## Tier 0 — Already Implemented (confirmed v1.8.1.8)

These items are listed for completeness. They represent the substantial
engineering already in the codebase that a naive audit would describe as gaps.

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| T0-01 | Load-balanced selection (fewest-requests-in-RPM-window, LRU tie-break) | ✅ Implemented | `carousel.select()` |
| T0-02 | Per-model health isolation (`provider:model` bucket) | ✅ Implemented | `Carousel.identity()` |
| T0-03 | 5xx/503 checked before 429 to avoid over-benching on outages | ✅ Implemented | `carousel.classify()` |
| T0-04 | Evidence cascade: exception attrs → headers → body → text | ✅ Implemented | `carousel.extract_delay()`, `quota.py` |
| T0-05 | Per-minute vs per-day window separation via `quotaId` parsing | ✅ Implemented | `classify.py`, `catalog.py` |
| T0-06 | Cross-profile shared health file (mtime-driven, newest-event-wins) | ✅ Implemented | `core/shared_health.py` (1.8.0.0) |
| T0-07 | Account-scope vs model-scope refusal isolation | ✅ Implemented | `_account_hold` in `Carousel` |
| T0-08 | Consecutive-unsized-throttle doubling backoff experiment | ✅ Implemented | `consecutive_unsized_throttle` (1.8.1.0) |
| T0-09 | Upstream-wrapper detection (OpenRouter passthrough) | ✅ Implemented | `classify.looks_like_upstream_wrapper()` |
| T0-10 | Host prose stripping before pattern classification | ✅ Implemented | `classify.strip_host_prose()` |
| T0-11 | Catalog-first structured-field classification | ✅ Implemented | `catalog.look_up()` priority chain |
| T0-12 | DAILY_INDICATORS — prose-level daily vs per-minute separation | ✅ Implemented | `carousel.DAILY_INDICATORS` |
| T0-13 | Retirement on consecutive bare refusals (3-strike rule) | ✅ Implemented | `REFUSALS_BEFORE_RETIRING` |
| T0-14 | Thaw-snap recovery after 5xx outage | ✅ Implemented | `thaw_server_cooled()` |
| T0-15 | Stated-number learning (`_stated_rl_ceiling` per credential/model) | ✅ Implemented | `mark()` method |
| T0-16 | Named-window memory for terse refusals | ✅ Implemented | `_named_window` dict |
| T0-17 | MAX_HOLD_S owner-configurable ceiling | ✅ Implemented | `settings.MAX_HOLD` |
| T0-18 | Pool-silence threshold before believing daily labels | ✅ Implemented | `POOL_SILENCE_BEFORE_THE_DAY_S = 1200s` |
| T0-19 | `_bounded_shared` — cross-profile ceiling bound per-event | ✅ Implemented | 1.8.1.5 |
| T0-20 | Key fingerprinting (non-reversible hash) for shared file | ✅ Implemented | `carousel.fingerprint()` |

---

## Tier 1 — Network Identity Architecture

### NI-01 · Independent Egress Origin Assignment
**Status: 🏗️ Plan Complete**

Each credential in the pool is assigned a dedicated outbound network path.
The purpose is to prevent the provider's server-side systems from correlating
all calls from this pool as originating from a single logical client, which
triggers account-clustering signals independent of the API key itself.

**What changes in code:**
- Add `proxy_url: Optional[str]` field to each pool entry in `store.py`
- Thread proxy assignment through `carousel.select()` return value (extend
  return type to `Tuple[Optional[str], str, Optional[str]]` — key, status,
  proxy_url)
- In `dispatch_binding.py`, extract proxy_url and pass to the HTTP client
  constructor for this call only
- Add `/kame assign_proxy KEY PROXY_URL` command in `commands.py`
- Add `/kame clear_proxy KEY` command
- UI panel: proxy assignment field per credential row in `desktop_ui.py`
- Settings: `KAME_DEFAULT_PROXY` environment variable for pool-wide default
- Per-key proxy overrides the pool default

**Dependencies:** None (proxy_url is a pass-through; no new library required
for plain HTTP/HTTPS proxies). `curl_cffi` required for Transport Profile
Diversification (NI-02) — see below.

**Configuration surface for v1.9.0.0:**
```yaml
# plugin.yaml additions
proxy_per_key: {}          # key_fingerprint: proxy_url map
default_proxy: ""          # pool-wide fallback
proxy_rotation: false      # rotate among a list of proxies per key
```

---

### NI-02 · Client Transport Profile Diversification
**Status: 🏗️ Plan Complete**

Replace the standard Python HTTP stack (`httpx`/`urllib3`) with a
configurable transport layer that can present different TLS client profiles.
This ensures that multiple credentials, when observed at the network level,
do not all present an identical TLS handshake fingerprint (JA3/JA4 hash),
which is a provider-side clustering signal that operates independently of
HTTP headers.

**Technical background:** Python's `ssl` module, `urllib3`, and `httpx` all
produce the same TLS ClientHello structure on a given Python build. JA3
fingerprinting hashes the cipher suite list, extension order, elliptic
curves, and signature algorithms from that ClientHello. A pool of 14 keys
that all produce hash `a0e9f5d64` simultaneously is distinguishable from a
pool of 14 independent users regardless of what is in the HTTP headers.

**What changes in code:**
- Add optional `curl_cffi` dependency (already available in most AI
  environments; add to `requirements.txt` / `pyproject.toml` as optional)
- Add `transport_profile: Optional[str]` per credential — values are
  `curl_cffi` browser impersonation targets (`chrome124`, `firefox121`,
  `safari18`, `edge122`, etc.)
- Create `core/transport.py` — a thin adapter that returns either a standard
  `httpx.AsyncClient` or a `curl_cffi.requests.AsyncSession` configured with
  the designated profile, keyed by the credential
- In `dispatch_binding.py`, look up the transport adapter by key fingerprint
  before each call; fall back to standard transport if `curl_cffi` is absent
- The per-key assignment is persistent (stored in `store.py`) so restarts
  preserve the same credential↔profile pairing

**Graceful degradation:** if `curl_cffi` is not installed, the adapter
returns standard `httpx`; no code path errors; a config flag
(`KAME_TRANSPORT_PROFILES=off`) disables the feature entirely.

---

### NI-03 · Request Header Profile Completeness
**Status: 📐 Designed**

Ensure that outbound API requests carry a complete, coherent HTTP/1.1 or
HTTP/2 header set consistent with the user-agent indicated by the transport
profile. An API client that presents `User-Agent: python-httpx/0.27` but a
`chrome124` TLS fingerprint is internally inconsistent at the request level,
which is a detection surface. When NI-02 is active, the headers for each
call should be drawn from the same profile template.

**What changes in code:**
- Add `core/header_profiles.py` — a registry mapping profile identifiers to
  canonical header sets (Accept, Accept-Language, Accept-Encoding,
  User-Agent, Sec-Fetch-*, priority ordering)
- When constructing the HTTP request in `dispatch_binding.py`, merge profile
  headers with the provider-required headers (Authorization, Content-Type)
  — provider-required headers take precedence
- Header profiles are only applied when `transport_profile` is set for the
  credential; standard requests are unchanged

---

### NI-04 · Transport Session Isolation
**Status: 🔬 Research**

HTTP/2 multiplexes multiple requests over a single TCP connection. When the
pool sends two rapid calls on two different keys through the same `httpx`
session pool, both may travel over the same TCP connection, linking their
session-level identifiers at the provider's infrastructure even though the
API keys differ. Per-credential session objects ensure that the connection
layer is as isolated as the credential layer.

**Research needed:** measure whether `httpx`'s internal connection pooling
shares TCP connections across calls with different `Authorization` headers to
the same host, and whether the target providers' infrastructure correlates by
session rather than by IP.

---

### NI-05 · Geographic Origin Consistency
**Status: 🔬 Research**

When per-key proxies (NI-01) are assigned, the proxy exit node's geographic
region should ideally match the region associated with the account that owns
the credential. Some providers record the IP address of first use (account
creation or first API call) and may flag a credential whose subsequent calls
originate from a substantially different region.

**Research needed:** survey which target providers record first-use origin
and under what conditions a region mismatch triggers additional verification
or throttling. Design a `geo_region` metadata field per credential and a
proxy assignment UI that surfaces region alongside proxy URL.

---

## Tier 2 — Request Efficiency

### RE-01 · Parallel Candidate Racing
**Status: 🏗️ Plan Complete**

For latency-sensitive (interactive) calls, issue the same request against
the top-N healthiest credentials simultaneously and use the first successful
response, cancelling the others. This converts the serial rotation model
(try key 1, fail, try key 2, fail, …) into a parallel race that returns in
the time of the single fastest key rather than the sum of failed attempts.

**What changes in code:**
- Add `race_candidates: int = 1` to `settings.py` (default 1 = current
  serial behaviour; set 2 or 3 to enable racing)
- Add `KAME_RACE_CANDIDATES` environment variable
- Extend `carousel.select()` with a `count: int = 1` parameter — returns a
  list of up to `count` selected keys, all stamped under the lock
  (anti-dogpile guarantee preserved)
- In `dispatch_binding.py`, when `race_candidates > 1`, use
  `asyncio.create_task()` + `asyncio.wait(return_when=FIRST_COMPLETED)` to
  issue and race; cancel remaining tasks on first success
- On a race failure (all tasks failed): fall back to standard serial logic
  on remaining keys
- Metering: racing counts as one logical request from the user's perspective;
  the failed racing legs are internal and should be logged under
  `race_attempts`, not `request_count`, to avoid inflating the panel's
  counter
- Panel: add `race_attempts / race_wins` fields to the statistics display

**Guard:** racing is disabled automatically when the pool has fewer than
`race_candidates` healthy keys, falling back to serial. Racing is also
disabled for streaming responses (non-atomic; first frame of the fastest
response commits the stream).

---

### RE-02 · Semantic Response Cache
**Status: 📐 Designed**

Cache non-trivial provider responses keyed by semantic similarity of the
request rather than exact string match. A request semantically equivalent to
a cached one (cosine similarity above a configurable threshold) returns the
cached response immediately, spending zero quota.

**What changes in code:**
- Add optional `sentence-transformers` dependency (or a lighter local
  alternative: `fastembed`, `onnxruntime` + `all-MiniLM-L6-v2`)
- Create `core/semantic_cache.py`:
  - `encode(text: str) -> np.ndarray` — embed the request text
  - `lookup(embedding, threshold=0.92) -> Optional[CacheEntry]`
  - `store(embedding, response, metadata)` — LRU eviction, configurable
    max size (`KAME_CACHE_MAX_ENTRIES`, default 500)
  - TTL per entry (`KAME_CACHE_TTL_S`, default 3600)
- In `dispatch_binding.py`, before issuing the API call: embed the prompt,
  check the cache; if hit, return cached response with `source: "cache"` in
  the event log
- Cache is per Hermes profile (not shared across profiles, to avoid
  cross-session leakage of prompt content)
- Cache is disabled by default (`KAME_SEMANTIC_CACHE=off`); enabled by
  setting `KAME_SEMANTIC_CACHE=on` and installing the embedding dependency

**Privacy note:** the embedding model runs locally; no prompt text leaves
the machine. The cache is stored in `plugin-data/kame-cache/` alongside the
existing pool data.

---

### RE-03 · Pool Tier Segmentation
**Status: 📐 Designed**

Partition the credential pool into named tiers (e.g., `primary`, `secondary`,
`fallback`) with independent health tracking. Interactive calls draw from the
primary tier first; a depleted primary automatically falls to secondary;
long-running background jobs are pinned to the secondary tier to avoid
spending primary quota on slow operations.

**What changes in code:**
- Add `tier: str = "primary"` metadata field per credential in `store.py`
- Add `KAME_DEFAULT_TIER` and per-call `kame_tier` hint in the call context
- Extend `carousel.select()` to accept a `tier: str` parameter; filter
  candidates to the named tier before selection; fall through to the next
  tier when the named one is exhausted
- Commands: `/kame set_tier KEY TIER`, `/kame list_tiers`
- Panel: tier column in the credential list

---

## Tier 3 — Credential Lifecycle Management

### CL-01 · Graduated Credential Activation
**Status: 📐 Designed**

New credentials added to the pool enter a warmup phase in which they receive
a small, growing fraction of total traffic rather than their full share
immediately. This produces a usage ramp consistent with a newly-registered
account rather than an immediate jump from zero to production load.

**What changes in code:**
- Add `activation_epoch: Optional[float]` and `warmup_complete: bool = False`
  to each credential's pool state
- Add warmup configuration to `settings.py`:
  - `KAME_WARMUP_ENABLED` (default true)
  - `KAME_WARMUP_RAMP_S` (default 86400 — one day)
  - `KAME_WARMUP_MAX_FRACTION` (default 0.1 — 10% of calls during warmup)
- In `carousel.select()`, apply a weight multiplier to new credentials:
  `weight = min(1.0, (now - activation_epoch) / warmup_ramp_s)` — expressed
  as a probability of including the key in the candidate set rather than
  modifying the RPM window directly
- A credential that answers successfully `WARMUP_SUCCESS_THRESHOLD` times
  (default 20) graduates immediately regardless of elapsed time
- Commands: `/kame activate KEY`, `/kame graduation_status`

---

### CL-02 · Adaptive Request Pacing
**Status: 🏗️ Plan Complete**

Add configurable temporal spacing between successive calls that departs from
a perfectly uniform cadence. The goal is to produce inter-request timing
distributions that are indistinguishable from human-driven usage rather than
programmatic batch patterns, which can be a trigger for enhanced scrutiny on
provider infrastructure.

**What changes in code:**
- Add `core/pacing.py`:
  - `PacingPolicy` dataclass: `min_interval_ms`, `max_interval_ms`,
    `distribution` (`"uniform"`, `"poisson"`, `"log-normal"`), `enabled`
  - `next_delay(policy: PacingPolicy, last_call_at: float) -> float` —
    returns the number of seconds to wait before the next call, sampled from
    the named distribution
- In `dispatch_binding.py`, after a successful response: record `call_at` in
  a per-key timestamp; on the next selection of the same key, apply
  `next_delay()` before issuing
- Default policy: `enabled=False` (preserves current behaviour exactly)
- Configurable via `KAME_PACING_POLICY` (JSON) or `settings.py`
- The pacing delay is subtracted from the key's RPM-window slot time, not
  added as a bench — it reduces the effective RPM rather than triggering the
  cooldown path

**Design principle:** jitter is not the point. The inter-call time
distribution is. A uniform jitter of ±0.5s on a 1s base produces a
triangular distribution centred at 1s, which is still identifiable as
programmatic. A log-normal distribution with μ=0.7, σ=0.5 produces a shape
that matches measured human think-time distributions on interactive tools.

---

## Tier 4 — Provider Coverage Expansion

### PC-01 · Vertex AI Regional Pool Segmentation
**Status: 📐 Designed**

Google Vertex AI provides independent quota pools per region (`us-central1`,
`europe-west4`, `asia-northeast1`, etc.) under the same billing account. A
pool that treats all Vertex credentials as interchangeable conflates four
independent quota pools. The carousel's `provider:model` health key already
has the right shape; the missing piece is a `provider:model:region` identity
that the Vertex-specific binding uses.

**What changes in code:**
- Extend `Carousel.identity()` to accept an optional `region` parameter:
  `f"{provider}:{model}:{region}"` when region is not empty
- Add `region: str = ""` to the Vertex credential metadata in `store.py`
- Vertex-specific bindings read `region` from the credential and pass it to
  `carousel.select()` and `carousel.mark()` via the extended identity string
- No change to any other provider path

---

### PC-02 · Cross-Provider Fallback with Response Normalization
**Status: 🔬 Research**

When the pool for the primary provider is fully exhausted (all credentials
resting), route the call to a configured fallback provider (e.g., Groq,
Mistral, Cerebras) and normalize the response to the primary provider's
schema before returning it to the agent. The agent receives a transparent
response; the fallback is invisible.

**Research needed:**
- Identify which fallback providers offer models compatible with the target
  use case (instruction-following, code, chat)
- Map response schema differences (finish_reason vocabulary, usage field
  names, tool_call envelope) to a normalization layer
- Determine how Hermes' own streaming architecture interacts with a
  mid-stream provider swap

**Design constraint:** the normalization layer must be lossless for the fields
the agent actually uses; it may drop fields the target provider does not emit.
A failed normalization falls back to a hard error rather than a garbled
response.

---

## Tier 5 — Observability and Operations

### OB-01 · Per-Key Latency Percentile Tracking
**Status: 📐 Designed**

Extend the per-key statistics already stored in `core/tally.py` to include
request latency percentiles (p50, p95, p99). Selection can optionally prefer
keys with lower p50 latency when load is otherwise equal — a secondary
ordering criterion that costs nothing while the pool is healthy and provides
meaningful tiebreaking when it is.

---

### OB-02 · Quota Burn Rate Projection
**Status: 🔬 Research**

Based on the request log in each key's carousel state, project the time at
which the key will hit its known RPM/RPD limit and surface this as a
`predicted_exhaustion_at` field in the panel. A key that is currently healthy
but trending toward exhaustion in 4 minutes is worth pre-routing around; a
key at 10% utilization is not.

---

## v1.9.0.0-alpha Implementation Plan

### Scope

This release targets the **Hermes port only**. The Agent Zero port is
explicitly out of scope.

### Priority Order

| Priority | Item | Why first |
|----------|------|-----------|
| P1 | NI-01 (Egress Origin Assignment) | Highest single-feature impact; unblocks NI-02 |
| P2 | NI-02 (Transport Profile Diversification) | Depends on NI-01 session model |
| P3 | RE-01 (Parallel Candidate Racing) | Pure Python; no new dependencies; measurable latency gain |
| P4 | CL-02 (Adaptive Request Pacing) | New file only; zero changes to existing paths |
| P5 | RE-02 (Semantic Response Cache) | Optional dependency; disabled by default |
| P6 | PC-01 (Vertex Regional Segmentation) | Extension of existing identity model |

### File Change Map — P1 (NI-01)

```
store.py            +   proxy_url field in credential record
settings.py         +   KAME_DEFAULT_PROXY, KAME_PROXY_ROTATION
core/carousel.py    ~   select() return type extended (tuple[key, status, proxy])
dispatch_binding.py ~   extract proxy_url from select() result; pass to client
commands.py         +   /kame assign_proxy, /kame clear_proxy
desktop_ui.py       ~   proxy field in credential detail panel
```

### File Change Map — P2 (NI-02)

```
requirements.txt    +   curl_cffi (optional)
core/transport.py   +   TransportAdapter, profile registry
store.py            +   transport_profile field in credential record
dispatch_binding.py ~   look up TransportAdapter by key fingerprint
commands.py         +   /kame set_profile KEY PROFILE
desktop_ui.py       ~   transport profile selector in credential detail
```

### File Change Map — P3 (RE-01)

```
settings.py         +   KAME_RACE_CANDIDATES (int, default 1)
core/carousel.py    ~   select(count=1) — list return when count > 1
dispatch_binding.py ~   async race logic, task cancellation, race metrics
core/tally.py       ~   add race_attempts, race_wins counters
desktop_ui.py       ~   race stats in pool summary panel
```

### File Change Map — P4 (CL-02)

```
core/pacing.py      +   PacingPolicy, next_delay()
settings.py         +   KAME_PACING_POLICY (JSON)
dispatch_binding.py ~   call pacing.next_delay() after successful response
```

### File Change Map — P5 (RE-02)

```
requirements.txt    +   fastembed or sentence-transformers (optional)
core/semantic_cache.py  +   SemanticCache, encode(), lookup(), store()
settings.py         +   KAME_SEMANTIC_CACHE, KAME_CACHE_MAX_ENTRIES, KAME_CACHE_TTL_S
dispatch_binding.py ~   cache lookup before API call; cache store after
```

### Testing Requirements

Every new file in `core/` requires an offline test suite. The existing
pattern (framework-free modules with injected clocks) must be preserved.
No test may require a network connection or a live credential.

NI-02 must have a graceful-degradation test confirming that absent
`curl_cffi` produces standard `httpx` behaviour with zero errors.

RE-01 must have a race-cancellation test confirming that the slower tasks
are cancelled and not counted in `request_count`.

### Version String

The target version string is `1.9.0.0`. The `-alpha-v0.1` label is used
in internal tracking; the released tag will drop the label once all P1–P4
items pass CI.

---

## Appendix — Items Explicitly Out of Scope

The following were considered and rejected for v1.9.0.0:

- **Account creation automation** — outside the plugin's domain; the plugin
  manages credentials, not accounts
- **Credential theft / replay of third-party keys** — not a capability
  this plugin will provide under any framing
- **Telemetry exfiltration** — this plugin has no telemetry and will not
  acquire any

---

*Revision 2 · 09/10/2026 · Direct codebase inspection v1.8.1.8*
