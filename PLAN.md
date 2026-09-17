# AI with Rust — Plan

Nine projects, one per stage of the agentic AI lifecycle. Named for what they
do. Each one is a working application, not an exercise.

---

## Premise

The AI stack splits in two, and the honest answer differs for each half.

**Orchestration frameworks** — LangChain, LlamaIndex, LangGraph, LiveKit Agents,
PEFT/TRL. Their value is accumulated affordances: integrations, retry semantics,
callback hooks, years of edge cases. Rust has no equivalent and won't soon.

**Everything those frameworks call** — tokenization, crawling, parsing,
chunking, BM25, vector search, protocol serving, HTTP, deployment. Here Rust is
frequently *already the implementation* and Python is the binding: `tokenizers`,
`safetensors`, `tiktoken`'s core, and Qdrant are all Rust.

This plan builds the second layer, completely.

## Domain

Deliberately domain-neutral. Every project runs against a real corpus of your
choosing — your own documents, a crawled site, a public dataset — and none of
them hard-code a vertical.

If you later want to specialize, the two strongest options are **SEC/EDGAR
filings** (XBRL gives free extraction ground truth) and **FDA drug labels**
(DailyMed SPL XML does the same). Both slot in at `ingestion` without touching
anything downstream. That's an appendix, not a dependency.

## How to use this

- **Every project has a 40% checkpoint** that is independently useful and git
  tagged. Stopping at one is a finished thing, not an abandoned one.
- **Each keeps a `NOTES.md`** — where I was, what's next, what confused me.
  Resuming after three weeks should cost minutes, not a day.
- **Crate choices get re-verified at the start of each project.** This ecosystem
  moves; a stale dependency list is a liability.
- Hours are focused hours. The calendar is irrelevant.

## Working agreement

- Compiler errors get a **why** before a fix. The borrow checker is the subject,
  not an obstacle to route around.
- Push back on bad Rust ideas explicitly.
- ML/AI concepts are assumed known. Explain the Rust.
- Clone freely early. Optimizing allocations before the program works is the
  single most common way Python people lose days to the borrow checker.

---

## The nine projects

Build order is the lifecycle order, which here also happens to ramp the language
difficulty correctly — one new category of pain at a time, never two stacked.

| # | Project | Rust verdict | New Rust concept | Hours |
|---|---------|--------------|------------------|-------|
| 1 | `structured-output` | **Wins** — compile-time where Pydantic is runtime | Ownership, `Result`/`?`, error enums | 12–18 |
| 2 | `evaluation` | Viable — the concurrent harness is pleasant Rust | Async fan-out, `JoinSet`, `Semaphore` | 10–15 |
| 3 | `ingestion` | **Wins** — crawling and chunking are Rust at its best | **Lifetimes**, UTF-8, `Cow`, `Iterator` | 25–35 |
| 4 | `retrieval` | **Wins** — `tantivy` is world-class; Qdrant is Rust | Trait objects, `Arc`/`RwLock`, features | 25–35 |
| 5 | `serving` | **Wins** — streaming, backpressure, no GIL | **Async + shared state**, `Stream`, `Pin` | 25–40 |
| 6 | `tools` | **Wins hardest** — one static binary, no venv | Workspaces, JSON-RPC, static linking | 12–20 |
| 7 | `memory` | **Wins** — compile-time-checked SQL | `sqlx` macros, migrations, transactions | 12–18 |
| 8 | `agent` | Viable, hardest — typed enums beat dict-passing | **`async_trait` + `dyn`**, state machines | 25–40 |
| 9 | `deployment` | **Wins hardest** — static binary vs a torch image | Multi-stage builds, LTO, OTel | 10–15 |

~156–236 focused hours.

---

## Setup (4 hours, not a project)

`rustup`, `cargo`, `rust-analyzer`. Do **not** work the whole Book and do **not**
do rustlings — exercise-driven learning is explicitly out. Read three chapters:

- **Ch 4** — Ownership. Non-negotiable.
- **Ch 10** — Generics, traits, lifetimes. Skim; it re-lands in `ingestion`.
- **Ch 13** — Iterators and closures. Familiar from Python, except lazy *and*
  zero-cost.

**Ships:** a binary that reads an API key from env, calls one LLM endpoint with
`reqwest`, prints the response.

Use `cargo check` while iterating, not `cargo build`. Several times faster, and
it's all you need until you actually want to run something.

---

## 1. `structured-output`

**Form:** library + CLI.

### What it does
Define a Rust type. Get JSON Schema derived from it, a prompt built around that
schema, and a validated instance of the type back. Handles the real failure
modes: missing fields, wrong types, extra keys, prose wrapped around JSON,
truncated output, and the model returning an error envelope instead.

Multi-provider from the start, so the same type works across models.

### Crates
`serde`/`serde_json`, `schemars`, `reqwest`, `tokio`, `thiserror`, `anyhow`,
`clap`, `tracing`.

### Rust concepts — the point
- **Ownership and moves.** The first thing that will confuse you.
- **`Result` and `?`.** No exceptions. `?` needs a `From` impl between error
  types — which is why `thiserror` exists for libraries and `anyhow` for
  binaries.
- **Error enums.** A `ExtractError` with a variant per failure mode, instead of
  catching `Exception` and inspecting the string.
- **`Option`.** Missing fields are a type, not a runtime surprise. No truthiness.
- **Enums + `match`.** `#[serde(untagged)]` for "the object, or an error
  envelope" — where Rust's enums beat Python dicts outright.
- **Derive macros.** What `#[derive(Deserialize, JsonSchema)]` actually
  generates, and why that makes the type the single source of truth.

### Checkpoint (40%)
One provider, one hardcoded struct, schema derived, validated instance returned.

### Done
Any `#[derive(JsonSchema, Deserialize)]` type extracts across ≥3 providers, with
typed errors distinguishing "model refused", "malformed JSON", "schema
violation", and "transport failure". Retry on the recoverable ones only.

### Wall you'll hit
`?` failing to convert between error types. The message names a missing `From`
impl; `thiserror`'s `#[from]` is the fix, but understand what it generates
before you reach for it.

---

## 2. `evaluation`

**Form:** CLI.

### What it does
Runs `structured-output` over a dataset, across models, concurrently, with
bounded parallelism and resumable progress. Reports conformance rate, field-level
accuracy, latency distribution, cost per record, and per-model disagreement.
Writes results to Parquet or SQLite so runs are comparable over time.

Where ground truth exists it scores against it. Where it doesn't, it scores
inter-model agreement and flags divergence for review.

### Crates
`tokio`, `futures`, `indicatif`, `comfy-table`, `serde`, `rusqlite` or `arrow`,
plus `structured-output` as a path dependency.

### Rust concepts — the point
- **Async fan-out.** `JoinSet` over a dataset, collecting results as they land.
- **`Semaphore` for bounded concurrency.** Rate limits are real; unbounded
  `join_all` over 10k records will get you throttled.
- **Moving data into spawned tasks.** `move` closures, and why the task must own
  what it captures. Clone or `Arc`.
- **`Stream` basics.** Processing results as they arrive rather than collecting.
- **Trait stays generic, not `dyn`.** Deliberate — `async_trait` pain belongs in
  project 8, not here on top of everything else.
- **Newtypes** for units that shouldn't mix: `Tokens(u32)`, `Cents(u64)`.

### Checkpoint (40%)
Sequential run over a CSV, one model, printing a conformance percentage.

### Done
10k-record run across 3 models, bounded at N concurrent, resumable after
Ctrl-C, producing a comparison table and a persisted result set.

### Wall you'll hit
Moving a `String` into a `tokio::spawn` closure. The closure outlives the
function, so it must own what it captures — hence `move`, hence a clone or an
`Arc<str>`. The compiler is right and the fix is cheap. Clone it.

---

## 3. `ingestion`

**Form:** library, and the most important project here.

> It looks like the most boring one. It is the only place where lifetimes are
> the *entire problem* rather than a side effect. Everything after it is easier
> because of it.

### What it does
Fetch → extract → chunk. Polite concurrent crawling with robots and rate limits.
Text extraction from HTML and native-text PDF. Then chunking that is actually
correct: UTF-8 safe windowing, sentence and paragraph aware splitting,
token-aware chunking via the real `tokenizers` crate, configurable overlap, and
byte offsets mapping every chunk back to its source for citation.

### Crates
`reqwest`, `scraper`, `tokenizers`, `unicode-segmentation`, `memchr`, `rayon`,
`governor`, `criterion`, `insta`. PDF via `lopdf`/`pdf-extract` — native-text
only; scanned-document OCR is a rabbit hole, not a lesson.

### Rust concepts — the point
- **UTF-8 is not indexable.** `text[start..end]` panics on a non-char boundary.
  Python let you pretend strings were arrays of characters. Rust will not.
  `char_indices()`, and `unicode-segmentation` for grapheme clusters.
- **Lifetimes, properly.** `fn chunk<'a>(&self, text: &'a str) -> Vec<Chunk<'a>>`
  — zero-copy chunks borrowing their source.
- **Self-referential structs are impossible.** You will try to hold both the
  `String` and the `Chunk`s borrowing it in one struct. It cannot be expressed,
  because moving the struct would invalidate the borrows. Own the data, or store
  `Range<usize>` indices. Understanding *why* is half of understanding the
  borrow checker.
- **`Cow<'a, str>`.** "Normalized sometimes, borrowed otherwise." The most
  Python-hostile type in the language, and it finally clicks here.
- **Custom `Iterator`.** Chunking lazily and streaming, not materializing a
  `Vec`.
- **`rayon` vs `tokio`.** CPU-bound parsing under rayon, IO-bound fetching under
  tokio, and why mixing them naively stalls a runtime.

### Checkpoint (40%)
Fixed-size, char-boundary-safe chunker with overlap, tested. Useful immediately.

### Done
Crawls a real site or ingests a real document set. Doctests, criterion bench,
snapshot tests over genuinely nasty input: CJK, emoji with ZWJ sequences, text
with no whitespace, a 50 MB file. Faster than `langchain-text-splitters` —
measure it.

### Wall you'll hit
The self-referential struct, guaranteed. Then `Cow` in return position.

---

## 4. `retrieval`

**Form:** library + CLI.

### What it does
Indexes a corpus with BM25 and dense vectors, fuses with reciprocal rank fusion,
reranks with a cross-encoder, and filters on metadata. Local inference enters
here — embeddings and reranker only, 130–400 MB models, comfortable in 16 GB.

### Crates
`tantivy`, `fastembed` or `candle-core`/`candle-transformers` + `hf-hub`,
`lancedb` or `usearch`, `ort` (ONNX reranker), `rayon`, `bincode`, plus
`ingestion`.

### Rust concepts — the point
- **Trait objects, synchronously.** `Box<dyn Embedder>` vs `impl Embedder`.
  First real `dyn`, with no async on top — deliberate sequencing.
- **Object safety.** "The trait cannot be made into an object" — generic methods
  break `dyn`. Learn the rule here, where it's cheap.
- **`Arc<T>`** for shared immutable state across rayon threads.
- **`RwLock`** for incremental updates. Interior mutability and lock scope.
- **Cargo feature flags.** `--features local-embed` pulling candle with Metal.
  Optional dependencies and `cfg` gating.
- **Associated types**, `impl Trait` in return position.

### Checkpoint (40%)
`tantivy`-only BM25 over a folder of markdown. Genuinely useful alone.

### Done
Indexes a real corpus. Fused results with both scores and rerank position,
sub-50 ms over 10k chunks, incremental reindex on change, metadata filters.

### Wall you'll hit
Candle's Metal feature flags and model loading paths. Then `Arc<RwLock<T>>` lock
scope — holding a read guard longer than intended and deadlocking a writer.

---

## 5. `serving`

**Form:** HTTP service.

> Where Rust genuinely beats Python rather than merely tying. If only one thing
> here gets finished, make it this.

### What it does
A gateway in front of N providers: streaming passthrough, per-key rate limiting,
retry with backoff and failover, response caching, token and cost accounting,
request logging. Also serves `retrieval` as an HTTP API.

### Crates
`axum`, `tower`/`tower-http`, `tokio`, `reqwest` + `eventsource-stream`,
`governor`, `moka`, `sqlx`, `dashmap`, `tokio-util`, `tracing-subscriber`.

### Rust concepts — the point
- **`std::sync::Mutex` vs `tokio::sync::Mutex`.** Holding a std mutex across an
  `.await` parks the whole worker thread. Sometimes compiles, eventually
  deadlocks. The GIL papered over this entire category of thinking.
- **`Arc<Mutex<T>>` vs `Arc<RwLock<T>>` vs `DashMap`**, and when each is wrong.
- **Streams.** `impl Stream`, `StreamExt`, `Pin<Box<dyn Stream>>`. Passing a
  response body through without buffering it.
- **`Send + Sync + 'static`.** Axum handlers demand them; the bounds propagate
  virally through the call graph.
- **Channels.** `mpsc` for a background metrics writer, `oneshot` for
  request-scoped replies.
- **Cancellation.** `CancellationToken`, `select!`, shutdown that drains
  in-flight requests. Drop semantics matter.
- **Backpressure** via `Semaphore`.
- **Tower middleware** — trait composition at its most idiomatic and most
  confusing.

### Checkpoint (40%)
Passthrough proxy for one provider, streaming without buffering. You'd run this.

### Done
Your own tooling points at it. Survives a provider outage via failover, adds no
measurable streaming latency, exposes `/metrics` with real token counts, shuts
down gracefully under load.

### Wall you'll hit
`Pin`. Nothing in Python prepares you — it exists because Rust futures are
movable state machines and self-references inside them break on move. Also
"future cannot be sent between threads safely", pointing forty frames from the
actual cause. Read those bottom-up.

---

## 6. `tools`

**Form:** MCP server, single static binary.

> Highest payoff per hour here, and only moderately hard.

### What it does
An MCP server exposing `retrieval` and real tools, registered in your own Claude
Code config. One binary, no runtime dependencies — materially better to
distribute than `pip install` plus a venv. Tool schemas derive from Rust types
using the same `schemars` machinery as project 1, so types stay the single
source of truth.

### Crates
`rmcp` (official Rust MCP SDK), `tokio`, `serde`, `schemars`, `tracing`, plus
`retrieval`.

### Rust concepts — the point
- **Cargo workspaces.** Wire projects 1–4 in as local path dependencies. Shared
  `Cargo.lock`, per-crate features, `cargo test --workspace`.
- **Schema derivation as API contract.** The type defines the tool.
- **JSON-RPC over stdio.** Framing, protocol state, structured errors.
- **Static linking and binary size.** `--release`, LTO, `codegen-units`,
  `strip`, `opt-level`.

### Checkpoint (40%)
One tool, connects and responds. Immediately usable.

### Done
Registered in Claude Code, searching your corpus. Under 20 MB stripped, starts
in under 50 ms.

---

## 7. `memory`

**Form:** library + service.

### What it does
Tiered memory: ephemeral working state, session history, and a durable
long-term store with semantic recall through `retrieval`. Plus the workspace
layer that makes it real — saved threads, watchlists, per-user context, and
query history.

### Crates
`sqlx` (Postgres), `sqlx-cli` for migrations, `serde`, `time`, `uuid`, plus
`retrieval`.

### Rust concepts — the point
- **`sqlx` compile-time-checked SQL.** The `query!` macro verifies your SQL
  against a live schema at build time, and types the result. There is no Python
  equivalent — this is a genuine capability gap in Rust's favour.
- **Migrations as code.** Schema versioning that fails the build, not runtime.
- **Transactions and ownership.** A `Transaction` borrows the connection; commit
  consumes it. The type system enforces the lifecycle.
- **`Option` at the storage boundary.** Nullable columns map to `Option<T>` and
  the compiler makes you handle it.
- **Newtype IDs.** `UserId(Uuid)` vs `ThreadId(Uuid)` — mixing them becomes a
  compile error rather than a production incident.

### Checkpoint (40%)
Session history persisted and recalled. Useful on its own.

### Done
Three tiers working, semantic recall over long-term store, migrations run clean
from empty, and a query that would have been a runtime error in Python failing
at `cargo check` instead.

---

## 8. `agent`

**Form:** service. The capstone.

### What it does
A tool-using reasoning loop over `tools` and `memory`: multi-source retrieval,
tool dispatch, reflection, and self-correction, with a bounded step count and
full tracing of every step.

### Crates
`async-trait`, `tokio`, `axum`, `serde`, `schemars`, `tracing` +
`opentelemetry`, plus `tools` and `memory`.

### Rust concepts — the point
- **`async fn` in traits with `dyn` dispatch.** `#[async_trait]`,
  `Box<dyn Tool + Send + Sync>`, and why bare `-> impl Future` doesn't work
  behind `dyn`. Rust's ugliest corner, unavoidable here.
- **Dynamic dispatch over a registry.** `HashMap<String, Arc<dyn Tool>>`.
- **Typed state machines.** The loop as an enum with a `step` function, not a
  `while` loop with flags. Illegal states become unrepresentable — where Rust
  decisively beats passing dicts around.
- **Recursive async.** Self-correction needs `Box::pin`: an async fn that awaits
  itself has infinitely sized state.
- **Structured concurrency.** Timeouts and cancellation across tool calls.
- **Tracing to OTel.** Langfuse has no Rust SDK; its OpenTelemetry endpoint is
  the bridge, and `tracing` is excellent on its own merits.

### Checkpoint (40%)
Single-tool agent that loops twice, with session memory. Works.

### Done
Answers over your corpus with citations, recalls across sessions, recovers from
a failed tool call, bounded step count, per-step spans in a trace viewer.

### Wall you'll hit
`async_trait` plus viral `Send` bounds. By now you'll have the vocabulary —
which is exactly why it's last and not first.

---

## 9. `deployment`

Multi-stage Docker to `distroless` or `scratch`. Healthchecks. OTel wired end to
end across `serving`, `tools`, and `agent`. Release profile tuning. Real docs.

**The measurement that justifies the whole track:** build the equivalent Python
stack — FastAPI, LangChain, torch, sentence-transformers — and compare honestly.
Image size, cold start, resident memory, p99 under load. Publish the numbers
either way, including wherever Rust loses.

---

## Deliberately excluded

Each considered and cut for a stated reason, so it doesn't get re-litigated.

- **Training and fine-tuning.** PEFT/TRL/accelerate/bitsandbytes have no Rust
  answer and won't. `burn` can train; the ergonomics and churn cost weeks and
  teach nothing transferable. Fine-tune in Python, serve in Rust.
- **Graph orchestration (LangGraph-style).** No Rust equivalent. A typed state
  graph on `petgraph` + `JoinSet` is a good learning project and poor economics.
  Project 8's typed state machine covers the same ground for a tenth the cost.
- **Voice / LiveKit Agents.** LiveKit ships a Rust SDK; the Agents framework
  orchestrating STT→LLM→TTS is Python/Node only.
- **Neo4j / GraphRAG.** `neo4rs` works but is noticeably thinner than the Python
  driver. Not where the hours should go.
- **Scanned-document OCR.** `ocrs` is real, but in 2026 most real extraction is
  a VLM call and the Rust becomes plumbing. `ingestion` handles native-text PDFs
  and stops.
- **`tch-rs`.** libtorch with a C++ build dependency and none of the ecosystem.
- **Hand-rolled tokenizers, HNSW indexes, SSE parsers.** `tokenizers`, `usearch`
  and `eventsource-stream` exist. Reimplementing known things in a new language
  is the failure mode this plan exists to avoid.
- **Rust GUI** (Tauri/Dioxus/Leptos). A second hard thing bolted onto the first.

---

## Traps specific to Python people

Ordered by how many hours they cost.

1. **Fighting `clone` too early.** Reading "avoid clone" advice, then spending
   three hours satisfying the borrow checker to skip an allocation that never
   mattered. Clone, ship, profile, then fix.
2. **Self-referential structs.** Lands in `ingestion`. Own the data or store
   indices.
3. **`std::sync::Mutex` held across `.await`.** Lands in `serving`.
4. **`async_trait` and viral `Send` bounds.** Lands in `agent`.
5. **Byte vs char indexing.** Lands in `ingestion`, loudly, as a panic.
6. **Trait bound errors as walls of text.** Read bottom-up: the last line is
   usually the actual unsatisfied bound.
7. **Deserializing unpredictable LLM JSON.** Python reaches for a dict; Rust
   makes you commit. `#[serde(untagged)]`, `#[serde(default)]`, and
   `serde_json::Value` as a deliberate escape hatch — not a habit.
8. **Slow builds.** `cargo check` while iterating; `cargo build` only to run.
