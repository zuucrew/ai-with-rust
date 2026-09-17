# AI with Rust

Building the **agentic AI lifecycle end to end in Rust** — nine projects, one
per stage, each a working application rather than an exercise.

A learning track, not a framework. The goal is to learn Rust properly by
building real systems in a domain I already know, instead of reimplementing
things I already understand in a new language.

---

## Why Rust for this at all

The AI stack splits in two, and the honest answer differs for each half.

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

## The nine projects

Build order is lifecycle order, which here also ramps the language difficulty
correctly: one new category of pain at a time, never two stacked.

| # | Project | What it is | Rust verdict | Hours |
|---|---------|-----------|--------------|-------|
| 1 | **structured-output** | Rust type → JSON Schema → validated instance, across providers | **Wins** — compile-time where Pydantic is runtime | 12–18 |
| 2 | **evaluation** | Concurrent, resumable eval harness over a dataset | Viable — the harness is pleasant Rust | 10–15 |
| 3 | **ingestion** | Crawl → extract → chunk, with citation offsets | **Wins** — Rust at its best | 25–35 |
| 4 | **retrieval** | BM25 + dense + fusion + rerank, local inference | **Wins** — `tantivy` is world-class | 25–35 |
| 5 | **serving** | Streaming gateway: routing, limits, failover, cost | **Wins** — no GIL, real backpressure | 25–40 |
| 6 | **tools** | MCP server, one static binary, no venv | **Wins hardest** | 12–20 |
| 7 | **memory** | Tiered memory + workspace on compile-time-checked SQL | **Wins** — no Python equivalent | 12–18 |
| 8 | **agent** | Tool-using reasoning loop with self-correction | Viable, hardest | 25–40 |
| 9 | **deployment** | Distroless images, OTel, and the honest benchmark | **Wins hardest** | 10–15 |

~156–236 focused hours. Every project has a **40% checkpoint** that is
independently useful and git tagged — stopping at one is a finished thing, not
an abandoned one.

---

## What each project teaches

The projects exist to force specific Rust concepts, in an order where each new
difficulty lands alone.

**1 · structured-output** — Ownership and moves. `Result`/`?` and why it needs a
`From` impl. Error enums with a variant per failure mode instead of catching
`Exception` and inspecting the string. `Option` as a type, not a runtime
surprise. `#[serde(untagged)]` for "the object, or an error envelope" — where
Rust's enums beat Python dicts outright.

**2 · evaluation** — Async fan-out with `JoinSet`. `Semaphore` for bounded
concurrency, because unbounded `join_all` over 10k records gets you throttled.
Moving data into spawned tasks. Newtypes so `Tokens` and `Cents` can't mix.

**3 · ingestion** — **The most important project here, and the one that looks
most boring.** Lifetimes as the entire problem rather than a side effect.
`text[start..end]` panicking on a non-char boundary, because Python let you
pretend strings were arrays of characters. Zero-copy chunks that borrow their
source. `Cow<'a, str>`. Custom `Iterator` for lazy streaming. `rayon` for
CPU-bound parsing against `tokio` for IO-bound fetching.

**4 · retrieval** — Trait objects, synchronously, so `dyn` lands without async on
top. Object safety and why generic methods break it. `Arc` across rayon threads,
`RwLock` for incremental updates, Cargo feature flags gating Metal.

**5 · serving** — **Where Rust genuinely beats Python rather than merely tying.**
`std::sync::Mutex` vs `tokio::sync::Mutex` and why holding one across `.await`
eventually deadlocks. `Stream`, `Pin`, and passing a response body through
without buffering. `Send + Sync + 'static` propagating virally. Cancellation
that drains in-flight requests.

**6 · tools** — Cargo workspaces wiring 1–4 together. Schema derivation as an API
contract. JSON-RPC over stdio. Static linking, LTO, and binary size.

**7 · memory** — `sqlx`'s `query!` macro verifying SQL against a live schema at
build time. Transactions whose lifecycle the type system enforces. Newtype IDs
so mixing `UserId` and `ThreadId` is a compile error rather than an incident.

**8 · agent** — `#[async_trait]`, `Box<dyn Tool + Send + Sync>`, and why bare
`-> impl Future` doesn't work behind `dyn`. The loop as a typed enum with a
`step` function, not a `while` loop with flags. Recursive async needing
`Box::pin`.

**9 · deployment** — Multi-stage builds, release profile tuning, OTel end to end.

---

## Domain

Deliberately domain-neutral. Every project runs against a real corpus of your
choosing — your own documents, a crawled site, a public dataset — and none of
them hard-code a vertical.

If you later want to specialize, the two strongest options are **SEC/EDGAR
filings** (XBRL gives free extraction ground truth) and **FDA drug labels**
(DailyMed SPL XML does the same). Both slot in at `ingestion` without touching
anything downstream.

## Progress

- [ ] Setup — toolchain, three chapters, one working API call
- [ ] 1 · structured-output
- [ ] 2 · evaluation
- [ ] 3 · ingestion
- [ ] 4 · retrieval
- [ ] 5 · serving
- [ ] 6 · tools
- [ ] 7 · memory
- [ ] 8 · agent
- [ ] 9 · deployment

## Stack

`tokio` · `axum` · `tower` · `reqwest` · `serde` · `schemars` · `thiserror` ·
`clap` · `tantivy` · `candle` / `fastembed` · `ort` · `tokenizers` · `rayon` ·
`sqlx` · `rmcp` · `governor` · `moka` · `tracing` + `opentelemetry` ·
`criterion` · `insta`

## Deliberately excluded

Each considered and cut for a stated reason, so it doesn't get re-litigated:

- **Training and fine-tuning** — PEFT/TRL/accelerate have no Rust answer. Fine-tune in Python, serve in Rust.
- **Graph orchestration (LangGraph-style)** — good learning project, poor economics. Project 8's typed state machine covers the same ground for a tenth the cost.
- **Voice / LiveKit Agents** — the Rust SDK exists; the Agents framework is Python/Node only.
- **Neo4j / GraphRAG** — `neo4rs` is noticeably thinner than the Python driver.
- **Scanned-document OCR** — in 2026 that's a VLM call wrapped in plumbing.
- **`tch-rs`** — libtorch with a C++ build dependency and none of the ecosystem.
- **Hand-rolled tokenizers, HNSW indexes, SSE parsers** — these exist. Reimplementing known things is the failure mode this repo avoids.

---

## Full plan

[**PLAN.md**](PLAN.md) has the execution detail: per-project dependency lists,
the specific compiler errors to expect and *why* they happen, 40% checkpoint
definitions, done criteria, and the traps that catch Python people specifically —
ordered by how many hours they cost.
