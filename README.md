# AI with Rust

**Building the agentic AI lifecycle end to end in Rust — nine projects, one per stage.**

Not a framework, and not a tutorial series. A learning track for engineers who
already build agentic systems in Python and want to learn Rust properly, by
building real things in a domain they already understand.

The projects are ordered so the *language* difficulty ramps deliberately — one
new category of pain at a time, never two stacked. Ownership, then lifetimes,
then trait objects, then async with shared state, and only at the end the
`async_trait` + `dyn` corner that makes people quit.

---

## Why Rust for this at all

The AI stack splits in two, and the honest answer differs for each half.

**Orchestration frameworks** — LangChain, LlamaIndex, LangGraph, LiveKit Agents,
PEFT/TRL. Their value is accumulated affordances: integrations, retry semantics,
callback hooks, years of edge cases. Rust has no equivalent and won't soon.
**Nothing here tries to replace them.**

**Everything those frameworks call** — tokenization, crawling, parsing,
chunking, BM25, vector search, protocol serving, HTTP, deployment. Here Rust is
frequently *already the implementation* and Python is the binding.
[`tokenizers`](https://github.com/huggingface/tokenizers),
[`safetensors`](https://github.com/huggingface/safetensors), `tiktoken`'s core,
and [Qdrant](https://github.com/qdrant/qdrant) are all written in Rust.

This repo builds the second layer, completely — the production layer a Python
prototype graduates into.

---

## Who this is for

**You need:**

- **Python and agentic AI experience.** You've built with LLM APIs and you know
  what embeddings, chunking, RAG, tool calling, and agent loops are. This repo
  explains the *Rust*. It never explains the ML.
- Comfort with the command line and git.
- At least one LLM provider API key.
- macOS or Linux. Developed on Apple Silicon; nothing is Mac-specific except the
  optional Metal acceleration.

**You do not need:**

- **Any Rust.** That is the entire point. Project 1 assumes you have never
  written a line of it.
- A systems programming background, or C/C++.
- A GPU. Local models used here are 130–400 MB and run fine on CPU or Metal.
- To have finished *The Rust Programming Language*. Three chapters is the
  recommended reading, and that's it — the rest is learned by hitting it.

---

## The nine projects

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

Roughly 156–236 focused hours. Every project has a **40% checkpoint** that is
independently useful and git tagged — stopping at one leaves you with a finished
thing, not an abandoned one.

---

## What you'll actually learn

**1 · structured-output** — Ownership and moves. `Result`/`?` and why it needs a
`From` impl. Error enums with a variant per failure mode instead of catching
`Exception` and inspecting the string. `Option` as a type, not a runtime
surprise. `#[serde(untagged)]` for "the object, or an error envelope" — where
Rust's enums beat Python dicts outright.

**2 · evaluation** — Async fan-out with `JoinSet`. `Semaphore` for bounded
concurrency, because unbounded `join_all` over 10k records gets you throttled.
Newtypes so `Tokens` and `Cents` can't be mixed up.

**3 · ingestion** — *The most important project here, and the one that looks
most boring.* Lifetimes as the entire problem rather than a side effect.
`text[start..end]` panicking on a non-char boundary, because Python let you
pretend strings were arrays of characters. Zero-copy chunks that borrow their
source. `Cow<'a, str>`. `rayon` for CPU-bound parsing against `tokio` for
IO-bound fetching.

**4 · retrieval** — Trait objects, synchronously, so `dyn` lands without async on
top. Object safety and why generic methods break it. `Arc` across threads,
`RwLock` for incremental updates, feature flags gating Metal.

**5 · serving** — *Where Rust genuinely beats Python rather than merely tying.*
`std::sync::Mutex` vs `tokio::sync::Mutex`, and why holding one across `.await`
eventually deadlocks. `Stream`, `Pin`, and passing a response body through
without buffering. `Send + Sync + 'static` propagating virally. Cancellation
that drains in-flight requests.

**6 · tools** — Cargo workspaces wiring 1–4 together. Schema derivation as an API
contract. JSON-RPC over stdio. Static linking, LTO, and binary size.

**7 · memory** — `sqlx`'s `query!` macro verifying SQL against a live schema *at
build time*. Transactions whose lifecycle the type system enforces. Newtype IDs
so mixing `UserId` and `ThreadId` is a compile error, not an incident.

**8 · agent** — `#[async_trait]`, `Box<dyn Tool + Send + Sync>`, and why bare
`-> impl Future` doesn't work behind `dyn`. The loop as a typed enum with a
`step` function, not a `while` loop with flags. Recursive async needing
`Box::pin`.

**9 · deployment** — Multi-stage builds, release profile tuning, OTel end to end.

---

## Getting started

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

```bash
git clone https://github.com/zuucrew/ai-with-rust.git && cd ai-with-rust
```

```bash
cd 01-structured-output && cp .env.example .env
```

Fill in at least one provider key in `.env` — it's gitignored. Then:

```bash
cargo check
```

Use `cargo check` while iterating, not `cargo build`. It's several times faster
and it's all you need until you actually want to run something.

**Before project 1**, read exactly three chapters of
[The Book](https://doc.rust-lang.org/book/) — about three hours:

- **Ch 4** — Ownership. Non-negotiable.
- **Ch 10** — Generics, traits, lifetimes. Skim; it re-lands in project 3.
- **Ch 13** — Iterators and closures. Familiar from Python, except lazy *and*
  zero-cost.

Do not work through the whole Book first, and skip rustlings. This track is
deliberately not exercise-driven.

---

## Layout

```
ai-with-rust/
├── README.md
├── PLAN.md                 per-project detail
└── 01-structured-output/
    ├── Cargo.toml
    ├── .env.example
    ├── NOTES.md            resume state
    └── src/
```

Each project is a standalone crate until project 6, where they become a Cargo
workspace — because wiring them together *is* that project's lesson.

Every project keeps a `NOTES.md` recording where things stand and what was
confusing. Resuming after a few weeks away should cost minutes, not a day.

## Progress

- [x] Setup — toolchain, three chapters, one working API call
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

---

## What this is not

Each of these was considered and cut for a stated reason, so it doesn't get
re-litigated later:

- **Training or fine-tuning.** PEFT/TRL/accelerate have no Rust answer and won't.
  Fine-tune in Python, serve in Rust.
- **A LangGraph port.** Good learning project, poor economics. Project 8's typed
  state machine covers the same ground for a tenth the cost.
- **Voice / LiveKit Agents.** The Rust SDK exists; the Agents framework that
  orchestrates STT→LLM→TTS is Python/Node only.
- **Neo4j / GraphRAG.** `neo4rs` works but is noticeably thinner than the Python
  driver.
- **Scanned-document OCR.** In 2026 that's a VLM call wrapped in plumbing.
- **Hand-rolled tokenizers, HNSW indexes, or SSE parsers.** These exist.
  Reimplementing things you already understand in a new language is exactly the
  failure mode this repo avoids.

---

## Full plan

[**PLAN.md**](PLAN.md) has the execution detail: per-project dependency lists,
the specific compiler errors to expect and *why* they happen, checkpoint and
done criteria, and the traps that catch Python people specifically — ordered by
how many hours they cost.
