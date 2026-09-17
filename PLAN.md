# AI with Rust

Learning Rust properly by building the agentic AI lifecycle end to end, in Rust,
where Rust actually belongs.

---

## Premise

The AI stack splits into two layers and Rust's answer differs for each.

**Orchestration frameworks** — LangChain, LlamaIndex, LangGraph, LiveKit Agents,
PEFT/TRL. Their value is accumulated affordances: integrations, retry semantics,
callback hooks, years of edge cases. Rust has no equivalent and won't soon.

**Everything the frameworks call** — tokenization, crawling, parsing, chunking,
BM25, vector storage, protocol serving, HTTP, deployment. Rust is frequently
*already the implementation* and Python is the binding: `tokenizers`,
`safetensors`, `tiktoken`'s core, and Qdrant are all Rust.

This plan builds the second layer, completely. Not a port of a Python course —
the production layer a Python prototype graduates into.

---

## The lifecycle

Nine stages. Every one gets covered; the table says by what, and how honest the
Rust story is at each.

| # | Stage | Rust verdict | Covered by |
|---|-------|--------------|------------|
| 1 | Model access & typed output | **Wins** — `serde`+`schemars` is compile-time where Pydantic is runtime | `ashlar` |
| 2 | Evaluation | Viable — metrics are trivial; the concurrent harness is pleasant Rust | `ashlar` |
| 3 | Ingestion — crawl, parse, chunk | **Wins** — crawling and chunking are two of Rust's strongest showings | `quarryworks` |
| 4 | Indexing & hybrid retrieval | **Wins** — `tantivy` is world-class; Qdrant is Rust | `beaconry` |
| 5 | Serving & routing | **Wins** — streaming, backpressure, no GIL | `roundhouse` |
| 6 | Tools & protocol | **Wins hardest** — `rmcp`, one static binary, no venv | `toolworks` |
| 7 | Memory & state | Wins — `sqlx` gives compile-time-checked SQL | `factotum` |
| 8 | Reasoning loop | Viable, hardest — patterns port; typed enums beat dict-passing | `factotum` |
| 9 | Observability & deployment | **Wins hardest** — static binary vs a torch-carrying image | `shipyard` |

**Stage order is not build order.** The lifecycle is the map; the build order
below is a path through it chosen so the *language* difficulty ramps
deliberately, one new category of pain at a time.

---

## Build order

| Project | Stages | The Rust concept it forces | Hours |
|---------|--------|---------------------------|-------|
| `ashlar` | 1, 2 | Ownership, moves, `Result`/`?`, error enums, shallow async | 20–30 |
| `quarryworks` | 3 | **Lifetimes**, UTF-8, `Cow`, custom `Iterator` | 25–35 |
| `beaconry` | 4 | Trait objects, `Arc`/`RwLock`, feature flags, local inference | 25–35 |
| `roundhouse` | 5 | **Async + shared state**, `Stream`, `Pin`, cancellation | 25–40 |
| `toolworks` | 6 | Workspaces, schema derivation, JSON-RPC, static linking | 12–20 |
| `factotum` | 7, 8 | **`async_trait` + `dyn`**, typed state machines, recursive async | 35–55 |
| `shipyard` | 9 | Multi-stage builds, OTel, the honest benchmark | 10–15 |

~160–235 hours total. Pace is bursty by design, so:

- **Every project has a 40% checkpoint** that is independently useful and git
  tagged. Stopping at one is a finished thing, not an abandoned one.
- **Each keeps a `NOTES.md`** — where I was, what's next, what confused me.
  Resuming after three weeks should cost minutes, not a day.
- **Crate choices get re-verified at the start of each project.** This ecosystem
  moves; a stale dependency list is a liability.

## Working agreement

- Compiler errors get a **why** before a fix. The borrow checker is the subject,
  not an obstacle to route around.
- Push back on bad Rust ideas explicitly.
- ML/AI concepts are assumed known. Explain the Rust.
- Clone freely early. Optimizing allocations before the program works is the
  single most common way Python people lose days to the borrow checker.

---

## Setup (4 hours, not a project)

`rustup`, `cargo`, `rust-analyzer`. Do **not** work the whole Book and do **not**
do rustlings — exercise-driven learning is explicitly out. Read three chapters:

- **Ch 4** — Ownership. Non-negotiable.
- **Ch 10** — Generics, traits, lifetimes. Skim; it re-lands in `quarryworks`.
- **Ch 13** — Iterators and closures. Familiar from Python, except lazy *and*
  zero-cost.

**Ships:** a binary that reads an API key from env, calls one LLM endpoint with
`reqwest`, prints the response.

Use `cargo check` while iterating, not `cargo build`. Several times faster, and
it's all you need until you actually want to run something.

---

## `ashlar` — model access, typed output, evaluation

**Stages 1–2.** Form: CLI.

### What it does
Takes a prompt, a target Rust type, and a list of models. Fans out concurrently,
deserializes every response into the *same* struct, reports which models
conformed and how they disagreed field by field. Then grows into an eval harness:
the same machinery pointed at a dataset instead of one prompt, with bounded
concurrency and aggregate metrics.

### Crates
`clap`, `tokio`, `reqwest`, `serde`/`serde_json`, `schemars`, `thiserror`,
`anyhow`, `tracing`, `comfy-table`.

### Rust concepts — the point
- **Ownership and moves.** Moving `String`s into spawned tasks. "Value moved
  here" within the first hour.
- **`Result` and `?`.** No exceptions. `?` needs a `From` impl between error
  types — which is why `thiserror` exists for libraries, `anyhow` for binaries.
- **Error enums.** A `ProviderError` with a variant per failure mode, instead of
  catching `Exception` and inspecting the string.
- **`Option`.** LLM JSON has missing fields. No truthiness, no implicit null.
- **Enums + `match`.** `#[serde(untagged)]` for "the object, or an error
  envelope" — where Rust's enums beat Python dicts outright.
- **Async, deliberately shallow.** `#[tokio::main]`, `JoinSet`, `Semaphore` for
  bounded fan-out. The `Provider` trait stays **generic, not `dyn`** — that is
  intentional, so `async_trait` pain lands in `factotum` and not on top of
  everything else at once.

### Checkpoint (40%)
One model, one hardcoded output struct, prints parsed JSON.

### Done
Runs a dataset across N models, reports conformance rate, field-level
disagreement, latency distribution, and cost. Schema derived from the Rust type,
never hand-written.

### Wall you'll hit
Moving a `String` into a `tokio::spawn` closure. The closure outlives the
function, so it must own what it captures — hence `move`, hence a clone or an
`Arc<str>`. The compiler is right and the fix is cheap. Clone it.

---

## `quarryworks` — ingestion: crawl, parse, chunk

**Stage 3.** Form: library, published.

> The most important project here and the one that looks most boring. It is the
> only place where lifetimes are the *entire problem* rather than a side effect.
> Everything after it is easier because of it.

### What it does
Fetches (polite concurrent crawling, robots, rate limits), extracts text from
HTML and PDF, then chunks: UTF-8 safe windowing, sentence and paragraph aware
splitting, token-aware chunking via the real `tokenizers` crate, configurable
overlap, and byte offsets mapping each chunk back to its source.

### Crates
`reqwest`, `scraper`, `tokenizers`, `unicode-segmentation`, `memchr`, `rayon`,
`governor`, `criterion`, `insta`. PDF via `lopdf`/`pdf-extract` — keep it to
native-text PDFs; scanned-document OCR is a rabbit hole, not a lesson.

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
- **Custom `Iterator`.** Chunking lazily and streaming, rather than
  materializing a `Vec`.
- **`rayon` vs `tokio`.** CPU-bound parsing under rayon, IO-bound fetching under
  tokio, and why mixing them naively stalls a runtime.

### Checkpoint (40%)
Fixed-size, char-boundary-safe chunker with overlap, tested. Useful immediately.

### Done
Clean `cargo publish --dry-run`, doctests, criterion bench, snapshot tests over
genuinely nasty input: CJK, emoji with ZWJ sequences, text with no whitespace, a
50 MB file. Faster than `langchain-text-splitters` — measure it.

### Wall you'll hit
The self-referential struct, guaranteed. Then `Cow` in return position.

---

## `beaconry` — indexing and hybrid retrieval

**Stage 4.** Form: library + CLI.

### What it does
Indexes a corpus with BM25 and dense vectors, fuses with reciprocal rank fusion,
reranks with a cross-encoder. Local inference enters here — embeddings and
reranker only, 130–400 MB models, comfortable in 16 GB.

### Crates
`tantivy`, `fastembed` or `candle-core`/`candle-transformers` + `hf-hub`,
`lancedb` or `usearch`, `ort` (ONNX reranker), `rayon`, `bincode`.

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
Indexes your own notes. Fused results with both scores, sub-50 ms over 10k
chunks, incremental reindex on file change.

### Wall you'll hit
Candle's Metal feature flags and model loading paths. Then `Arc<RwLock<T>>` lock
scope — holding a read guard longer than intended and deadlocking a writer.

---

## `roundhouse` — serving and routing

**Stage 5.** Form: HTTP service.

> Where Rust genuinely beats Python rather than merely tying. If only one thing
> here gets finished, make it this.

### What it does
A gateway in front of N providers: streaming passthrough, per-key rate limiting,
retry with backoff and failover, response caching, token accounting, request
logging to SQLite.

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
down gracefully.

### Wall you'll hit
`Pin`. Nothing in Python prepares you — it exists because Rust futures are
movable state machines and self-references inside them break on move. Also
"future cannot be sent between threads safely", pointing forty frames from the
actual cause. Read those bottom-up.

---

## `toolworks` — tools and protocol

**Stage 6.** Form: MCP server, single static binary.

> Highest payoff per hour on this list, and only moderately hard.

### What it does
An MCP server exposing `beaconry` and a few real tools, registered in your own
Claude Code config. One binary, no runtime dependencies — materially better to
distribute than `pip install` plus a venv.

### Crates
`rmcp` (official Rust MCP SDK), `tokio`, `serde`, `schemars`, `tracing`, plus
`beaconry` as a path dependency.

### Rust concepts — the point
- **Cargo workspaces.** Wire `quarryworks` and `beaconry` in as local path dependencies.
- **Schema derivation.** `schemars` on tool input types — the same machinery as
  `ashlar`, now generating MCP tool definitions. Types as the single source of
  truth, which is the whole argument for doing this in Rust.
- **JSON-RPC over stdio.** Framing, protocol state, structured errors.
- **Static linking and binary size.** `--release`, LTO, `strip`, `opt-level`.

### Checkpoint (40%)
One tool, one endpoint, connects and responds. Immediately usable.

### Done
Registered in Claude Code, searching your notes. Under 20 MB stripped, starts in
under 50 ms.

---

## `factotum` — memory and the reasoning loop

**Stages 7–8.** Form: service. The capstone.

### What it does
A tool-using agent over `beaconry`, with tiered memory — working state, session
history, durable long-term store — multi-source retrieval, and a self-correction
loop with a bounded step count.

### Crates
`async-trait`, `tokio`, `axum`, `sqlx` (Postgres), `serde`, `schemars`,
`tracing` + `opentelemetry`, plus `beaconry` and `toolworks`.

### Rust concepts — the point
- **`async fn` in traits with `dyn` dispatch.** `#[async_trait]`,
  `Box<dyn Tool + Send + Sync>`, and why bare `-> impl Future` doesn't work
  behind `dyn`. Rust's ugliest corner, unavoidable here.
- **Dynamic dispatch over a registry.** `HashMap<String, Arc<dyn Tool>>`.
- **Typed state machines.** The agent loop as an enum with a `step` function,
  not a `while` loop with flags. Illegal states become unrepresentable — where
  Rust decisively beats passing dicts around.
- **Recursive async.** Self-correction needs `Box::pin`: an async fn that awaits
  itself has infinitely sized state.
- **`sqlx` compile-time-checked SQL.** Queries verified against a live schema at
  build time. Better than any Python ORM, and there's no Python equivalent.
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

## `shipyard` — observability and deployment

**Stage 9.**

Multi-stage Docker, `distroless` or `FROM scratch`. Healthchecks. OTel export
wired end to end across `roundhouse`, `toolworks`, and `factotum`. Real README and docs.

**The measurement that justifies the whole track:** build the equivalent Python
stack — FastAPI, LangChain, torch, sentence-transformers — and compare honestly.
Image size, cold start, resident memory, p99 under load. Publish the numbers
either way, including wherever Rust loses.

---

## Deliberately excluded

- **Training and fine-tuning.** PEFT/TRL/accelerate/bitsandbytes have no Rust
  answer and won't. `burn` can train; the ergonomics and churn cost weeks and
  teach nothing transferable. Fine-tune in Python, serve in Rust.
- **Graph orchestration (LangGraph-style).** No Rust equivalent. A typed state
  graph on `petgraph` + `JoinSet` is a good learning project and poor economics.
  `factotum`'s typed state machine covers the same ground at a tenth the cost.
- **Voice / LiveKit Agents.** LiveKit ships a Rust SDK; the Agents framework
  orchestrating STT→LLM→TTS is Python/Node only. VAD via `ort` would be a real
  Rust win; the surrounding framework is not there.
- **Neo4j / GraphRAG.** `neo4rs` works but is noticeably thinner than the Python
  driver. Not where the hours should go.
- **Scanned-document OCR.** `ocrs` is real, but in 2026 most real extraction is
  a VLM call and the Rust becomes plumbing. `quarryworks` handles native-text PDFs and
  stops there.
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
2. **Self-referential structs.** Lands in `quarryworks`. Own the data or store indices.
3. **`std::sync::Mutex` held across `.await`.** Lands in `roundhouse`.
4. **`async_trait` and viral `Send` bounds.** Lands in `factotum`.
5. **Byte vs char indexing.** Lands in `quarryworks`, loudly, as a panic.
6. **Trait bound errors as walls of text.** Read bottom-up: the last line is
   usually the actual unsatisfied bound.
7. **Deserializing unpredictable LLM JSON.** Python reaches for a dict; Rust
   makes you commit. `#[serde(untagged)]`, `#[serde(default)]`, and
   `serde_json::Value` as a deliberate escape hatch — not a habit.
8. **Slow builds.** `cargo check` while iterating; `cargo build` only to run.
