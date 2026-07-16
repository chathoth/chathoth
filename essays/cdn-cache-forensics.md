# CDN Cache Forensics: Failure Modes Nobody Documents

*Field notes from debugging a multi-tier caching stack (Akamai → Fastly → Apache/Dispatcher → AEM publish) where "caching is enabled" and yet the CDN bill kept growing. Every failure mode below was observed in production-grade infrastructure and verified with header evidence. Hostnames and paths are anonymized.*

> **TL;DR** — Most cache debugging guides tell you how to *enable* caching. This one is about the ways caching **lies to you after it's enabled**: stale header snapshots that outlive config fixes, revalidation loops that preserve the very state you're trying to repair, TTLs that silently count down to zero, and one binary fragmenting into five cache keys. Plus the forensic techniques to catch each one from headers alone.

---

## 1. The Frozen Snapshot Problem

**The trap:** an HTTP cache stores the response *as a unit* — body **and headers together**. On a HIT it replays that snapshot verbatim (refreshing only `Date`). It never asks origin "have your headers changed?"

**Why it bites:** you fix a header bug at origin (add `Cache-Control`, add `ETag`), verify with curl against origin, declare victory. But every edge node keeps serving the **pre-fix snapshot** — with the old headers — until the object expires, is purged, or the *binary* changes. Your fix exists everywhere except where users are served.

**How we caught it — timestamp forensics:** Fastly's `x-timer` header embeds the Unix timestamp of when Fastly processed the request (`x-timer: S1783003048.780239,...`). When a response arrives via an upstream CDN, compare that embedded timestamp against the response's `Date`:

```
x-timer:  S1782839467...   →  ~45.4 hours ago
date:     (current time)
```

A 45-hour gap = the edge is replaying a 45-hour-old origin response. The `Date` header looks current (caches refresh it), which is exactly why stale snapshots go unnoticed. Any origin that emits a request-time timestamp (`X-Timer`, `X-Request-Start`, a custom header) gives you this free tripwire.

**Corollary for deployment runbooks:** *every header-rule change must ship with a one-time purge of the affected path space.* Otherwise long-lived, rarely-modified objects (PDFs, fonts, archives — the exact objects you cache longest) keep their pre-fix headers indefinitely, and you'll conclude the fix "didn't work."

---

## 2. The Revalidation Trap (304s That Preserve the Bug)

**The trap:** stale snapshot expires → cache revalidates with `If-Modified-Since` → binary hasn't changed since forever → origin answers `304 Not Modified` → cache **keeps the stored object, headers included**, and resets the TTL.

For immutable-per-version assets, this is a perpetual-motion machine:

```
binary never changes → every revalidation is a 304 →
headers never refresh → stale snapshot lives forever
```

TTL expiry will *never* heal it. Only a purge or a binary change breaks the cycle.

**The subtle Apache footgun that causes it:** RFC 9110 allows a 304 to carry updated `Cache-Control`, and compliant caches merge those headers into the stored copy — 304s can *repair* downstream caches. But Apache's plain `Header set` writes to the *onsuccess* table, **which does not apply to 304 responses**. Your carefully crafted Cache-Control goes out on 200s only; every revalidation ships a bare 304 and the stale edge copy survives.

**The fix is one word:**

```apache
# Applies to 304s too → revalidations actively repair edge caches
Header always set Cache-Control "public, max-age=86400, s-maxage=604800"
```

Self-healing test to add to your validation suite: force a revalidation (`curl -H "If-Modified-Since: <stored LM>"` against origin) and assert the 304 carries `Cache-Control`. If it doesn't, your edge can never self-repair.

---

## 3. The Countdown TTL (Expiry-Anchored max-age)

**The symptom:** the same URL returns `max-age=172317`, then later `max-age=10111`, then `max-age=9631`. The browser TTL is **counting down**.

**The cause:** an origin-side cache (Apache serving from a disk cache, `mod_expires` computing against a cached file's mtime, or an app framework anchoring to an expiry date) is emitting `max-age = fixed_expiry − now`. Users who fetch early in the cycle get ~48 h of browser cache; users who fetch near the end get **minutes**. Browser cache efficiency literally degrades as the origin cache entry ages — and your CDN egress bill absorbs the difference, invisibly.

**Detection:** two requests spaced N seconds apart; if `max-age` drops by ≈ N, you've got a countdown. (An explicit `Expires` at a fixed wall-clock time alongside a varying `max-age` is the confession.)

**Fix:** emit a **constant** Cache-Control at serve time (`Header always set`), and `Header always unset Expires` so legacy expiry math can't leak through. Or override downstream TTL at the CDN (see §6).

---

## 4. Cache-Key Fragmentation Economics

One logical asset, many URLs — selector/variant URLs (`asset.pdf`, `asset.pdf.coredownload.pdf`, `asset.pdf.coredownload.inline.pdf`), query-string permutations, tracking parameters. Each URL = a **separate cache key** = a separate copy at *every* edge region = a separate cold miss.

The cost model people miss: fragmentation multiplies **cold misses**, and cold misses are billed twice — midgress (edge→origin) *and* edge delivery. With V variants and R regions, your first-view origin load is `V × R` fetches per asset instead of `R`. For a large document estate this alone can be a double-digit percentage of origin traffic.

**Fixes, in order of leverage:**
1. **Canonicalize at the source** — templates/links emit one variant.
2. **Normalize the cache key at the edge** — strip the variant selector from the key when bodies are byte-identical (re-apply the differing header, e.g. `Content-Disposition`, by URL match at serve time).
3. If neither: at least make all variants carry identical caching headers, so none becomes a per-request revalidation hole (variants routinely take different code paths at origin — verify each one; see §7).

---

## 5. Reading Hits When the Cache Won't Admit Them

Not every tier announces `X-Cache: HIT`. Signals that expose the truth:

| Signal | What it proves |
|---|---|
| `Age > 0` and incrementing across requests | Response is being served from a shared cache — even with no `X-Cache` header at all |
| `ETag` **format fingerprint** | *Which backend* produced the body: Apache emits `"<size-hex>-<mtime-hex>"` (you can verify: first field = Content-Length in hex); Azure Blob emits `"0x8D..."`; S3 emits an MD5-ish hex. Same URL family returning different ETag formats = **different serving paths** (e.g., direct-binary blob delivery vs. web-server servlet) with independent caching behavior |
| Warm-request TTFB collapse with no HIT header | A silent cache (often an origin-side disk cache) is absorbing repeats |
| Embedded request timestamps vs. `Date` (§1) | Snapshot age |
| `Content-Length` ≠ bytes received | An intermediary is re-encoding (gzip); you may be measuring a *different stored object* than an uncompressed client receives |
| `Set-Cookie` present on cold fetch, absent on warm | You're now being served from a CDN cache that strips cookies — a hit-detector in itself |

And the golden probe: **every URL gets two requests, spaced a few seconds apart.** Cold state tells you almost nothing; the *transition* (MISS→HIT, MISS→MISS, HIT-with-revalidation-every-time) is the diagnosis:

| cold → warm | Meaning |
|---|---|
| MISS → HIT | Healthy: object is being stored |
| MISS → MISS | Not stored: headers forbid it, or a rule bypasses cache |
| REFRESH_HIT → REFRESH_HIT | Stored but *always stale*: no TTL on the stored copy → an origin round-trip per request, forever (bandwidth-cheap, request-expensive) |
| MISS → NEGATIVE_HIT | Error response is negative-cached — check that TTL (§8) |

---

## 6. The Header Ownership Model

The architectural fix underneath all of the above: **decide, per header, which layer owns it — and prefer layers that apply headers at serve time over layers that freeze them into stored objects.**

| Header | Owner | Rationale |
|---|---|---|
| `ETag` / `Last-Modified` | Origin app / web server | Validators must reflect the actual binary |
| `s-maxage` / edge TTL | Web server config, or CDN property rule | Controls shared-cache storage |
| Browser `max-age` | **CDN, generated at serve time** | Immune to frozen snapshots; policy changes activate with a config push, no purge |
| Per-tier cache directives | `Surrogate-Control` for the tier that honors it (Fastly-family); note Akamai ignores it | Lets you disable one tier without blinding the others |

Serve-time layers (Apache `mod_headers` responding even for disk-cache hits; CDN "modify outgoing response header" / downstream-cacheability behaviors) can never serve stale *policy* — only stale *bodies*, which purge-on-change handles. Snapshot layers (CDN stored objects, disk-cache header sidecars) will eventually serve stale policy unless a serve-time layer overrides them. Design accordingly.

---

## 7. Variant Parity Audits

When one asset is reachable through multiple URL forms, the variants often traverse **different origin code paths** (blob-store direct delivery vs. servlet rendering vs. rewrite rules). Consequences observed in the wild for a *single* asset:

- different `Last-Modified` timestamps (blob mtime vs. rendition mtime),
- strong ETag on one variant, none on another,
- full Cache-Control on two variants, **none** on the third,
- one variant compressed by the CDN, siblings not.

The header-less variant is the expensive one: with no TTL, the edge revalidates per-request (§5, REFRESH_HIT row) or the browser falls back to heuristic caching (10% of time-since-Last-Modified — unpredictable, and without validators, expiry means a full re-download).

**Practice:** audit *every* variant, not the canonical URL. A regex-based header rule scoped to `\.pdf$` silently misses `\.pdf\.selector\.pdf`. Write the LocationMatch to cover the whole variant grammar, and prove parity empirically.

---

## 8. Cheap Wins Hiding in the Error Path

- **Negative caching**: edges cache 404s (`TCP_NEGATIVE_HIT`). Good — but check two things: the negative TTL isn't so long that a purge-then-slow-republish window pins a 404 at the edge; and the 404 *body* isn't heavyweight. We found an **85 KB HTML error page** behind every malformed asset URL — bots spraying invalid selector permutations paid 85 KB of midgress per pattern per region until negative cache warmed.
- **`Vary` hygiene**: an unnecessary `Vary` (e.g., `User-Agent`) shards one object into thousands of cache keys. `Header unset Vary` on static binaries is one line.
- **Cookies on static assets**: `Set-Cookie` alongside `public, max-age` is a classic foot-gun — some intermediaries refuse to cache the response, others cache and replay the cookie. Strip cookies on static paths at the web server.
- **HEAD ≠ GET**: `curl -I` is not what browsers do, and caches may treat HEAD differently (or not serve it from a cached GET). Diagnose with `curl -s -o /dev/null -D -` (GET, discard body). Also match `Accept-Encoding` to what you're comparing against (`--compressed`), or you're inspecting a different stored object.

---

## 9. A Purge Strategy That Scales

Purging by URL breaks down the moment variants exist (you must enumerate every permutation) and the moment volume grows. The durable pattern:

1. **Tag at origin**: emit a cache-tag header on responses (`Edge-Cache-Tag: docs-pdf, asset-<id>` for Akamai; `Surrogate-Key` for Fastly). One line of web-server config.
2. **Purge by tag**: content replaced → purge `asset-<id>` → *all* URL variants of that asset evicted in one call. Policy rollout → purge the collection tag.
3. **Automate on the publish event**: replication/publish event → queue → purge API call. (Event → serverless function → CDN API is a weekend build if you already have eventing.)
4. **Runbook rule** (§1): header-rule deployments include a one-time collection purge.

With purge-on-change wired, long edge TTLs (7–30 days) become *safe*, and long TTLs are where the offload actually comes from.

---

## 10. The Validation Sequence

After any caching change, run this exact sequence and keep the header dumps as evidence:

```
1. Purge the target path/tag
2. Request #1 (debug headers on)      → expect MISS + fresh, complete headers
3. Request #2 after a few seconds     → expect HIT, headers retained, Age > 0
4. Forced revalidation (If-Modified-Since against origin)
                                      → expect 304 *carrying* Cache-Control
5. Compare max-age across requests    → expect constant, not counting down
6. Repeat 2–5 for EVERY URL variant   → expect parity
7. Diff CDN-served headers vs. live origin headers
                                      → expect equality (no frozen snapshot)
```

Steps 4, 5, 6 and 7 are the ones standard checklists omit — and they correspond exactly to the four failure modes (§2, §3, §7, §1) that survive a naive "is it a HIT?" test.

---

## Appendix: Minimal Probe Commands

```bash
# GET without body, exact headers (never use -I / HEAD for cache diagnosis)
curl -s -o /dev/null -D - "https://cdn.example.com/path/asset.pdf"

# Akamai debug telemetry (requires debug enabled on the property)
curl -s -o /dev/null -D - \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable, akamai-x-get-cache-key, akamai-x-get-true-cache-key" \
  "https://cdn.example.com/path/asset.pdf"

# Match CDN-tier encoding when comparing objects
curl -s -o /dev/null -D - --compressed "https://origin.example.com/path/asset.pdf"

# Countdown-TTL detector
for i in 1 2; do curl -s -o /dev/null -D - "$URL" | grep -i cache-control; sleep 30; done

# 304 header-repair test
LM=$(curl -s -o /dev/null -D - "$ORIGIN_URL" | awk -F': ' 'tolower($1)=="last-modified"{print $2}' | tr -d '\r')
curl -s -o /dev/null -D - -H "If-Modified-Since: $LM" "$ORIGIN_URL"   # 304 must include Cache-Control
```


