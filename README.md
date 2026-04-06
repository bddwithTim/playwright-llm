# playwright-llm

A local, specialized LLM system for Playwright test automation. Combines a locally-running Gemma 4 model with RAG (Retrieval-Augmented Generation), context optimization, and autonomous experimentation to produce high-quality Playwright test code.

## Why

- **Cost reduction** — Run locally instead of paying for cloud LLM API calls
- **Specialized model** — Fine-tuned specifically for Playwright patterns, APIs, and best practices
- **Continuous improvement** — Autoresearch loop optimizes the system overnight, unattended
- **Privacy** — Your test patterns, selectors, and codebase stay on your machine
- **Portable** — Works with any Playwright project, not tied to a specific repo

## Architecture

```
User Query → CLI (pw-llm chat / generate)
    ↓
Context Pipeline (selection → displacement)
    ↓
Hybrid Retriever (Vector + BM25) → ChromaDB
    ↓
Gemma 4 E4B 8B (GGUF via Ollama)
    ↓
Response

Autoresearch (overnight) → optimizes RAG params → better retrieval → better answers
```

### How RAG + fine-tuning complement each other

| | Fine-tuning | RAG |
|---|---|---|
| **What it does** | Teaches the model *how* to think about Playwright | Gives the model *what* to know at query time |
| **Best for** | Code patterns, locator strategies, test structure idioms | Latest API docs, your project's page objects, changelogs |
| **Updates** | Retrain needed (slow) | Update the vector store (fast) |

Phase 1 uses RAG only with the base model. Phase 2 adds fine-tuning on a dedicated Mac Mini.

## Hardware Requirements

| Phase | Machine | Specs | Role |
|-------|---------|-------|------|
| **Phase 1** (now) | Any Mac / Linux | 16GB+ RAM | RAG + base model |
| **Phase 2** (later) | Mac Mini M4 Pro | 64GB unified memory | Fine-tuning + serving |

The system is designed to run on Apple Silicon with 16GB minimum. The base Gemma 4 E4B model at Q4 quantization uses ~5GB, leaving room for RAG, embeddings, and the OS.

## Prerequisites

Before starting, you need:

### 1. Node.js (v20+)

```bash
# Check your version
node --version

# Install via nvm if needed
nvm install 20
```

### 2. Ollama

Ollama serves the LLM locally. Install it:

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh
```

Or download from [ollama.com](https://ollama.com).

After installing, pull the base model:

```bash
# Start Ollama (runs in background)
ollama serve

# In another terminal, pull Gemma 4 E4B (8B params, ~5GB download)
ollama pull gemma4:e4b

# Verify
ollama list
```

> **Note:** If you have more RAM (48GB+), you can also pull the 31B model:
> `ollama pull gemma4:31b`. Change the model in `config/default.yaml` to use it.

### 3. Docker (for ChromaDB)

ChromaDB is the vector database that stores document embeddings for RAG.

```bash
# Install Docker Desktop from https://docker.com
# Or via Homebrew:
brew install --cask docker
```

### 4. Git

Required for cloning Playwright docs and the currents-dev best practices repo during indexing.

```bash
git --version  # Should be installed already on macOS
```

## Installation

```bash
# Clone the repo
git clone <your-repo-url> playwright-llm
cd playwright-llm

# Install dependencies
npm install

# Verify the CLI works
npx tsx src/index.ts --help
```

You should see:

```
Usage: pw-llm [options] [command]

Local LLM specialized for Playwright test automation

Commands:
  doctor [options]        Check system health
  index [options]         Index documents into the vector store
  query [options] <text>  Test RAG retrieval without LLM generation
  chat [options]          Interactive chat with Playwright LLM
  generate [options]      One-shot Playwright code generation
  watch [options]         Watch a directory and auto-reindex on changes
  autoresearch            Autonomous RAG optimization
```

### Alias setup (optional)

Add this to your `~/.zshrc` or `~/.bashrc` for convenience:

```bash
alias pw-llm="npx tsx /path/to/playwright-llm/src/index.ts"
```

## Quick Start

This gets you from zero to chatting with a Playwright-aware LLM in ~10 minutes.

### Step 1: Check system health

```bash
npx tsx src/index.ts doctor
```

This checks:
- Hardware (platform, CPU, memory, Apple Silicon detection)
- Ollama (running? target model available?)
- ChromaDB (reachable?)

Fix anything marked with ✗ before continuing.

### Step 2: Start services

```bash
# Terminal 1: Start Ollama (if not already running)
ollama serve

# Terminal 2: Start ChromaDB
docker compose up chromadb -d
```

Verify ChromaDB is running:
```bash
curl http://localhost:8000/api/v1/heartbeat
# Should return: {"nanosecond heartbeat": ...}
```

### Step 3: Index your knowledge base

This is the key step — it scrapes documentation, processes it into chunks, generates embeddings, and stores them in ChromaDB. You have three sources available:

#### 3a. Playwright official docs (recommended first)

Clones the Playwright GitHub repo (sparse checkout, docs only) and indexes all documentation:

```bash
npx tsx src/index.ts index --source playwright-docs
```

This produces ~6,000 chunks covering the full Playwright API reference, guides, and tutorials.

#### 3b. Currents.dev best practices

Clones the [currents-dev/playwright-best-practices-skill](https://github.com/currents-dev/playwright-best-practices-skill) repo — 61 comprehensive docs covering testing patterns, debugging, CI/CD, framework-specific guides, and more:

```bash
npx tsx src/index.ts index --source currents-dev
```

This produces ~1,200 chunks of curated best practices.

#### 3c. Your own Playwright project

Index any local Playwright repo. The scraper reads `.ts`, `.md`, `.yaml`, and `.json` files, classifies them (test, page-object, component, fixture, utility, config, documentation), and chunks them appropriately:

```bash
npx tsx src/index.ts index --source repo --path /path/to/your/playwright/project
```

For example, indexing the sentinel repo:
```bash
npx tsx src/index.ts index --source repo --path /Users/devrev/Documents/devrev/automation/sentinel
```

This picks up page objects, components, fixtures, tests, rules, and documentation from your project.

> **Tip:** Index all three sources for the best results. The RAG system will retrieve the most relevant chunks regardless of source.

### Step 4: Verify retrieval

Test that the RAG system finds relevant content:

```bash
npx tsx src/index.ts query "How to wait for an element to be visible" --show-scores
```

You should see ranked results from your indexed sources with relevance scores.

### Step 5: Start chatting

```bash
npx tsx src/index.ts chat
```

This opens an interactive chat session. The model:
1. Takes your question
2. Retrieves relevant docs/code from ChromaDB
3. Augments the prompt with that context
4. Generates a response using Gemma 4 via Ollama
5. Streams the response token by token

Example questions to try:
- "Write a test that logs into a website and verifies the dashboard"
- "How do I handle file uploads in Playwright?"
- "What's the best locator strategy for a dynamic table?"
- "Write a page object for a search modal"

### Step 6: One-shot code generation

For generating complete files without an interactive session:

```bash
# Generate to stdout
npx tsx src/index.ts generate --task "Write a Playwright test for a login page with email and password fields"

# Generate to file
npx tsx src/index.ts generate --task "Write a page object for a settings page" --output settings.page.ts
```

## CLI Reference

### `pw-llm doctor`

Checks system health: hardware info, Ollama status, ChromaDB reachability, model availability.

```bash
npx tsx src/index.ts doctor
```

### `pw-llm index`

Scrapes, processes, embeds, and stores documents in ChromaDB.

```bash
# Index Playwright official docs
npx tsx src/index.ts index --source playwright-docs

# Index currents-dev best practices
npx tsx src/index.ts index --source currents-dev

# Index a local repo
npx tsx src/index.ts index --source repo --path /path/to/repo

# Use a custom config
npx tsx src/index.ts index --source playwright-docs --config ./my-config.yaml
```

**What happens during indexing:**

1. **Scrape** — Clones repos (sparse checkout for Playwright docs) or reads local files
2. **Process** — Markdown-aware chunking (respects headings, keeps code blocks intact) or code-aware chunking (respects function/class boundaries, preserves imports)
3. **Embed** — Generates 384-dimensional embeddings using `all-MiniLM-L6-v2` (runs locally via transformers.js, no API calls)
4. **Store** — Saves embeddings + metadata in ChromaDB with cosine similarity index
5. **Debug** — Also saves processed chunks as JSON in `data/processed/` for inspection

### `pw-llm query`

Tests RAG retrieval without invoking the LLM. Useful for debugging and tuning.

```bash
# Basic query
npx tsx src/index.ts query "page.getByRole"

# Show similarity scores
npx tsx src/index.ts query "How to handle authentication" --show-scores

# Adjust number of results
npx tsx src/index.ts query "file upload" --top-k 10
```

### `pw-llm chat`

Interactive streaming chat with RAG augmentation.

```bash
# Default: RAG enabled, uses model from config
npx tsx src/index.ts chat

# Without RAG (direct model queries)
npx tsx src/index.ts chat --no-rag

# Override model (e.g., use 31B if you have the RAM)
npx tsx src/index.ts chat --model gemma4:31b
```

### `pw-llm generate`

One-shot code generation for Playwright tests, page objects, or utilities.

```bash
npx tsx src/index.ts generate --task "Write a test for search functionality"
npx tsx src/index.ts generate --task "Create a page object for the login page" --output login.page.ts
npx tsx src/index.ts generate --task "Write a fixture for authenticated users" --no-rag
```

### `pw-llm watch`

Watches a directory for file changes and auto-reindexes. (Stub — implementation coming.)

```bash
npx tsx src/index.ts watch --path /path/to/your/playwright/project
```

### `pw-llm autoresearch`

Autonomous RAG optimization. Runs experiments varying retrieval parameters and scores output quality.

```bash
# Run optimization loop (default: 100 experiments)
npx tsx src/index.ts autoresearch run

# Limit experiments
npx tsx src/index.ts autoresearch run --max-experiments 20

# View results
npx tsx src/index.ts autoresearch status
```

**How autoresearch works:**

1. Selects RAG parameters to try (Bayesian: 70% exploit best, 30% explore random)
2. Runs 5 eval prompts through the LLM with those parameters
3. Scores each output on a composite metric:
   - **40%** TypeScript syntax validity (imports, async/await, balanced braces)
   - **30%** Playwright API accuracy (uses real APIs, no banned methods)
   - **30%** Convention compliance (getByRole, auto-retrying assertions, AAA pattern)
4. Keeps or discards the configuration based on improvement
5. Logs everything to SQLite (`data/experiments.db`)
6. Repeats

**Search space:**

| Parameter | Values |
|-----------|--------|
| `chunk_size` | 256, 384, 512, 768, 1024 |
| `chunk_overlap` | 25, 50, 100, 150 |
| `top_k` | 3, 5, 7, 10 |
| `bm25_weight` | 0.1, 0.2, 0.3, 0.4, 0.5 |

At ~2 minutes per experiment, you can run ~240 experiments overnight.

## Configuration

All configuration lives in `config/default.yaml`. Override with `--config` flag or environment variables.

### Config file

```yaml
serving:
  backend: ollama
  ollama:
    host: http://localhost:11434
    model: gemma4:e4b            # Change to gemma4:31b for larger model
    temperature: 0.1
    num_ctx: 8192

rag:
  embedding_model: all-MiniLM-L6-v2
  vector_store: chroma
  chroma:
    host: http://localhost:8000
    collection: playwright_docs
  chunk_size: 512                # Tokens per chunk
  chunk_overlap: 50              # Overlap between chunks
  top_k: 5                      # Number of retrieved chunks
  hybrid:
    enabled: true
    bm25_weight: 0.3            # Keyword search weight
    vector_weight: 0.7          # Semantic search weight

context:
  max_tokens: 6144              # Max context tokens sent to model
  operators:
    - selection                 # Filter by relevance + token budget
    - displacement              # Reorder: critical info at start/end

autoresearch:
  level: rag                    # rag (Phase 1) or training (Phase 2)
  max_experiments: 100
```

### Environment variables

Override any config value with `PWLLM_` prefix and `__` as separator:

```bash
PWLLM_SERVING__OLLAMA__MODEL=gemma4:31b npx tsx src/index.ts chat
PWLLM_RAG__TOP_K=10 npx tsx src/index.ts query "locators"
```

## Project Structure

```
playwright-llm/
├── src/
│   ├── index.ts                 # CLI entry point (Commander.js)
│   ├── commands/                # CLI command handlers
│   │   ├── doctor.ts            # System health check
│   │   ├── index-docs.ts        # Scrape + chunk + embed + store
│   │   ├── query.ts             # Test RAG retrieval
│   │   ├── chat.ts              # Interactive chat with RAG
│   │   ├── generate.ts          # One-shot code generation
│   │   ├── watch.ts             # File watcher (stub)
│   │   └── autoresearch.ts      # RAG optimization loop
│   │
│   ├── rag/                     # RAG pipeline
│   │   ├── embeddings.ts        # all-MiniLM-L6-v2 via transformers.js
│   │   ├── store.ts             # ChromaDB vector store
│   │   ├── indexer.ts           # chunk → embed → store pipeline
│   │   ├── retriever.ts         # Hybrid vector + BM25 with RRF fusion
│   │   └── query-engine.ts      # Retrieval → context → LLM generation
│   │
│   ├── context/                 # Context optimization
│   │   ├── selection.ts         # Token budget filtering
│   │   ├── displacement.ts      # Primacy-recency reordering
│   │   └── pipeline.ts          # Operator chain
│   │
│   ├── data/                    # Data preparation
│   │   ├── types.ts             # RawDocument, ProcessedChunk types
│   │   ├── processor.ts         # Routes docs to right chunker
│   │   ├── scrapers/
│   │   │   ├── playwright-docs.ts   # Sparse git clone of playwright repo
│   │   │   ├── currents-dev.ts      # Clone currents-dev best practices
│   │   │   └── local-repo.ts        # Read local Playwright project
│   │   └── processors/
│   │       ├── markdown-chunker.ts  # Heading-aware, code-block-preserving
│   │       └── code-extractor.ts    # Function/class boundary-aware
│   │
│   ├── autoresearch/            # Autonomous optimization
│   │   ├── rag-optimizer.ts     # Experiment loop controller
│   │   ├── search-space.ts      # Bayesian parameter selection
│   │   ├── tracker.ts           # SQLite experiment logger
│   │   └── evaluator.ts         # Composite quality scorer
│   │
│   ├── serving/                 # LLM integration
│   │   ├── ollama-client.ts     # Ollama REST API (generate, chat, stream)
│   │   └── prompt-templates.ts  # System prompts, RAG templates
│   │
│   ├── config/
│   │   └── index.ts             # Pydantic-style config (Zod + YAML + env)
│   │
│   └── utils/
│       ├── hardware.ts          # Platform detection
│       ├── logger.ts            # Structured logging
│       └── tokens.ts            # Token estimation
│
├── config/
│   └── default.yaml             # Default configuration
│
├── tests/
│   ├── unit/                    # 19 tests, no external deps
│   └── integration/             # Requires Ollama + ChromaDB
│
├── data/                        # Runtime data (gitignored)
│   ├── raw/                     # Cloned repos
│   ├── processed/               # Chunked JSON (for debugging)
│   ├── vectordb/                # ChromaDB persistent storage
│   └── experiments.db           # Autoresearch results (SQLite)
│
├── models/                      # Model artifacts (gitignored)
│   └── gguf/                    # Fine-tuned GGUF models (Phase 2)
│
├── docker-compose.yml           # ChromaDB service
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

## Training Data Sources

The system indexes three sources, producing ~8,700 chunks in total:

| Source | What it contains | Chunks |
|--------|-----------------|--------|
| **Playwright docs** | Official API reference, guides, tutorials from playwright.dev | ~6,000 |
| **currents-dev** | 61 best-practice docs: testing patterns, debugging, CI/CD, framework guides | ~1,200 |
| **Your repo** | Your own tests, page objects, components, fixtures, rules, docs | Varies |

All three feed into the same ChromaDB collection. The retriever finds the most relevant chunks regardless of source.

## How the Context Pipeline Works

When you ask a question, the system:

1. **Retrieves** top-K chunks via hybrid search (vector similarity + BM25 keyword matching)
2. **Selects** chunks that fit within the token budget (drops lowest-scored chunks)
3. **Displaces** chunks using primacy-recency ordering:
   - Highest-relevance chunks go at the **beginning** and **end** of the context
   - Lower-relevance chunks go in the **middle**
   - This counters the ["lost in the middle"](https://arxiv.org/abs/2307.03172) effect where LLMs pay less attention to the center of their context window
4. **Augments** the prompt with the optimized context + your question
5. **Generates** a response via the LLM

## Evaluation Metrics

The autoresearch evaluator scores generated code on three dimensions:

### Syntax Validity (40%)
- Has proper imports
- Has `test()` or `describe()` blocks
- Uses `async`/`await`
- Balanced braces and parentheses
- TypeScript annotations present

### Playwright API Accuracy (30%)
- Uses valid Playwright APIs: `getByRole`, `getByTestId`, `locator`, `expect`, etc.
- Does NOT use banned methods:
  - `page.waitForTimeout()` — use auto-retrying assertions instead
  - `page.waitForSelector()` — use `locator().waitFor()` instead
  - `page.$()` / `page.$$()` — use `locator()` instead
  - `page.click()` / `page.fill()` — use locator methods instead
  - `waitUntil: 'networkidle'` — fragile, avoid

### Convention Compliance (30%)
- Uses preferred locators: `getByRole()`, `getByTestId()`
- Uses auto-retrying assertions: `toBeVisible()`, `toHaveText()`, `toHaveURL()`
- Follows Arrange-Act-Assert pattern
- Descriptive test names (`should ...`)
- Element-prefixed locator names (`buttonSubmit`, `textBoxEmail`)

## Phased Roadmap

### Phase 1: RAG + Base Model (current)

Everything in this repo works today on 16GB+ machines:
- [x] CLI with all commands
- [x] Data scrapers (Playwright docs, currents-dev, local repos)
- [x] Markdown-aware + code-aware chunking
- [x] Local embeddings (all-MiniLM-L6-v2 via transformers.js)
- [x] ChromaDB vector store
- [x] Hybrid retriever (vector + BM25)
- [x] Context optimization (selection + displacement)
- [x] Streaming chat with RAG
- [x] One-shot code generation
- [x] RAG autoresearch (Bayesian optimization + SQLite tracking)
- [x] Composite evaluator (syntax + API accuracy + conventions)
- [x] 19 unit tests passing
- [ ] Watch mode (file watcher for auto-reindex)
- [ ] Web UI

### Phase 2: Fine-Tuning (Mac Mini M4 Pro 64GB)

Adds a Python training sidecar to the repo:

```
training/
├── pyproject.toml        # uv, mlx-lm dependencies
├── prepare.py            # JSONL → tokenized dataset
├── train.py              # mlx-lm QLoRA fine-tuning
├── evaluate.py           # Run eval suite
├── export.py             # Export to GGUF for Ollama
└── program.md            # Human guidance for autoresearch
```

- [ ] Synthetic Q&A generation from indexed docs (~10,000 pairs)
- [ ] mlx-lm QLoRA fine-tuning (Apple Silicon native)
- [ ] GGUF export + Ollama registration
- [ ] Training-level autoresearch (~24 experiments/night)
- [ ] Compare: fine-tuned 8B vs base 31B (let eval decide)
- [ ] Web UI (Gradio or Next.js)

## Troubleshooting

### Ollama not running

```
[error] Ollama is not running. Start it with: ollama serve
```

Fix: Open a terminal and run `ollama serve`. It runs in the foreground. You can also set it up as a background service.

### Model not found

```
Target: gemma4:e4b ✗ not found — run: ollama pull gemma4:e4b
```

Fix: `ollama pull gemma4:e4b` (downloads ~5GB).

### ChromaDB not reachable

```
[error] ChromaDB not reachable. Run: docker compose up chromadb -d
```

Fix: Make sure Docker is running, then `docker compose up chromadb -d`. Check with `docker ps`.

### No documents indexed

```
[error] No documents indexed. Run: pw-llm index --source <source>
```

Fix: Run the index command for at least one source (see Step 3 above).

### Embedding model download slow

The first time you run `index` or `query`, the `all-MiniLM-L6-v2` embedding model (~80MB) is downloaded and cached locally by transformers.js. Subsequent runs use the cache.

### Out of memory during indexing

If indexing a very large repo, the embedding model processes in batches of 32. If you still hit memory limits, reduce the batch size in `src/rag/embeddings.ts` or index in smaller batches:

```bash
# Index just the docs first, then the repo
npx tsx src/index.ts index --source playwright-docs
npx tsx src/index.ts index --source repo --path /small/subset/of/repo
```

## Development

### Running tests

```bash
# All tests
npm test

# Watch mode
npm run test:watch
```

### Type checking

```bash
npx tsc --noEmit
```

### Building for production

```bash
npm run build        # Outputs to dist/
node dist/index.js   # Run without tsx
```

## Inspired By

- [karpathy/autoresearch](https://github.com/karpathy/autoresearch) — Autonomous AI-driven LLM research: agents modify training code, run experiments, evaluate results
- [Context Cartography](https://arxiv.org/abs/2603.20578) (OpenViking) — Structured governance of context space in LLM systems (7 operators: reconnaissance, selection, simplification, aggregation, projection, displacement, layering)
- [currents-dev/playwright-best-practices-skill](https://github.com/currents-dev/playwright-best-practices-skill) — Comprehensive Playwright best practices (61 docs, used as training data)
- [mattpocock/skills/grill-me](https://github.com/mattpocock/skills) — Systematic plan interrogation (used to pressure-test this project's design)

## License

MIT
