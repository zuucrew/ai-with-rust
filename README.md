# AI with Rust

Building the **agentic AI lifecycle end to end in Rust** — six crates that
together cover everything from typed model output to a deployed, observable
tool-using agent.

This is a learning track, not a framework. The goal is to learn Rust properly by
building real systems in a domain I already know, rather than reimplementing
things I already understand in a new language.

---

## Why Rust for this at all

The AI stack splits into two layers, and the honest answer differs for each.

**Orchestration frameworks** — LangChain, LlamaIndex, LangGraph, LiveKit Agents,
PEFT/TRL. Their value is accumulated affordances: integrations, retry semantics,
callback hooks, years of edge cases. Rust has no equivalent and won't soon.
Nothing here tries to replace them.

**Everything those frameworks call** — tokenization, crawling, parsing,
chunking, BM25, vector search, protocol serving, HTTP, deployment. Here Rust is
frequently *already the implementation* and Python is the binding.
[`tokenizers`](https://github.com/huggingface/tokenizers),
[`safetensors`](https://github.com/huggingface/safetensors), `tiktoken`'s core,
and [Qdrant](https://github.com/qdrant/qdrant) are all written in Rust.

**This repo builds the second layer, completely.** Not a port of a Python
course — the production layer a Python prototype graduates into.

---

## The lifecycle

Nine stages. Every one is covered, and the table is honest about how strong the
Rust story actually is at each.

| # | Stage | Rust verdict | Crate |
|---|-------|--------------|-------|
| 1 | Model access & typed output | **Wins** — `serde` + `schemars` is compile-time where Pydantic is runtime | `ashlar` |
| 2 | Evaluation | Viable — metrics are trivial; the concurrent harness is pleasant Rust | `ashlar` |
| 3 | Ingestion — crawl, parse, chunk | **Wins** — crawling and chunking are two of Rust's strongest showings | `quarryworks` |
| 4 | Indexing & hybrid retrieval | **Wins** — `tantivy` is world-class; Qdrant is Rust | `beaconry` |
| 5 | Serving & routing | **Wins** — streaming, backpressure, no GIL | `roundhouse` |
| 6 | Tools & protocol | **Wins hardest** — `rmcp`, one static binary, no venv | `toolworks` |
| 7 | Memory & state | **Wins** — `sqlx` gives compile-time-checked SQL | `factotum` |
| 8 | Reasoning loop | Viable, hardest — patterns port; typed enums beat dict-passing | `factotum` |
| 9 | Observability & deployment | **Wins hardest** — a static binary vs a torch-carrying image | `shipyard` |

---

## The crates

### `ashlar` — typed model output and evaluation
*Ashlar is finely dressed stone, cut to exact dimensions.*

Takes a prompt, a target Rust type, and a list of models. Fans out concurrently,
deserializes every response into the **same** struct, and reports which models
conformed and how they disagreed field by field. Grows into an eval harness: the
same machinery pointed at a dataset with bounded concurrency and aggregate
metrics.

JSON Schema is derived from the Rust type with `schemars` — never hand-written.

**Teaches:** ownership and moves, `Result`/`?`, error enums, `Option`, `match`,
and async kept deliberately shallow.

### `quarryworks` — ingestion
*A quarry is where raw material is extracted and cut into blocks.*

Polite concurrent crawling, HTML and native-text PDF extraction, then chunking:
UTF-8 safe windowing, sentence and paragraph aware splitting, token-aware
chunking via the real `tokenizers` crate, configurable overlap, and byte offsets
mapping each chunk back to its source.

Published to crates.io.

**Teaches:** **lifetimes**, `&str` vs `String`, `Cow`, custom `Iterator`, and
`rayon` vs `tokio`. This is the single most important crate here and the one
that looks most boring.

### `beaconry` — indexing and hybrid retrieval
*A beaconry is a network of beacons — multiple signals fused into one bearing.*

BM25 via `tantivy` plus dense vectors, fused with reciprocal rank fusion and
reranked with a cross-encoder. Local inference enters here — embeddings and
reranker only, 130–400 MB models.

**Teaches:** trait objects, object safety, `Arc`/`RwLock`, Cargo feature flags,
associated types.

### `roundhouse` — serving and routing
*A rail roundhouse routes locomotives onto the right track via a turntable.*

A gateway in front of N providers: streaming passthrough, per-key rate limiting,
retry with backoff and failover, response caching, token accounting, request
logging.

**Teaches:** **async with shared state** — `Stream`, `Pin`, `Arc<Mutex>` vs
`DashMap`, `std` vs `tokio` mutexes, channels, cancellation, backpressure,
tower middleware.

### `toolworks` — tools and protocol
*Plain, obvious, and unambiguous in an MCP context.*

An MCP server exposing `beaconry` and real tools, registered in Claude Code.
One static binary, no runtime dependencies — materially better to distribute
than `pip install` plus a venv. Tool schemas derive from Rust types, so the
types are the single source of truth.

**Teaches:** Cargo workspaces, schema derivation, JSON-RPC over stdio, static
linking and binary size.

### `factotum` — memory and the reasoning loop
*A factotum is a servant employed to do all kinds of work.*

A tool-using agent over `beaconry` with tiered memory — working state, session
history, durable long-term store — multi-source retrieval, and a self-correction
loop with a bounded step count. The capstone.

**Teaches:** `async_trait` and `dyn` dispatch, typed state machines, recursive
async, `sqlx` compile-time-checked SQL, structured concurrency, OTel tracing.

### `shipyard` — observability and deployment

Multi-stage Docker to `distroless`, healthchecks, OTel wired end to end, and
**the honest benchmark**: the equivalent Python stack — FastAPI, LangChain,
torch, sentence-transformers — compared on image size, cold start, resident
memory, and p99 under load. Numbers get published either way, including wherever
Rust loses.

---

## Build order

**Stage order is not build order.** The lifecycle is the map; the build order is
a path through it chosen so the *language* difficulty ramps deliberately — one
new category of pain at a time, never two stacked.

| Order | Crate | New Rust concept | Hours |
|-------|-------|-----------------|-------|
| 1 | `ashlar` | Ownership, moves, `Result`/`?` | 20–30 |
| 2 | `quarryworks` | **Lifetimes**, UTF-8, `Cow` | 25–35 |
| 3 | `beaconry` | Trait objects (sync), `Arc`/`RwLock` | 25–35 |
| 4 | `roundhouse` | **Async + shared state**, `Stream`, `Pin` | 25–40 |
| 5 | `toolworks` | Workspaces, JSON-RPC, static linking | 12–20 |
| 6 | `factotum` | **`async_trait` + `dyn`**, state machines | 35–55 |
| 7 | `shipyard` | Multi-stage builds, OTel | 10–15 |

Roughly 160–235 hours. `ashlar` is generic-over-`dyn` on purpose so
`async_trait` lands in `factotum` rather than on top of everything at once.

## Progress

- [ ] Setup — toolchain, three chapters, one working API call
- [ ] `ashlar`
- [ ] `quarryworks`
- [ ] `beaconry`
- [ ] `roundhouse`
- [ ] `toolworks`
- [ ] `factotum`
- [ ] `shipyard`

Every crate has a **40% checkpoint** that is independently useful and git
tagged. Stopping at one is a finished thing, not an abandoned one.

---

## Stack

`tokio` · `axum` · `tower` · `reqwest` · `serde` · `schemars` · `thiserror` ·
`clap` · `tantivy` · `candle` / `fastembed` · `ort` · `tokenizers` · `rayon` ·
`sqlx` · `rmcp` · `governor` · `moka` · `tracing` + `opentelemetry` ·
`criterion` · `insta`

## Deliberately excluded

Each of these was considered and cut for a stated reason, so it doesn't get
re-litigated later:

- **Training and fine-tuning** — PEFT/TRL/accelerate have no Rust answer. Fine-tune in Python, serve in Rust.
- **Graph orchestration (LangGraph-style)** — good learning project, poor economics. `factotum`'s typed state machine covers the same ground for a tenth the cost.
- **Voice / LiveKit Agents** — the Rust SDK exists; the Agents framework is Python/Node only.
- **Neo4j / GraphRAG** — `neo4rs` is noticeably thinner than the Python driver.
- **Scanned-document OCR** — in 2026 that's a VLM call wrapped in plumbing.
- **`tch-rs`** — libtorch with a C++ build dependency and none of the ecosystem.
- **Hand-rolled tokenizers, HNSW indexes, SSE parsers** — these exist. Reimplementing known things is the failure mode this repo avoids.

---

## Full plan

[**PLAN.md**](PLAN.md) has the execution detail: per-crate dependency lists, the
specific compiler errors to expect and *why* they happen, 40% checkpoint
definitions, done criteria, and the traps that catch Python people specifically —
ordered by how many hours they cost.
