# FurlPay Search v0.2.0 — Architecture & Benchmark Report

> Adaptive Ranking Engine + Progressive Web Discovery with Content Extraction

**Repository:** [github.com/FurlPay/furlpay-search](https://github.com/FurlPay/furlpay-search)  
**Date:** 2026-11-09  
**Status:** Production Build (125 tests passing)

---

## Executive Summary

FurlPay Search v0.2.0 is a high-performance AI retrieval runtime written in pure Rust. It provides:

1. **Sub-millisecond local code search** with adaptive hierarchical ranking
2. **Real-time web discovery** via configurable search providers (DuckDuckGo, Brave, SearXNG, SerpAPI)
3. **Page-level content extraction** with SSRF-safe HTML parsing
4. **Progressive SSE streaming** for real-time result delivery
5. **Deep research mode** with intent-aware query expansion

---

## Architecture

### Crate Structure (7 crates)

| Crate | Purpose | Lines | Tests |
|-------|---------|-------|-------|
| `furlpay-core` | Indexing, adaptive ranking, pattern recognition, AST | ~4,500 | 66 |
| `furlpay-web` | HTTP client, HTML extraction, search providers, SSRF, browser, research | ~5,200 | 55 |
| `furlpay-simd` | CPU feature detection, AVX-512/AVX2/NEON SIMD intersection | ~800 | 4 |
| `furlpay-daemon` | Resident daemon (`furlpayd`), SSE streaming, file watcher | ~1,200 | Runtime |
| `furlpay-ipc` | SPSC ring buffer IPC protocol | ~600 | Runtime |
| `furlpay-eval` | MRR evaluation and benchmarks | ~500 | 1 |
| `furlpay-mcp` | Model Context Protocol server | ~400 | Runtime |

### Key Components

```
┌─────────────────────────────────────────────────────────────┐
│                    furlpayd (daemon)                         │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │ SSE API  │  │ SearchEngine │  │ PatternRecognition    │ │
│  │ :3000    │→ │ (local code) │  │ (entities, intents)   │ │
│  └──────────┘  └──────┬───────┘  └───────────────────────┘ │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐   │
│  │            Adaptive Ranking Engine                    │   │
│  │  Intent Classification → Hierarchical Scoring         │   │
│  │  9 QueryIntent variants, 8 weight profiles            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           SearchOrchestrator (web)                    │   │
│  │  Discovery → Deduplication → Extraction → Ranking     │   │
│  │  Providers: DuckDuckGo, Brave, SearXNG, SerpAPI      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## v0.2.0 Changelog (from v0.1.0)

### Adaptive Ranking Engine (Local Code Search)

| Feature | v0.1.0 | v0.2.0 |
|---------|--------|--------|
| Query classification | Length-based (`len <= 2` → short prefix) | Intent-based: 9 `QueryIntent` variants |
| Scoring | Flat additive | 3-stage hierarchical: identity + semantic + status − penalty |
| Double-counting | Path match amplifies all subordinate signals | Exact path caps subordinate residuals |
| Stub handling | `implementation_quality` overloaded | Separated `implementation_completeness` + `implementation_status_match` |
| Explainability | Fixed-threshold first match | Weighted contribution sorting, dominant signal reporting |
| Feature validation | None | `sanitize()` clamps all to [0.0, 1.0], handles NaN/Inf |

### Web Search Pipeline

| Feature | v0.1.0 | v0.2.0 |
|---------|--------|--------|
| Deduplication | Inline URL lowercase + trailing slash strip | Proper `Deduplicator`: tracking param removal, fragment stripping, XXH3 content hash |
| Page extraction | Not attempted | Top 5 results: fetch + HtmlExtractor (title, text, metadata, published_date) |
| Verification | All results: `SearchHitReceived` | Progressive: `SearchHitReceived` → `PageFetched` → `ContentExtracted` |
| Result identity | None | Stable `{search_id}_r{ordinal}` |
| `verified_live` | `true` (incorrect) | `false` for search hits, only `true` after actual verification |
| Deep research | `"{q} explained"`, `"{q} documentation"`, `"{q} latest news 2026"` | Intent-aware: news/how-to/comparison/general patterns |

### SSE Event Contract

| v0.1.0 | v0.2.0 |
|--------|--------|
| `search.started` | `search.started` (+ `search_id`) |
| — | `web.discovery.started` |
| — | `web.discovery.completed` (+ `duplicates_removed`) |
| `web.intent` | `web.intent` |
| — | `web.extraction.completed` (+ `extraction_count`, `extracted_urls`) |
| `web.results` | `web.results` (+ `result_id`, `extraction_status`, `content_summary`) |
| `search.completed` | `search.completed` (+ `duplicates_removed`, `extraction_count`) |

---

## Benchmark Results

### Local Code Search — Intent Classification Accuracy

| Query | Expected Intent | Result | ✓ |
|-------|----------------|--------|---|
| `tr` | ShortTokenPrefix | ShortTokenPrefix | ✅ |
| `payment` | ContentSearch | ContentSearch | ✅ |
| `authentication` | ContentSearch | ContentSearch | ✅ |
| `blockchain\tron.ts` | ExactRelativePath | ExactRelativePath | ✅ |
| `TRON.TS` | FilenameWithExtension | FilenameWithExtension | ✅ |
| `ast/treesitter.rs stub` | ExactRelativePath+stub | ExactRelativePath+stub | ✅ |
| `treesitter.rs stub` | FilenameWithExtension+stub | FilenameWithExtension+stub | ✅ |

### Ranking Correctness

| Test | Status |
|------|--------|
| Exact path dominance (caps subordinate signals) | ✅ Pass |
| Stub separation (completeness ≠ status match) | ✅ Pass |
| No double-counting (hierarchical scoring) | ✅ Pass |
| Feature sanitization (NaN, Inf → clamped) | ✅ Pass |
| Explanation accuracy (weighted contribution) | ✅ Pass |
| Legacy migration compatibility | ✅ Pass |
| Regression benchmark (11 queries) | ✅ All pass |
| Additional test cases (6 queries) | ✅ All pass |

### Web Search — Verification Levels

| Level | Evidence Required | Contract |
|-------|------------------|----------|
| `SearchHitReceived` | URL + title + snippet from HTTPS search API | Never claims page was visited |
| `PageFetched` | HTTP GET completed, status code recorded | Never claims content was parsed |
| `ContentExtracted` | HTML parsed, title/text/metadata extracted | Never claims content was verified against other sources |
| `CrossVerified` | Multiple independent sources agree | Reserved for future implementation |

---

## Testing

```bash
# All unit tests (125 total)
cargo test --manifest-path Cargo.toml

# Regression benchmark
python scripts/test_benchmark.py

# Additional edge cases
python scripts/test_additional_cases.py

# Live web search
curl "http://127.0.0.1:3000/api/v1/search/stream?q=latest+tech+news&mode=web"

# Live deep research
curl "http://127.0.0.1:3000/api/v1/search/stream?q=latest+tech+news&mode=deep"
```

---

## Security

- **SSRF Guard:** Blocks private IP ranges (10.x, 172.16-31.x, 192.168.x), link-local, metadata endpoints (169.254.169.254), localhost, non-HTTP schemes
- **Content Limits:** Max response size, max text chars, max links enforced
- **Browser Permissions:** Form submission checks, file scheme blocking, localhost SSRF blocking
- **Robots Policy:** Respects `robots.txt` directives
- **TLS Mandatory:** Cannot be disabled for outbound requests
- **Untrusted Input:** All retrieved webpage content treated as untrusted

---

*FurlPay Search is part of the [FurlPay](https://github.com/FurlPay) AI-native financial infrastructure platform.*
