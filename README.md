# Astrel SDK Conformance Specification

Guide and index for building Astrel remote-config SDKs.

This document is **normative**. Any SDK that claims to be an Astrel SDK MUST implement everything marked **MUST** here and SHOULD implement everything marked **SHOULD**. It is language agnostic: it defines behaviour and contracts, not syntax. A correct Go SDK, a correct Swift SDK, and a correct PHP SDK must all resolve the same config + context to the same value, byte for byte.

The contract that SDKs consume is the published config payload defined by the JSON Schema at:

```
https://schema.astrel.io/v1/astrel-config.schema.json
```

The keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used as defined in RFC 2119.

---

## Contents

1. [The model](#1-the-model)
2. [Conformance levels](#2-conformance-levels)
3. [Schema version handling](#3-schema-version-handling)
4. [The evaluation context](#4-the-evaluation-context)
5. [Evaluation engine](#5-evaluation-engine)
6. [The lottery algorithm](#6-the-lottery-algorithm)
7. [Type-safe accessors](#7-type-safe-accessors)
8. [Error handling and fallbacks](#8-error-handling-and-fallbacks)
9. [Config delivery](#9-config-delivery)
10. [Lifecycle and runtime](#10-lifecycle-and-runtime)
11. [Telemetry and exposure](#11-telemetry-and-exposure)
12. [Conformance test vectors](#12-conformance-test-vectors)
13. [Public API shape](#13-public-api-shape)
14. [Open questions](#14-open-questions)

---

## 1. The model

Astrel publishes a single JSON **config payload**. SDKs ingest that payload and, given an **evaluation context** (a single opaque identifier supplied by the host), resolve the value of an **aspect**.

- **Aspect** — a single named piece of configuration (a flag, a value, a JSON blob). Keyed by slug. Has a declared `type` and a `default`.
- **Override** — a per-aspect marker (`overridden`) saying the backend has pinned this aspect to a forced value. When set, the backend has already baked the forced value into `default` and removed the lottery. The SDK does no override logic of its own; it surfaces that the value is forced. Used for kill-switches and incident response.
- **Lottery** — deterministic percentage rollout. Hashes the host-supplied identifier into a bucket. The same identifier always lands in the same bucket.
- **Default** — the base value when nothing else matches.

Every aspect is resolved through the same fixed order:

```
Override  →  Lottery  →  Default
```

The first layer that produces a value wins. This order is fixed by the schema description and MUST NOT be configurable.

> **v1 has no explicit per-identifier targeting.** There is no allowlist/cohort layer: an aspect is either pinned by a backend override, bucketed by the lottery, or left at its default. "Turn this on for exactly these users" is not expressible in the payload; the host must achieve it another way (its own gating, or a dedicated single-bucket lottery). This is a deliberate v1 scope decision.

---

## 2. Conformance levels

An SDK is **conformant** if it:

1. Passes the full [golden test vector suite](#12-conformance-test-vectors) (Section 12) — this is the hard gate.
2. Implements the evaluation engine (Section 5) and lottery (Section 6) exactly.
3. Implements schema version handling (Section 3).
4. Implements type-safe accessors and the fallback rules (Sections 7–8).

Config delivery (Section 9), lifecycle (Section 10), and telemetry (Section 11) are **required for a production SDK** but are decoupled from the evaluation core. A minimal "bring your own payload" SDK MAY ship with only Sections 3–8 implemented, provided it documents that delivery is the host's responsibility.

The golden vectors are the source of truth. Where prose in this document and a vector disagree, the vector wins and the prose is a bug — report it.

---

## 3. Schema version handling

The payload carries a top-level `schema_version` integer. For this spec that value is `1`.

- The SDK MUST read `schema_version` before evaluating anything.
- An SDK built for major version `N` MUST accept payloads where `schema_version == N`.
- It MUST reject (refuse to load, keep last-known-good, surface an error) payloads where `schema_version > N`. A higher major means breaking changes the SDK cannot understand.
- It MUST reject payloads where `schema_version < N` unless the SDK explicitly advertises multi-major support.
- Rejection MUST NOT crash the host application. It MUST be observable (Section 11) and MUST leave any previously loaded good config in place (Section 10).

Per the schema repo's versioning policy, a new major (`/v2/`, …) is cut only for breaking changes: removing/renaming a field, changing a type, making an optional field required, tightening validation, or **adding a value to an enum the SDK matches exhaustively** (e.g. the lottery `algorithm`). This last point is why Section 6 requires SDKs to fail safe on an unrecognised algorithm rather than guess.

SDKs MUST pin to a major-version schema URL. They MUST NOT fetch an unversioned or "latest" schema URL at runtime.

---

## 4. The evaluation context

Evaluation is driven entirely by the payload plus one runtime input. There are no other inputs.

| Input | Type | Required | Used by |
|-------|------|----------|---------|
| `identifier` | string | for the lottery only | lottery seed (Section 6) |

The `identifier` is an opaque string chosen by the host (a user ID, team ID, account ID, device ID, whatever the host wants the rollout bucketed by). Astrel does not interpret it.

Rules:

- The `identifier` is required only to run the **lottery** layer. If it is absent or empty, the lottery layer MUST be skipped (treated as "no match") and resolution falls through to the default. The SDK MUST NOT throw.
- **Overrides do not use the identifier.** When `aspect.overridden` is true the value is forced regardless of whether an identifier is supplied (Section 5.1). An overridden aspect resolves even when no identifier is passed.
- The identifier is an opaque string. The SDK MUST NOT normalise, trim, lowercase, or otherwise transform it. `"User-1"` and `"user-1"` are different identifiers. The lottery seed is built from the raw string (Section 6).

The override marker takes **no** runtime input; its state lives entirely in the payload (see Section 5.1). The host does not pass override state in.

---

## 5. Evaluation engine

To resolve aspect `A` for a host-supplied `identifier` (which may be absent):

### 5.1 Override (forced value)

If `aspect.overridden` is `true`, the aspect is pinned to a forced value. **Return `aspect.default` immediately** and report the deciding layer as `override` (Section 11). Do not run the lottery.

When the backend pins an aspect it sets `default` to the forced value and omits `lottery`, so even an SDK that ignored the flag would return the forced value via the default layer. Checking the flag first makes the override authoritative (it wins even if a `lottery` is erroneously present) and lets telemetry distinguish a forced value from one that merely fell through to its baseline default.

If `overridden` is absent or `false`, fall through.

> There is no trigger object, priority list, or runtime input in the SDK. The override is decided entirely in the backend. "Flipping" it is a publish operation: the backend republishes the aspect with `overridden: true`, the forced value in `default`, and no `lottery`. Every SDK picks it up on its next config load. This is what makes it usable as a kill-switch.

### 5.2 Lottery (deterministic rollout)

If `aspect.lottery` is present, run the lottery (Section 6) and **return the selected bucket's value**. The lottery always produces a value when present (its weights sum to 10000, covering the whole range), so when an aspect has a lottery, resolution never reaches the default for a valid context.

If `aspect.lottery` is absent, fall through.

### 5.3 Default

Return `aspect.default`. Every aspect declares one; this layer always produces a value.

### 5.4 Worked example

```jsonc
{
  "schema_version": 1,
  "published_at": "2026-06-13T10:00:00Z",
  "lottery": {
    "algorithm": "sha256_mod10000",
    "seed_format": "{identifier}:{aspect_slug}"
  },
  "aspects": {
    "new-checkout": {
      "type": "boolean",
      "default": false,
      "lottery": {
        "buckets": [
          { "value": false, "weight": 5000 },
          { "value": true,  "weight": 5000 }
        ]
      }
    }
  }
}
```

- Identifier `user-789` → not overridden → lottery (Section 6 shows `user-789:new-checkout` → point 7064 → bucket index 1) → **`true`**.
- Identifier `user-123` → not overridden → lottery (`user-123:new-checkout` → point 1515 → bucket index 0) → **`false`**.
- No identifier supplied → not overridden → lottery skipped → default → **`false`**.

To force `new-checkout` off for everyone (e.g. an incident kill-switch), the backend republishes the aspect with the lottery stripped and the value pinned:

```jsonc
"new-checkout": {
  "type": "boolean",
  "default": false,
  "overridden": true
}
```

Now every context resolves to **`false`** via the `override` layer, identifier irrelevant and the lottery gone.

---

## 6. The lottery algorithm

This is the part that MUST be bit-identical across every SDK. Implement it exactly; do not improvise.

### 6.1 Algorithm: `sha256_mod10000`

The global `config.lottery.algorithm` declares the algorithm. v1 defines exactly one value: `sha256_mod10000`. The SDK MUST check this value and MUST fail safe (treat the lottery as unavailable and fall through to the default, while surfacing an error per Section 11) if it sees any other value. It MUST NOT guess.

### 6.2 Building the seed

The seed template is `{identifier}:{aspect_slug}`, taken from `config.lottery.seed_format`.

- `identifier` is the opaque string the host passes into the accessor at resolution time (Section 4).
- `aspect_slug` is the aspect's key.
- The two are joined with a single ASCII colon `:`.

Example: aspect `new-checkout`, `identifier = "user-789"` → seed string `user-789:new-checkout`.

If no identifier is supplied, the lottery layer is skipped (Section 4) and resolution falls through to the default.

### 6.3 Hash to a point

1. Encode the seed string as **UTF-8** bytes.
2. Compute the **SHA-256** digest (32 bytes).
3. Take the **first 4 bytes** and read them as a **big-endian unsigned 32-bit integer**.
4. Take that integer **modulo 10000**. The result `point` is in the inclusive range `[0, 9999]`.

Reference (pseudocode):

```
seed   = identifier + ":" + aspect_slug
digest = SHA256(UTF8(seed))            // 32 bytes
u32    = (digest[0] << 24) | (digest[1] << 16) | (digest[2] << 8) | digest[3]   // big-endian
point  = u32 mod 10000                 // 0..9999
```

Watch the language traps: use an **unsigned** 32-bit read (in languages without unsigned ints, mask to 64 bits before the modulo so the sign bit doesn't poison the result), and confirm big-endian byte order.

### 6.4 Selecting the bucket

`lottery.buckets` is an **ordered** array (minimum 2 entries). Each has a `value` and an integer `weight` in basis points. **Weights MUST sum to 10000.** Selection uses half-open cumulative intervals:

```
running = 0
for bucket in buckets:            // in array order
    running += bucket.weight
    if point < running:
        return bucket.value
// unreachable for valid payloads (weights sum to 10000, point <= 9999)
```

So bucket *i* owns the half-open interval `[sum_of_prior_weights, sum_of_prior_weights + weight_i)`. The lower bound is inclusive, the upper bound exclusive. A `point` of exactly `5000` against buckets `[5000, 5000]` lands in the **second** bucket, because the first owns `[0, 5000)`.

Bucket order in the array is significant and MUST be preserved as published. SDKs MUST NOT sort, dedupe, or reorder buckets.

### 6.5 Determinism guarantees

Given the same seed and the same ordered buckets, every SDK MUST return the same bucket. The hash depends only on the UTF-8 seed bytes, so it is stable across languages, platforms, architectures, and time. Because the seed includes the aspect slug, the same user gets independently-distributed buckets across different aspects (no correlation between rollouts).

### 6.6 Verified vectors

These were computed with a reference SHA-256 implementation. Your SDK MUST reproduce the `u32` and `point` columns exactly. (Full machine-readable set in Section 12.)

| Seed | u32 (big-endian) | point | 2-way `[5000,5000]` → `[false,true]` | 3-way `[3334,3333,3333]` → `[a,b,c]` |
|------|------------------|-------|------|------|
| `user-123:new-checkout`   | 667561515  | 1515 | false (`[0,5000)`) | a (`[0,3334)`) |
| `user-456:new-checkout`   | 1173581668 | 1668 | false | a |
| `user-789:new-checkout`   | 3985347064 | 7064 | true (`[5000,10000)`) | c (`[6667,10000)`) |
| `team-42:beta-dashboard`  | 3993341872 | 1872 | false | a |
| `user-123:beta-dashboard` | 3732749424 | 9424 | true | c |

Note `user-123` lands in bucket `false`/`a` for `new-checkout` (point 1515) but `true`/`c` for `beta-dashboard` (point 9424) — different aspect slug, independent draw. That decorrelation is a required property.

---

## 7. Type-safe accessors

Each aspect declares a `type`: one of `boolean`, `string`, `number`, `json`. The SDK MUST expose type-specific accessors and MUST give the caller a way to supply a fallback default. Idiomatic naming per language, e.g.:

```
get_boolean(slug, context, fallback) -> boolean
get_string(slug, context, fallback)  -> string
get_number(slug, context, fallback)  -> number
get_json(slug, context, fallback)    -> object | array
```

Rules:

- `boolean` → native boolean. `string` → native string. `number` → native numeric (see below). `json` → the parsed object or array.
- **Number handling**: JSON numbers are double-precision. The SDK MUST preserve numeric value faithfully and SHOULD offer integer and float access where the language distinguishes them. It MUST NOT silently truncate a float to an int.
- **Type mismatch**: if the resolved value's runtime type does not match the accessor called (e.g. `get_boolean` on a `string` aspect, or a lottery bucket value whose type differs from the aspect's declared `type`), the SDK MUST return the caller's `fallback`, MUST NOT throw, and MUST surface the mismatch via the error channel (Section 8/11).
- **Unknown slug**: if the aspect is not present in the payload, return the caller's `fallback` (Section 8).
- A generic/untyped accessor MAY be provided in addition, but the typed accessors are the required surface.

The declared `type` is advisory metadata for the accessor layer. Resolution (Sections 5–6) is type-agnostic — it returns whatever JSON value the winning layer holds. Type checking happens at the accessor boundary, not during resolution.

---

## 8. Error handling and fallbacks

The guiding rule: **an SDK MUST NEVER throw into, block, or crash the host application during evaluation.** Config systems sit in hot paths. Every failure mode degrades to a safe value.

Required behaviour by failure:

| Failure | Behaviour |
|---------|-----------|
| No config loaded yet | Return the caller's `fallback`. Surface "not ready". |
| Payload fails schema validation | Reject the payload, keep last-known-good, surface error. Never partially apply. |
| `schema_version` newer than SDK supports | Reject payload, keep last-known-good (Section 3). |
| Unknown aspect slug | Return caller's `fallback`. |
| Type mismatch at accessor | Return caller's `fallback`, surface error (Section 7). |
| Unknown lottery `algorithm` | Skip lottery layer, fall through to default, surface error (Section 6.1). |
| Missing `identifier` | Skip the lottery layer, fall through to default; an `overridden` aspect still resolves to its forced value. Surface a usage error (Section 4). |

There are two distinct "defaults" and they MUST NOT be conflated:

- **`aspect.default`** — from the payload, the last layer of resolution (Section 5.4).
- **caller `fallback`** — passed into the accessor, used only when resolution cannot run at all (no config, unknown slug, type mismatch).

The SDK SHOULD validate the full payload against the JSON Schema at load time (Section 9) so malformed configs are rejected wholesale rather than discovered mid-evaluation.

---

## 9. Config delivery

How an SDK obtains the payload. A production SDK MUST implement a fetch-and-cache loop; the evaluation core (Sections 3–8) does not depend on it.

- **Transport**: the SDK MUST fetch the published payload over HTTPS from the configured Astrel config endpoint (a CDN-backed URL). It MUST verify TLS. The payload is a static, cacheable object at the CDN edge, so the SDK SHOULD lean on edge caching rather than hitting an origin on every poll.
- **Validation on ingest**: every fetched payload MUST be validated (schema version per Section 3; full JSON Schema validation SHOULD be done) before it replaces the active config. An invalid payload MUST be discarded and the previous good config retained.
- **Caching / conditional requests**: the SDK SHOULD use HTTP caching validators (`ETag` / `If-None-Match`, or `Last-Modified` / `If-Modified-Since`) and treat `304 Not Modified` as "no change". This keeps polling cheap.
- **Polling**: a production SDK MUST re-fetch the payload from the CDN on a recurring schedule so config changes propagate without a host restart. The interval MUST be configurable with a documented sane default (e.g. 30–60s). The poll loop MUST run on a background timer, MUST NOT block evaluation, and MUST start after init and stop on shutdown (Section 10). Each poll SHOULD send conditional-request validators (per the caching bullet) so an unchanged payload costs only a cheap `304` at the CDN edge. The SDK SHOULD apply small random jitter to the interval (e.g. ±10–20%) so a fleet of instances does not synchronise into a thundering herd against the CDN. It MAY additionally expose an explicit `refresh()` for on-demand fetches and MAY support push/streaming where the platform offers it, but the interval poll is the baseline and MUST exist.
- **Backoff**: on fetch failure the SDK MUST back off (exponential with jitter SHOULD be used) and MUST keep serving the last-known-good config meanwhile. Transient fetch failures MUST NOT clear loaded config.
- **Offline / cold start**: the SDK SHOULD persist the last-known-good payload to local storage so a restart with no network still evaluates against the last good config rather than only caller fallbacks. On first ever run with no network and no cache, all accessors return caller fallbacks.
- **Atomicity**: see Section 10 — a new payload MUST be swapped in atomically.

Only the **config payload** is polled. The JSON Schema itself (the pinned `/v1/` contract, Section 3) is a build-time dependency and MUST NOT be re-fetched on the poll loop; a new schema major is adopted by shipping a new SDK build, not by polling a "latest" URL.

`published_at` is informational provenance. The SDK MAY expose it and MAY use it to ignore an older payload than the one currently loaded, but MUST NOT use it for cache freshness in place of HTTP validators.

---

## 10. Lifecycle and runtime

- **Initialisation**: the SDK MUST expose an explicit init that takes configuration (endpoint, SDK key/credentials, poll interval, logger/telemetry hooks). Init SHOULD support an async "wait until first config is ready" path and SHOULD NOT block the host indefinitely — a bounded timeout that then serves caller fallbacks is the correct degrade.
- **Atomic swap**: when a new valid payload arrives, the SDK MUST replace the active config atomically. An in-flight evaluation MUST see either the whole old config or the whole new config, never a mix. Treat the loaded config as immutable and swap a pointer/reference.
- **Thread safety**: evaluation MUST be safe to call concurrently from many threads/goroutines/tasks. Reads MUST NOT require the caller to hold a lock. Evaluation MUST be free of shared mutable state beyond the atomically-swapped config reference.
- **No surprise I/O on the hot path**: `get_*` accessors MUST be pure, in-memory, side-effect-free with respect to config state (telemetry emission per Section 11 excepted, and that MUST be non-blocking). No network or disk I/O during evaluation.
- **Last-known-good**: the currently active config persists until a *valid* replacement arrives. Errors, rejects, and fetch failures never downgrade it (Sections 8–9).
- **Shutdown**: the SDK SHOULD expose a clean shutdown that stops polling and flushes any buffered telemetry.
- **Determinism over time**: for a fixed loaded config and context, repeated evaluations MUST return the same value. Nothing in resolution may depend on wall-clock time, call count, or randomness — the lottery is the only "randomness" and it is a pure hash.

---

## 11. Telemetry and exposure

To measure rollouts, SDKs report **exposure events**: a record that a given context was exposed to a given resolved value, and which layer decided it.

- The SDK SHOULD emit an exposure event each time an aspect is evaluated for a context. An exposure event MUST include at least: aspect slug, resolved value (or a stable digest of it), the deciding layer (`override` | `lottery` | `default`), the identifier used (if any), and a timestamp. When the layer is `lottery` it SHOULD include the selected bucket index.
- Emission MUST be non-blocking and MUST NOT affect the resolved value. Telemetry failure MUST NOT fail evaluation.
- The SDK SHOULD batch and flush exposures on an interval and on shutdown, with bounded buffering and a drop policy under backpressure (dropping telemetry is always preferable to blocking the host or growing memory unboundedly).
- The SDK SHOULD de-duplicate repeated identical exposures within a window to control volume, and SHOULD make this configurable.
- The SDK MUST provide a structured logging/diagnostics hook so a host can observe rejects, type mismatches, unknown algorithms, and fetch failures (Section 8).
- All telemetry SHOULD be disableable for privacy-sensitive or offline deployments.

The exposure schema and ingestion endpoint are an Astrel platform concern; pin the exact wire format in the platform docs and keep this section's required fields as the floor.

---

## 12. Conformance test vectors

This is the hard gate. Cross-language consistency is enforced by a **shared, language-agnostic golden-vector suite**, not by prose. Every SDK MUST pass it before release.

### 12.1 Structure

Vectors live in this repo (proposed `vectors/`) as JSON so any language can load them. Two kinds:

**(a) Lottery vectors** — isolate the hash so a port can be validated before the whole engine exists:

```json
{
  "algorithm": "sha256_mod10000",
  "cases": [
    { "identifier": "user-123", "aspect_slug": "new-checkout",   "seed": "user-123:new-checkout",   "u32": 667561515,  "point": 1515 },
    { "identifier": "user-456", "aspect_slug": "new-checkout",   "seed": "user-456:new-checkout",   "u32": 1173581668, "point": 1668 },
    { "identifier": "user-789", "aspect_slug": "new-checkout",   "seed": "user-789:new-checkout",   "u32": 3985347064, "point": 7064 },
    { "identifier": "team-42",  "aspect_slug": "beta-dashboard", "seed": "team-42:beta-dashboard",  "u32": 3993341872, "point": 1872 },
    { "identifier": "user-123", "aspect_slug": "beta-dashboard", "seed": "user-123:beta-dashboard", "u32": 3732749424, "point": 9424 }
  ]
}
```

**(b) End-to-end resolution vectors** — a full payload, a context, and the expected resolved value and deciding layer:

```json
{
  "payload": { "schema_version": 1, "published_at": "2026-06-13T10:00:00Z", "...": "..." },
  "cases": [
    { "aspect": "killed-feature", "identifier": "anyone",   "expected": false, "via": "override" },
    { "aspect": "new-checkout",   "identifier": "user-789", "expected": true,  "via": "lottery", "bucket": 1 },
    { "aspect": "new-checkout",   "identifier": "user-123", "expected": false, "via": "lottery", "bucket": 0 },
    { "aspect": "new-checkout",   "identifier": "",         "expected": false, "via": "default" }
  ]
}
```

### 12.2 Required coverage

The suite MUST include cases for: each layer winning in isolation; an overridden aspect returning its forced value and skipping the lottery; an overridden aspect resolving with no identifier supplied; missing identifier skipping the lottery and falling through to default; lottery bucket-boundary cases (a `point` of exactly `0`, exactly one below a boundary, and exactly on a boundary); each aspect `type`; type-mismatch falling back; unknown slug falling back; unknown lottery algorithm failing safe; and `schema_version` too-new being rejected.

### 12.3 Boundary cases worth pinning explicitly

Because off-by-one in bucket selection is the most likely cross-SDK divergence, pin synthetic point values directly (independent of any hash) against buckets `[{"value":"a","weight":5000},{"value":"b","weight":5000}]`:

| point | expected bucket | why |
|-------|-----------------|-----|
| 0 | a | bottom of `[0,5000)` |
| 4999 | a | top of `[0,5000)` |
| 5000 | b | bottom of `[5000,10000)` — boundary is exclusive on the low bucket |
| 9999 | b | top of range |

### 12.4 Process

- The vector files are the single source of truth and are versioned alongside the schema major.
- A change to evaluation/lottery behaviour MUST update the vectors in the same change.
- Each SDK's CI MUST run the vectors on every build and MUST fail the build on any divergence.

---

## 13. Public API shape

Language-agnostic surface every SDK SHOULD expose (names idiomatic per language):

```
Client.init(options) -> Client            // endpoint, key, poll interval, hooks
Client.is_ready() -> boolean              // first valid config loaded?
Client.refresh() -> void | future         // force a fetch
Client.get_boolean(slug, identifier, fallback) -> boolean
Client.get_string(slug, identifier, fallback)  -> string
Client.get_number(slug, identifier, fallback)  -> number
Client.get_json(slug, identifier, fallback)    -> object | array
Client.get_details(slug, identifier) -> { value, type, via, bucket? }   // for debugging/exposure
Client.shutdown() -> void | future        // stop polling, flush telemetry

identifier := string   // opaque, host-chosen; optional. omit or empty skips the lottery
```

`get_details` (or equivalent) SHOULD return the deciding layer so hosts can debug resolution and so exposure events (Section 11) can be assembled. The typed `get_*` accessors are the required surface; everything else is strongly recommended.

---

## 14. Open questions

Flag for the Astrel platform team before SDKs ship:

1. **Config endpoint & auth** — Section 9 assumes HTTPS fetch with an SDK key. The exact endpoint, auth scheme (header? query? signed?), and whether configs are per-environment/per-project need pinning.
2. **Exposure wire format** — Section 11 defines the required fields; the exact event schema and ingestion endpoint are not yet specified.
3. **Override provenance** — the payload exposes only a boolean `overridden`; it does not say *why* an aspect is pinned (which incident, who pinned it, when). If hosts need that provenance for audit or telemetry, it must be added to the schema or carried out-of-band. Confirm the bare boolean is enough for v1.
4. **Number precision policy** — confirm whether `number` aspects are always JSON doubles or whether integer aspects need a distinct contract for languages that care.
5. **Default poll interval & SLA** — Section 9 suggests 30–60s; the platform should set the canonical default and any rate limits.
```
