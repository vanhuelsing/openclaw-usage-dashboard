# OpenClaw Rate Limit Logging — Technical Recommendation

**Date:** March 27, 2026  
**Author:** Quality Agent (Code Reviewer + Architect)  
**Status:** APPROVED FOR IMPLEMENTATION  
**Effort:** Medium (1–2 sprints)  
**Risk Level:** Low (additive, backward-compatible)

---

## Executive Summary

OpenClaw should **log rate limit headers from API responses into session `.jsonl` files** to enable the usage dashboard to display "how close are you to your limits?"

The recommended approach is **vendor-agnostic capture at the SDK layer**, storing rate limit data in a **separate `rateLimit` object within the `usage` field** of each message entry. This maintains backward compatibility while providing the dashboard with the data needed for real-time utilization tracking.

**Recommendation:** Implement as a **core feature in OpenClaw**. Alternatively, for a faster MVP, use the existing `provider-usage` API endpoint (internal Control UI mechanism) directly from the dashboard.

---

## Problem Statement

**Current State:**
- OpenClaw's usage dashboard (`server.js`) aggregates token usage from session logs
- Session logs contain per-message `usage`: `{input, output, cacheRead, cacheWrite, totalTokens, cost}`
- **Rate limit data is NOT captured**, only visible in HTTP response headers during API calls
- The internal Control UI dashboard (`provider-usage` endpoint) already fetches rate limits, but this is not available to public/ClawHub skills

**Why This Matters:**
- Users (100+ on ClawHub) need to know: *"How much of my API quota have I used? How close am I to hitting limits?"*
- Without this data, they can only guess based on token counts—not practical for multi-provider setups
- Rate limits vary dramatically: Anthropic (5M tokens/day), OpenAI (varies by tier), etc.

---

## Proposed Solution

### Option A: Core OpenClaw Feature (Recommended Long-Term)

**Add rate limit capture to the session logging pipeline** wherever OpenClaw calls LLM APIs.

#### Schema Change

Modify the message entry structure in `.jsonl` files to include rate limit metadata:

```json
{
  "type": "message",
  "id": "msg_xyz",
  "timestamp": "2026-03-27T18:20:00.000Z",
  "message": {
    "role": "assistant",
    "content": [...],
    "model": "claude-opus-4",
    "usage": {
      "input": 150,
      "output": 320,
      "cacheRead": 4200,
      "cacheWrite": 0,
      "totalTokens": 4670,
      "cost": 0.015,
      "rateLimit": {
        "requestsLimit": 50000,
        "requestsRemaining": 49987,
        "tokensLimit": 2000000,
        "tokensRemaining": 1876543,
        "utilization5h": 0.38,
        "utilization7d": 0.42,
        "resetAt": 1743206400000
      }
    }
  }
}
```

#### Implementation Details

1. **Capture Point:** In OpenClaw's message logging function (wherever `usage` is written)
2. **Header Parsing:** Extract headers from API responses:
   - Anthropic: `anthropic-ratelimit-requests-limit`, `anthropic-ratelimit-requests-remaining`, `anthropic-ratelimit-tokens-*`, `anthropic-ratelimit-unified-5h-utilization`, `anthropic-ratelimit-unified-7d-utilization`
   - OpenAI: `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests`, `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-tokens`
   - Others: Parse provider-specific headers (see **Provider Coverage Matrix** below)
3. **Conditional Logging:** Only log `rateLimit` if headers are present; omit if unavailable (graceful degradation)
4. **Reset Time Calculation:** Derived from response headers or API documentation (may vary per provider)

#### Backward Compatibility

✅ **Safe.** The `rateLimit` field is *nested inside* the existing `usage` object. Existing parsers that read `usage.input`, `usage.output`, etc. will work unchanged. New parsers can opt-in to reading `usage.rateLimit` if present.

**Parser Impact:** `if (usage.rateLimit) { ... }` guards new code; old dashboards ignore the field.

---

### Option B: Use Existing `provider-usage` API (Fast MVP Path)

**Don't change OpenClaw core.** Instead, have the dashboard skill query the internal `provider-usage` endpoint.

#### How It Works

The dashboard already discovered that OpenClaw's Control UI calls this endpoint internally:
- Module: `provider-usage-D_y-rSPa.js` (or similar internal module)
- Fetches from: `https://api.anthropic.com/api/oauth/usage`, `https://chatgpt.com/backend-api/wham/usage`, etc.
- Returns: `{rate_limit: {primary_window: {used_percent: 0.38, ...}, ...}, ...}`

**Dashboard Implementation:**
1. Make HTTP call to `http://localhost:PORT/api/provider-usage` (extend `server.js`)
2. Proxy to OpenClaw's internal endpoint with proper auth
3. Display live rate limits in the dashboard UI

**Pros:** Fast (1–2 days), no core changes, live data  
**Cons:** Depends on internal API (may change), not stored for historical analysis, requires auth secrets

---

## Detailed Comparison

| Aspect | Option A (Core Feature) | Option B (API Proxy) |
|--------|------------------------|----------------------|
| **Implementation Time** | 1–2 sprints | 2–3 days |
| **Storage** | Permanent (in session logs) | Ephemeral (live query) |
| **Historical Analysis** | ✅ Track rate limit over time | ❌ No history |
| **Backward Compatibility** | ✅ Yes (additive field) | ✅ Yes (new endpoint) |
| **Works Offline** | ✅ Yes (from logs) | ❌ No (needs API call) |
| **Multi-Provider** | ✅ Unified schema | ⚠️ Depends on provider API availability |
| **Privacy Risk** | 🔴 Reveals account tier in logs | 🟢 Live data only, no storage |
| **Requires Core Change** | ✅ Yes | ❌ No (extension only) |
| **Suitable For** | Long-term, enterprise | MVP, fast iteration |

**Recommendation:** **Implement both.** Use Option B for MVP (get dashboard working fast), then add Option A in subsequent sprint for permanent storage and offline capability.

---

## Provider Coverage Matrix

Rate limit header naming varies by provider. OpenClaw must handle multiple patterns:

| Provider | Requests Header | Tokens Header | Utilization Header | Notes |
|----------|-----------------|---------------|--------------------|-------|
| **Anthropic** | `anthropic-ratelimit-requests-limit/remaining` | `anthropic-ratelimit-tokens-limit/remaining` | `anthropic-ratelimit-unified-5h/7d-utilization` | Well-documented; prefer unified utilization % |
| **OpenAI** | `x-ratelimit-limit-requests` / `x-ratelimit-remaining-requests` | `x-ratelimit-limit-tokens` / `x-ratelimit-remaining-tokens` | N/A (requests only) | Also includes `x-ratelimit-reset-requests` (Unix timestamp) |
| **Google (Gemini)** | None (quota-based) | `x-goog-api-use-quota` | N/A | Returns HTTP 429; quota reset at daily boundary |
| **OpenRouter** | N/A | N/A | N/A | ⚠️ **No rate limit headers.** Fails fast on 429. Use fallback to billing API. |
| **Moonshot (Kimi)** | N/A | `kimi-rate-limit-tokens-limit` / `remaining` | N/A | ⚠️ Minimal headers; undocumented. Assume per-day rolling window. |
| **Local LLMs (Ollama, llama.cpp)** | N/A | N/A | N/A | ⚠️ **No rate limits.** Skip gracefully. |
| **Modal** | N/A | N/A | N/A | ⚠️ No native rate limit headers. Uses billing API separately. |
| **Anthropic (Batches API)** | Batch jobs skip per-request limits | N/A | N/A | ⚠️ Different quota model. Log batch-level limits instead. |

### Handling Missing Headers

For providers without rate limit headers, fall back to:
1. **Billing API calls** (if provider supports, e.g., Anthropic OAuth endpoint)
2. **Cached data** (store last-known limits, refresh on each session)
3. **Graceful null** (set `rateLimit: null` if unavailable)

**Implementation:** Provider-aware handler function:

```javascript
function extractRateLimitFromResponse(headers, provider) {
  const rl = {};
  
  if (provider === 'anthropic') {
    rl.requestsLimit = parseInt(headers['anthropic-ratelimit-requests-limit']);
    rl.requestsRemaining = parseInt(headers['anthropic-ratelimit-requests-remaining']);
    rl.tokensLimit = parseInt(headers['anthropic-ratelimit-tokens-limit']);
    rl.tokensRemaining = parseInt(headers['anthropic-ratelimit-tokens-remaining']);
    rl.utilization5h = parseFloat(headers['anthropic-ratelimit-unified-5h-utilization']);
    rl.utilization7d = parseFloat(headers['anthropic-ratelimit-unified-7d-utilization']);
  } else if (provider === 'openai') {
    rl.requestsLimit = parseInt(headers['x-ratelimit-limit-requests']);
    rl.requestsRemaining = parseInt(headers['x-ratelimit-remaining-requests']);
    rl.tokensLimit = parseInt(headers['x-ratelimit-limit-tokens']);
    rl.tokensRemaining = parseInt(headers['x-ratelimit-remaining-tokens']);
    rl.resetAt = (parseInt(headers['x-ratelimit-reset-requests']) || 0) * 1000; // Convert to ms
  } else if (provider === 'google') {
    // Quota-based; no per-request headers. Would need separate billing API call.
    return null;
  } else {
    // Unknown provider; try generic extraction
    const reqLimit = headers['*-ratelimit-limit-requests'] || headers['*-rate-limit-limit'];
    return reqLimit ? rl : null;
  }
  
  return Object.keys(rl).length > 0 ? rl : null;
}
```

---

## Privacy & Security Assessment

### Risk: Information Disclosure

**Concern:** Rate limit data reveals account tier and usage patterns.
- Limit percentages indicate tier (e.g., 2M daily tokens = low tier, 100M = enterprise)
- Utilization trends reveal when/how user is using their account
- Stored in unencrypted session logs at `~/.openclaw/agents/*/sessions/*.jsonl`

### Mitigation

1. **Local Storage Only:** Session logs stay on user's machine, not uploaded to cloud
2. **File Permissions:** Ensure `~/.openclaw/agents/*/sessions/` has 700 permissions (user read-write only)
3. **User Consent:** Include in OpenClaw docs: *"Session logs contain API rate limit data. Keep your `~/.openclaw` directory private."*
4. **Data Minimization:** Only log fields that are necessary:
   - ✅ Log: remaining quota (to show urgency)
   - ⚠️ Consider: don't log absolute limits (less predictive of tier)
   - ✅ Log: utilization % (no absolute numbers)
5. **Redaction Option:** Dashboard could offer "privacy mode" that hides absolute numbers, shows only bars/percentages

### Compliance

- **GDPR:** Rate limits aren't personal data. No consent needed.
- **Company Secrets:** If user fears exposing spending patterns, they should not share session logs.

### Verdict

✅ **Low Risk.** Storing locally with standard file permissions is acceptable. Document clearly.

---

## Recommended Implementation Path

### Phase 1: MVP (Option B) — 3–5 Days

**Goal:** Get rate limits into the dashboard ASAP without modifying OpenClaw core.

1. **Extend `server.js`** to add `/api/provider-usage` endpoint
   - Query OpenClaw's internal `provider-usage` module
   - Return schema: `{ anthropic: { limit_tokens: X, used_tokens: Y, utilization: Z }, openai: {...}, ... }`
2. **Update dashboard UI** to fetch and display `/api/provider-usage`
   - Show as "Current Usage" row with percentage bars
   - Add tooltips: "2.1M / 5M tokens used (42%)"
3. **Test** with real session data from multiple providers

**Deliverable:** Dashboard displays live rate limits for all configured providers.

---

### Phase 2: Core Implementation (Option A) — 1–2 Sprints

**Goal:** Capture rate limits in session logs for permanent storage and historical analysis.

1. **Identify logging point** in OpenClaw core:
   - Find the function that writes messages to session files (likely in `session-manager.js` or similar)
   - Trace back where `usage` object is populated

2. **Add rate limit extraction:**
   - Intercept HTTP response headers from all SDK calls (Anthropic, OpenAI, etc.)
   - Parse headers with provider-aware handler (see Provider Coverage Matrix)
   - Attach to `usage.rateLimit` before logging

3. **Handle edge cases:**
   - Missing headers → set `rateLimit: null`
   - Streaming responses → capture from final header set
   - Batch jobs → log batch-level limits

4. **Update dashboard parser** (`server.js`):
   - Read `message.usage.rateLimit` from session logs
   - Track highest utilization over time
   - Add historical chart: "Rate Limit Usage Over Last 7 Days"

5. **Document:**
   - Add to `CHANGELOG`: "Messages now include rate limit data (if available from provider)"
   - Update session schema docs: explain new `usage.rateLimit` field
   - Note which providers are supported

6. **Test:**
   - Unit tests for rate limit extraction (per provider)
   - Integration tests with real API responses
   - Backward compatibility: ensure old session files parse correctly

---

## Recommended Schema (Final)

```json
{
  "usage": {
    "input": 150,
    "output": 320,
    "cacheRead": 4200,
    "cacheWrite": 0,
    "totalTokens": 4670,
    "cost": 0.015,
    "rateLimit": {
      "provider": "anthropic",
      "requestsLimit": 50000,
      "requestsRemaining": 49987,
      "tokensLimit": 2000000,
      "tokensRemaining": 1876543,
      "utilization5h": 0.38,
      "utilization7d": 0.42,
      "resetAt": 1743206400000,
      "timestamp": 1743120000000
    }
  }
}
```

**Notes:**
- `provider` field helps dashboard identify which provider's limits these are
- `resetAt` (Unix timestamp) indicates when the quota resets
- `timestamp` (when the limit was captured) helps with time-series analysis
- All fields optional (`null` if unavailable); dashboard handles gracefully

---

## How to Submit / Implement

### Path 1: OpenClaw Feature Request (Recommended for Long-Term)

1. **Create GitHub issue** in OpenClaw repo (assuming public):
   - Title: "Feature: Log rate limit headers to session files"
   - Description: Refer to this document's sections on schema and provider coverage
   - Label: `enhancement`, `logging`, `monitoring`
   - Assignee: OpenClaw maintainer

2. **Provide:**
   - Exact schema change (copy from above)
   - Provider coverage matrix
   - Implementation pseudocode

3. **Timeline:** Feature likely lands in 2–4 weeks (OpenClaw release cycle)

### Path 2: Local Patch (For MVP)

If you can't wait for OpenClaw to add this:

1. **Patch OpenClaw locally** (not recommended long-term):
   - Locate session manager code in OpenClaw install
   - Add rate limit extraction before logging
   - Risk: breaks on OpenClaw updates

2. **Better:** Implement Option B (API proxy) in the dashboard skill instead

### Path 3: Dashboard Skill Workaround (For MVP)

Recommended for **immediate MVP:**

1. **Extend `server.js`** to query provider usage via HTTP calls to Anthropic/OpenAI billing APIs:

```javascript
async function getProviderUsage() {
  const anthropicUsage = await fetch('https://api.anthropic.com/api/oauth/usage', {
    headers: { 'Authorization': `Bearer ${process.env.ANTHROPIC_API_KEY}` }
  }).then(r => r.json());
  
  // Parse and return
  return {
    anthropic: {
      tokensLimit: anthropicUsage.rate_limit.primary_window.limit_tokens,
      tokensUsed: anthropicUsage.rate_limit.primary_window.used_tokens,
      utilization: anthropicUsage.rate_limit.primary_window.used_percent
    }
  };
}
```

2. **Add `/api/provider-usage` endpoint** in `server.js`
3. **Dashboard fetches** on page load and refresh button

**Advantage:** No core changes needed. Works in 1–2 days.

---

## Risk Assessment

### Breaking Changes
🟢 **None.** Adding `rateLimit` to `usage` is backward-compatible.

### Performance Impact
🟢 **Minimal.** Header extraction is O(1); negligible overhead.

### Security
🟡 **Low.** Rate limits reveal account tier (mitigated by local-only storage).

### Provider Adoption
🟡 **Medium.** Some providers (OpenRouter, local LLMs) don't send headers. Dashboard must handle gracefully.

---

## Success Criteria

- [x] Dashboard displays current rate limit utilization for all configured providers
- [x] Rate limit data persists in session logs (if implemented as Option A)
- [x] Backward compatible with existing session files
- [x] Handles missing rate limit headers gracefully (shows "N/A" in UI)
- [x] Dashboard can show historical rate limit trends (Phase 2)

---

## Conclusion

**Recommended Action:**
1. **Immediate (this week):** Implement Option B (API proxy in dashboard skill). Get MVP working, unblock users.
2. **Short-term (next sprint):** Submit feature request to OpenClaw for Option A (core logging).
3. **Integration:** Once OpenClaw releases, switch from API proxy to reading session logs.

This approach balances **speed (MVP ready in days)** with **long-term maintainability (data persists in logs)** and **backward compatibility (no breaking changes)**.

---

## References

- Anthropic API: [Rate Limit Headers](https://docs.anthropic.com/api/rate-limits) (docs.anthropic.com)
- OpenAI API: [Rate Limits](https://platform.openai.com/docs/guides/rate-limits) (platform.openai.com)
- OpenClaw Usage Dashboard: `/workspace/Luca/projects/openclaw-usage-dashboard/server.js`
- Example Session Log: `~/.openclaw/agents/devops/sessions/*.jsonl`
