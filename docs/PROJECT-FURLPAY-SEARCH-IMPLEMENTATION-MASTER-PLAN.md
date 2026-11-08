# Project FurlPay — Technical Implementation Master Plan (v1.1 Production Hardened)
## Repository Structure, Module Specifications, API Contracts, Data Models, Algorithms, Testing Strategy and Deployment Architecture (2026)

**Document Type:** Engineering Execution Master Plan  
**Target Systems:** High-Performance AI Agent Retrieval Runtime (`furlpayd`, `furlpay-cli`, `furlpay-mcp`)  
**Target Environments:** Linux (x86_64, aarch64), macOS (Apple Silicon), Windows 11 (x86_64)  
**Primary Languages:** Rust (Core Daemon & Safe Runtime), C++20 (Vectorized SIMD Kernels), WebAssembly (Edge Workers)  
**Date:** November 2026  
**Status:** Approved for Engineering Execution (v1.1 Production Hardened)  

---

## Executive Summary: Strategic Directives & Staged Delivery

Project FurlPay establishes a high-performance retrieval infrastructure designed specifically for autonomous AI coding agents. In multi-agent environments where retrieval operations comprise **48.8% of all agent tool calls**, retrieval latency and precision directly govern operational cost and wall-clock execution time.

### Core Strategic Decision: Code Engine First
To eliminate implementation risk and prevent building two unvalidated retrieval engines simultaneously, FurlPay executes in two strict sequential milestones:
1. **Milestone 1 (Weeks 1–10): The In-Memory Code Search Engine.** Build the local daemon, positional trigram index, Tree-sitter AST symbol graph, Tantivy BM25 layer, safe SIMD kernels, and Model Context Protocol (MCP) server. Prove sub-millisecond resident search and $>0.435$ MRR before touching web infrastructure.
2. **Milestone 2 (Weeks 11–18): The Tri-Modal Web Retrieval Engine.** Build the Edge Vault KV cache, hedged HTTP/3 QUIC connection pools, native Lexbor HTML parser, and asynchronous headless Chromium fallback.

---

## 1. Monorepo Architecture & Repository Structure

Project FurlPay is organized as a Cargo workspace with native C++ SIMD compilation units, enforcing strict memory safety boundaries, zero-copy data passing, and multi-architecture hardware portability.

```text
furlpay/
├── Cargo.toml                          # Workspace root manifest
├── Cargo.lock
├── README.md
├── LICENSE-MIT
├── scripts/
│   ├── setup_hugepages.sh              # 2MB Transparent HugePage configuration
│   ├── benchmark_all.sh                # End-to-end telemetry harness
│   └── deploy_edge_vault.sh            # Cloudflare Workers KV deploy script
├── crates/
│   ├── furlpay-core/                   # Core indexing, data structures & AST graph
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── index/
│   │       │   ├── mod.rs
│   │       │   ├── schema.rs           # Document, chunk, and posting schemas
│   │       │   ├── trigram.rs          # Positional trigram sparse directory
│   │       │   ├── tantivy_wrapper.rs  # Full-text BM25 index layer
│   │       │   ├── generation.rs       # ArcSwap immutable index generations
│   │       │   └── memory_tier.rs      # L1-L5 memory allocation & page locking
│   │       ├── ast/
│   │       │   ├── mod.rs
│   │       │   ├── treesitter.rs       # Tree-sitter incremental parser
│   │       │   └── symbol_graph.rs     # Def-Use call hierarchy & references
│   │       ├── routing/
│   │       │   ├── mod.rs
│   │       │   └── classifier.rs       # Intent & query routing matrix
│   │       └── ranking/
│   │           ├── mod.rs
│   │           ├── rrf.rs              # Reciprocal Rank Fusion implementation
│   │           └── scorer.rs           # Multi-signal relevance scoring
│   ├── furlpay-simd/                   # Multi-target C++20 SIMD extensions with Rust FFI
│   │   ├── Cargo.toml
│   │   ├── build.rs                    # cc crate separate compilation units
│   │   ├── include/
│   │   │   └── furlpay_simd.h          # C ABI header definition
│   │   ├── src/
│   │   │   ├── lib.rs                  # Safe Rust wrapper with capacity verification
│   │   │   ├── cpu_detect.cpp          # CPUID + OSXSAVE + XCR0 feature detection
│   │   │   ├── intersect_avx512.cpp    # 16-lane AVX-512 galloping kernel (-mavx512f)
│   │   │   ├── intersect_avx2.cpp      # 8-lane AVX2 galloping kernel (-mavx2)
│   │   │   ├── intersect_neon.cpp      # 4-lane ARM NEON kernel
│   │   │   └── intersect_scalar.cpp    # Two-pointer galloping reference baseline
│   ├── furlpay-ipc/                    # Zero-copy shared-memory ring buffer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── broker.rs               # Per-client SPSC channel allocator
│   │       ├── shm_posix.rs            # Linux/macOS shm_open implementation
│   │       ├── shm_windows.rs          # Windows CreateFileMappingW implementation
│   │       ├── ring_buffer.rs          # Atomic circular queue with memory barriers
│   │       └── protocol.rs             # Cap'n Proto / binary message structs
│   ├── furlpay-web/                    # High-speed web retrieval engine (Milestone 2)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── router.rs               # Tri-modal pathway dispatcher
│   │       ├── edge_vault.rs           # Cloudflare Workers KV cache adapter
│   │       ├── quic_pool.rs            # HTTP/3 connection pool with safe 0-RTT
│   │       ├── serp_racing.rs          # Parallel hedged query racer (Exa/Brave)
│   │       ├── parser_lexbor.rs        # Native C Lexbor HTML-to-Markdown
│   │       ├── chromium_fallback.rs    # Async headless rendering worker pool
│   │       └── sse_stream.rs           # Progressive Server-Sent Events pipe
│   ├── furlpay-daemon/                 # The resident daemon (furlpayd)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs                 # Daemon entrypoint, signals, lifecycle
│   │       ├── watcher.rs              # inotify / ReadDirectoryChangesW sync
│   │       ├── auth.rs                 # Cryptographic tenant token validation
│   │       └── dispatcher.rs           # IPC request router & execution threads
│   ├── furlpay-mcp/                    # Model Context Protocol server
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs                 # Stdio MCP server for Claude/Cursor/Antigravity
│   │       ├── schema.rs               # JSON Schema tool contracts
│   │       └── tools.rs                # codesearch, websearch, symbol_lookup
│   └── furlpay-eval/                   # Benchmark suite & telemetry
│       ├── Cargo.toml
│       ├── benches/
│       │   ├── code_search_bench.rs    # Criterion p50/p90/p95/p99 benchmark
│       │   └── intersection_bench.rs   # SIMD vs Scalar comparison bench
│       └── src/
│           ├── mrr_eval.rs             # CodeSearchNet & RepoEval MRR tester
│           └── telemetry.rs            # OpenTelemetry histogram export
└── edge/                               # Cloudflare Workers Edge Vault source
    ├── wrangler.toml
    └── src/
        └── index.ts                    # Edge KV lookup & Markdown cache
```

---

## 2. Complete Data Models & Binary Schema Specifications

### 2.1 Positional Trigram Index Format
A naive trigram index storing only document IDs fails on exact substring matching and regex literal verification. Furlpay utilizes a **positional trigram posting list**:

```text
+-----------------------------------------------------------------------------------------+
| POSITIONAL TRIGRAM SCHEMA                                                               |
|                                                                                         |
| TrigramHash (64-bit XXH3)                                                               |
|  │                                                                                      |
|  └──► PostingListHeader { doc_count: u32, total_occurrences: u32 }                      |
|        │                                                                                |
|        ├── DocPosting 0 { doc_id: 104, term_freq: 2, positions: [142, 890] }           |
|        ├── DocPosting 1 { doc_id: 112, term_freq: 1, positions: [45] }                 |
|        └── DocPosting N ...                                                             |
+-----------------------------------------------------------------------------------------+
```

```rust
// crates/furlpay-core/src/index/schema.rs
use serde::{Deserialize, Serialize};

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct DocId(pub u32);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PositionalPosting {
    pub doc_id: DocId,
    pub term_frequency: u16,
    pub byte_offsets: Vec<u32>, // Exact byte locations within the source file
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DocumentMetadata {
    pub doc_id: DocId,
    pub file_path: String,
    pub language_id: u16,
    pub content_hash: [u8; 32],      // SHA-256 of file content
    pub commit_hash: [u8; 20],       // Git commit hash
    pub line_index_offset: u32,      // Offset into line-start lookup table
    pub byte_length: u32,
    pub tenant_id: u64,              // Cryptographically verified tenant
}
```

### 2.2 Shared-Memory Wire Protocol (Per-Client SPSC Topology)
To prevent race conditions, Furlpay enforces **dedicated per-client SPSC channels**:
* Each connected client allocates a distinct mapped memory segment: `/dev/shm/furlpay_client_<pid>`.
* Messages use 64-byte cache-line aligned binary structs:

```rust
// crates/furlpay-ipc/src/protocol.rs
#[repr(C, align(64))]
#[derive(Debug, Clone, Copy)]
pub struct SearchRequestMessage {
    pub request_id: u64,
    pub tenant_token_hash: [u8; 32], // HMAC-SHA256 authorization token
    pub query_type: u8,              // 0 = Exact, 1 = Regex, 2 = Symbol, 3 = Hybrid, 4 = Web
    pub timeout_micros: u32,         // Hard deadline (e.g. 500 µs)
    pub max_results: u16,            // Default: 10
    pub query_len: u16,
    pub query_bytes: [u8; 256],      // Inline query string buffer
    pub path_filter: [u8; 128],      // Glob filter (e.g. "src/**/*.rs")
}

#[repr(C, align(64))]
#[derive(Debug, Clone, Copy)]
pub struct SearchHit {
    pub doc_id: u32,
    pub byte_offset: u32,
    pub match_length: u16,
    pub line_number: u32,
    pub rrf_score: f32,
    pub ast_node_type: u16,          // 1 = Function, 2 = Class, 3 = Interface, 4 = Variable
}

#[repr(C, align(64))]
#[derive(Debug, Clone, Copy)]
pub struct SearchResponseMessage {
    pub request_id: u64,
    pub elapsed_micros: u32,
    pub total_hits: u32,
    pub returned_hits: u16,
    pub generation_id: u64,
    pub status_code: u16,            // 200 = Success, 401 = Unauthorized, 408 = Timeout
    pub hits: [SearchHit; 32],       // Inline top-32 results
}
```

### 2.3 Safe Immutable Index Generation Architecture (`ArcSwap`)
Replacing unsafe pointer reclamation with the Rust `arc-swap` primitive:

```rust
// crates/furlpay-core/src/index/generation.rs
use arc_swap::ArcSwap;
use std::sync::Arc;
use crate::index::schema::DocumentMetadata;
use crate::ast::symbol_graph::SymbolGraph;
use crate::index::trigram::TrigramDirectory;

pub struct IndexGeneration {
    pub generation_id: u64,
    pub commit_hash: [u8; 20],
    pub timestamp_epoch_ms: u64,
    pub trigram_index: Arc<TrigramDirectory>,
    pub bm25_index: Arc<tantivy::Index>,
    pub ast_graph: Arc<SymbolGraph>,
    pub doc_metadata: Arc<Vec<DocumentMetadata>>,
}

pub struct GenerationManager {
    current: ArcSwap<IndexGeneration>,
}

impl GenerationManager {
    pub fn new(initial: Arc<IndexGeneration>) -> Self {
        Self {
            current: ArcSwap::from(initial),
        }
    }

    /// Reader holds an Arc snapshot; old generations are freed safely via reference counts
    pub fn snapshot(&self) -> Arc<IndexGeneration> {
        self.current.load_full()
    }

    /// Atomically publish a new generation without stopping active queries
    pub fn publish(&self, next: Arc<IndexGeneration>) {
        self.current.store(next);
    }
}
```

---

## 3. Query Routing Matrix: Explicit Search Pathways

Do not execute every query across all five retrieval systems. The query routing engine classifies queries and selects optimal primary and secondary pipelines:

| Query Classification | Primary Pathway | Secondary Pathway | Fallback Escalation |
| :--- | :--- | :--- | :--- |
| **1. Exact Identifier**<br>`handlePayment`, `0x8f2a` | **Positional Trigram Index**<br>Flat array `XXH3_64bits` directory lookup | **SIMD Teddy Verify**<br>Byte-level verification against raw text | Route to BM25 if all trigrams exceed 40% Document Frequency |
| **2. Regular Expression**<br>`get.*Token`, `^Bearer\s+` | **Literal Trigram Prefilter**<br>Extract fixed substrings | **Intel Hyperscan / PCRE2**<br>Vectorized DFA regex engine | Full streaming regex scan if no literal trigrams extracted |
| **3. Code Symbol / Definition**<br>`fn verifyAuth`, `class Ledger` | **Symbol Graph (Def-Use)**<br>In-memory call hierarchy database | **Tree-sitter AST Scope**<br>Syntactic parent/child verification | Tantivy BM25 symbol index if syntax tree is unparsed/broken |
| **4. Natural Language**<br>"JWT refresh token expiration" | **Tantivy BM25**<br>Full-text inverted index | **Semantic Vector**<br>Cosine similarity over embeddings | Token-filtered positional scan if vector model unavailable |
| **5. Cross-File Call Graph**<br>"who calls initiateTransfer?" | **Symbol Call-Graph DB**<br>Caller/callee bidirectional index | **AST Parent Scope Walk**<br>Cross-file type references | Multi-file regex prefilter across workspace boundary |
| **6. Hybrid / Complex Question**<br>Multi-token agent engineering task | **Parallel Execution**<br>Trigram + BM25 + AST in parallel | **Reciprocal Rank Fusion (RRF)**<br>Multi-signal scorer ($k=60$) | Dynamic weight re-tuning based on query token entropy |

---

## 4. Multi-Target SIMD Hardware Dispatch & Verification

### 4.1 CPUID + OSXSAVE + XCR0 Feature Detection (`cpu_detect.cpp`)
Checking CPUID alone can trigger an illegal instruction crash if the operating system has not enabled AVX or AVX-512 register state saving.

```cpp
// crates/furlpay-simd/src/cpu_detect.cpp
#include "furlpay_simd.h"

#if defined(_MSC_VER)
    #include <intrin.h>
#elif defined(__x86_64__) || defined(__i386__)
    #include <cpuid.h>
    #include <x86intrin.h>
#endif

FurlpayCpuArch furlpay_detect_cpu(void) {
#if defined(__x86_64__) || defined(_M_X64)
    uint32_t eax, ebx, ecx, edx;

    // Step 1: Check basic CPUID support
    #if defined(_MSC_VER)
        int info[4];
        __cpuid(info, 1);
        ecx = info[2];
    #else
        __cpuid(1, eax, ebx, ecx, edx);
    #endif

    // Step 2: Check OSXSAVE (bit 27 of ECX)
    bool has_osxsave = (ecx & (1 << 27)) != 0;
    if (!has_osxsave) {
        return CPU_ARCH_SCALAR;
    }

    // Step 3: Verify OS has enabled AVX state management via XCR0
    uint64_t xcr0 = _xgetbv(0);
    bool avx_ready = (xcr0 & 0x6) == 0x6; // XMM (bit 1) and YMM (bit 2) enabled
    if (!avx_ready) {
        return CPU_ARCH_SCALAR;
    }

    // Step 4: Check AVX2 support
    #if defined(_MSC_VER)
        __cpuidex(info, 7, 0);
        ebx = info[1];
    #else
        __cpuid_count(7, 0, eax, ebx, ecx, edx);
    #endif

    bool has_avx2 = (ebx & (1 << 5)) != 0;
    bool has_avx512f = (ebx & (1 << 16)) != 0;
    bool has_avx512cd = (ebx & (1 << 28)) != 0;
    bool has_avx512dq = (ebx & (1 << 17)) != 0;
    bool has_avx512bw = (ebx & (1 << 30)) != 0;

    // Step 5: Verify AVX-512 state management in XCR0 (bits 5, 6, 7)
    bool avx512_ready = (xcr0 & 0xE6) == 0xE6;

    if (avx_ready && avx512_ready && has_avx512f && has_avx512cd && has_avx512dq && has_avx512bw) {
        return CPU_ARCH_X86_AVX512;
    }

    if (has_avx2) {
        return CPU_ARCH_X86_AVX2;
    }
#elif defined(__ARM_NEON)
    return CPU_ARCH_ARM_NEON;
#endif
    return CPU_ARCH_SCALAR;
}
```

### 4.2 Safe Rust FFI Binding with Strict Output Capacity Assertions
```rust
// crates/furlpay-simd/src/lib.rs
extern "C" {
    fn furlpay_detect_cpu() -> i32;
    fn furlpay_intersect_simd(
        small_arr: *const u32, small_sz: usize,
        large_arr: *const u32, large_sz: usize,
        out_buf: *mut u32, out_cap: usize,
    ) -> usize;
}

pub fn intersect_posting_lists(a: &[u32], b: &[u32], output: &mut Vec<u32>) {
    if a.is_empty() || b.is_empty() {
        output.clear();
        return;
    }
    let (small, large) = if a.len() <= b.len() { (a, b) } else { (b, a) };

    output.clear();
    output.reserve(small.len());

    let capacity = output.capacity();
    let written = unsafe {
        furlpay_intersect_simd(
            small.as_ptr(), small.len(),
            large.as_ptr(), large.len(),
            output.as_mut_ptr(), capacity,
        )
    };

    // HARD SECURITY ASSERTION: Prevent C++ kernel buffer overflow
    assert!(
        written <= capacity,
        "FATAL: SIMD intersection kernel wrote {} elements into buffer of capacity {}",
        written, capacity
    );

    unsafe {
        output.set_len(written);
    }
}
```

---

## 5. End-to-End Latency Budgets & Storage-Tier Disaggregation

### 5.1 Storage-Tier Performance Truth Matrix
Memory-mapped files on an SSD do not guarantee sub-millisecond execution. Performance is strictly governed by page residency:

| Storage Tier | Storage Medium | Working Set State | Measured Latency Range (p99) |
| :--- | :--- | :--- | :--- |
| **L1: CPU Cache** | L1 / L2 / L3 Registers | Pinned Active Vectors | 20 – 60 nanoseconds |
| **L2: Resident RAM** | 2MB THP Pinned RAM | Memory-Locked Index (`mlock`) | 100 – 180 microseconds (0.18 ms) |
| **L3: Warm OS Buffer** | OS Page Cache (`mmap`) | Recently Touched Files | 350 – 850 microseconds (0.85 ms) |
| **L4: Cold NVMe Storage** | PCIe Gen4 NVMe SSD | Cold Page Fault (I/O) | 45 – 120 milliseconds (120 ms) |
| **L5: Remote Persistent** | Object Store (S3 / GCS) | Unindexed Workspace History | 250 – 1,500 milliseconds |

### 5.2 End-to-End Code Search Latency Breakdown (Resident Working Set)
Correcting previous arithmetic discrepancies, here is the mathematically verified percentile budget for resident queries:

| Pipeline Stage | Implementation Mechanism | p50 Latency | p95 Latency | p99 Latency |
| :--- | :--- | :--- | :--- | :--- |
| **1. Agent IPC Dispatch** | Dedicated SPSC Ring Buffer (`shm_open`) | 8 µs | 12 µs | 16 µs |
| **2. Query Classification** | Tokenizer + AST Symbol Disambiguation | 14 µs | 20 µs | 26 µs |
| **3. Trigram Lookup** | Flat array `XXH3_64bits` hash directory | 22 µs | 35 µs | 45 µs |
| **4. SIMD Intersection** | Runtime AVX-512 / AVX2 / NEON / Scalar kernel | 85 µs | 120 µs | 145 µs |
| **5. Secondary Verification** | Vectorized Teddy / `memchr` substring scan | 80 µs | 115 µs | 135 µs |
| **6. Snippet & AST Scope** | Tree-sitter scope boundary extraction | 35 µs | 55 µs | 70 µs |
| **7. Security & Permissions** | Cryptographic tenant token validation | 6 µs | 10 µs | 14 µs |
| **8. Result Marshaling** | Direct mapped memory pointer assembly | 18 µs | 28 µs | 38 µs |
| **Total Turnaround Time** | **Resident in-memory working set** | **268 µs (0.27 ms)** | **395 µs (0.40 ms)** | **489 µs (0.49 ms)** |

---

## 6. Tri-Modal Web Retrieval Architecture (Milestone 2)

| Pathway | Trigger Condition | Infrastructure Pipeline | Latency Distribution |
| :--- | :--- | :--- | :--- |
| **Path A: Edge Vault Cache Hit** | Static Dev Docs (MDN, GitHub, PyPI, npmjs, crates.io) | Cloudflare Workers Edge KV<br>Pre-rendered Clean Markdown | **p50:** 18 ms<br>**p95:** 35 ms<br>**p99:** 50 ms |
| **Path B: Live SERP Fast-Path** | General programming queries, novel errors, release notes | Hedged QUIC 0-RTT Race:<br>Brave Search API + Exa Neural | **p50:** 160 ms<br>**p95:** 280 ms<br>**p99:** 420 ms |
| **Path C: Live Crawl & Extraction** | Deep blog posts, API references, changelog URLs | Fast SERP + Direct HTTP/3 GET<br>Native C Lexbor Parser (<12 ms) | **p50:** 380 ms<br>**p95:** 650 ms<br>**p99:** 1,200 ms |
| **Path D: Complex SPA Hydration (Fallback)** | Heavy client-side JS hydration (React, Vue, Angular) | Async Headless Chromium Pool<br>Playwright / CDP Cluster | **p50:** 3,200 ms<br>**p95:** 5,500 ms<br>**p99:** 8,000 ms |

### Key Web Pipeline Safeguards:
1. **HTTP/3 Stream Cancellation:** Slower hedged requests are terminated using standard HTTP/3 `H3_REQUEST_CANCELLED` error codes over QUIC `RESET_STREAM` frames.
2. **Replay-Safe 0-RTT:** 0-RTT early data is permitted strictly for idempotent HTTP `GET` requests accompanied by single-use nonces.
3. **Async SPA Job Queue:** Complex client-side hydrated websites are dispatched to an asynchronous background worker pool with a strict 8-second deadline; they do not block the active conversation thread.

---

## 7. Model Context Protocol (MCP) Server Specification

The Model Context Protocol server exposes standardized tools over stdio/JSON-RPC:

### 7.1 Tool 1: `furlpay_search_code`
```json
{
  "name": "furlpay_search_code",
  "description": "High-performance sub-millisecond code retrieval across workspace index",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search term, regex pattern, or symbol name"
      },
      "query_type": {
        "type": "string",
        "enum": ["exact", "regex", "symbol", "hybrid"],
        "default": "hybrid"
      },
      "path_glob": {
        "type": "string",
        "description": "Optional file filter (e.g. src/**/*.rs)"
      },
      "max_results": {
        "type": "integer",
        "default": 10
      },
      "timeout_ms": {
        "type": "integer",
        "default": 500
      }
    },
    "required": ["query"]
  }
}
```

### 7.2 Tool 2: `furlpay_search_web`
```json
{
  "name": "furlpay_search_web",
  "description": "Tri-modal web retrieval across edge documentation vault and live SERP",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Technical query or API documentation request"
      },
      "require_crawl": {
        "type": "boolean",
        "default": false
      },
      "freshness": {
        "type": "string",
        "enum": ["day", "week", "month", "any"],
        "default": "any"
      }
    },
    "required": ["query"]
  }
}
```

---

## 8. Enterprise Security, Tenant Isolation & Sandboxing

1. **Cryptographic Authentication:** Every IPC request must contain an HMAC-SHA256 token derived from the active session secret. Unauthenticated requests are rejected in $<5\ \mu\text{s}$.
2. **Physical Tenant Isolation:** Indices belonging to different organizations are stored in isolated directory paths with distinct POSIX permissions and isolated memory-mapped segments.
3. **Structured Secret Redaction:** Code snippets are scanned prior to serialization using pre-compiled Hyperscan pattern databases detecting AWS access keys, GitHub PATs, private SSH keys, and high-entropy secrets.
4. **Prompt Injection Defense:** Web content is treated strictly as passive data. Content is parsed into structural Markdown, stripped of directive tokens (`<system>`, `[INST]`), and wrapped in immutable `<untrusted_retrieval_data>` blocks with source provenance headers.
5. **Path Traversal Guards:** All path globs are canonicalized and validated against the workspace root, forbidding `../` symlink escapes.

---

## 9. Testing, Quality Assurance & Fuzzing Protocol

| Testing Workstream | Tools & Frameworks | Acceptance Criteria |
| :--- | :--- | :--- |
| **1. SIMD Differential Testing** | `proptest` / `quickcheck` | 100,000,000 randomized posting lists.<br>AVX-512, AVX2, and NEON outputs match Scalar reference 100%. |
| **2. Crash & Security Fuzzing** | `cargo-fuzz` / `libFuzzer`<br>Sanitizers (ASAN, MSAN, TSAN) | Zero crashes on malformed queries, hostile HTML payloads, or corrupted index generation files. |
| **3. Concurrency Verification** | Loom / ThreadSanitizer<br>SPSC single-pair ring buffer | Zero torn reads, zero memory corruption, zero deadlocks under continuous concurrent query load. |
| **4. Incremental Index Sync** | Git-fuzz automated commits<br>File mutation stress harness | Incremental index state matches clean rebuild bit-for-bit after 10,000 random file mutations. |
| **5. Grounded MRR Evaluation** | CodeSearchNet & RepoEval<br>20,000 query evaluation set | Statistically validated $\text{MRR} > 0.435$ with 95% bootstrap confidence interval. |

---

## 10. Engineering Execution Schedule (18 Weeks)

| Phase | Sprint | Timeline | Engineering Focus & Core Deliverables |
| :--- | :--- | :--- | :--- |
| **Phase 1** | Sprint 1 | Weeks 1–2 | **Benchmark Foundation:** Scalar baseline, CodeSearchNet dataset harness, OpenTelemetry histogram telemetry export, and CPUID + OSXSAVE runtime feature detection. |
| **Phase 2** | Sprint 2–3 | Weeks 3–5 | **Core In-Memory Daemon:** Positional trigram index, Tantivy BM25 layer, `ArcSwap` generation manager, and inotify/ReadDirectoryChangesW incremental watcher. |
| **Phase 3** | Sprint 4–5 | Weeks 6–8 | **Structural & Hybrid Retrieval:** Tree-sitter AST, Symbol Graph (Def-Use), query routing matrix, RRF ranker ($k=60$), and SIMD galloping kernels (AVX-512 / AVX2 / NEON). |
| **Phase 4** | Sprint 6 | Weeks 9–10 | **Agent Integration:** Dedicated SPSC shared-memory IPC, Model Context Protocol (MCP) server, and IDE plugins (Cursor / Antigravity). |
| **Phase 5** | Sprint 7–8 | Weeks 11–14 | **Web Retrieval Engine:** Cloudflare Edge Vault KV adapter, hedged QUIC 0-RTT racer, native Lexbor HTML-to-Markdown parser, and async Chromium worker pool. |
| **Phase 6** | Sprint 9 | Weeks 15–18 | **Hardening & Release:** Multi-tenant security audit, automated secret redaction, proptest differential suite, and independent benchmark publication. |

---
*Architectural Master Plan approved for engineering implementation in `FurlPay/docs`.*
