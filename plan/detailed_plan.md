# LangGraph Research Agent: Detailed Implementation Plan

Work through the steps in order. Finish each step's **Verify** check before starting the next step. Each step lists the goal, files, implementation notes, and verification.

## Assumptions and defaults

- Python 3.14 (matches `.python-version`). If a dependency has no 3.14 wheel, lower `requires-python` and `.python-version` to 3.13 and record the change in this file.
- LLM: MLflow AI Gateway chat endpoint through `langchain-openai`, configured by `LLM_BASE_URL` (gateway base URL), `LLM_API_KEY`, and `LLM_MODEL` (endpoint name, default `nemotron-3-ultra`; alternative `nemotron-super`). `deepseek-v4.1-flash` is listed by the gateway, but its backing model is unverified. Do not use it until the gateway configuration is confirmed and tested.
- Embeddings: local `sentence-transformers` model set by `EMBEDDING_MODEL` (default `all-MiniLM-L6-v2`). Free and offline. Gateway alternative `open-ai-embedding` (`text-embedding-3-small`, 1536 dimensions) requires re-embedding all chunks. Each embedding model writes to its own Chroma collection (Step 18), so switching models does not overwrite existing vectors.
- Docling does all extraction (web and PDF). OCR is off by default.
- Storage: Chroma persistent client on disk, Neo4j, Redis, MLflow via Docker Compose.
- SQLite (`data/catalog.sqlite`) is a local catalog for document, section, and chunk metadata only. No database stores research findings.

## Target layout

```text
wiki_reasercher/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── providers.py
│   ├── graphs/
│   │   ├── ingestion.py
│   │   └── research.py
│   ├── ingestion/
│   │   ├── validate.py
│   │   ├── docling_extractor.py
│   │   ├── intermediate.py
│   │   ├── cleaner.py
│   │   └── chunker.py
│   ├── retrieval/
│   │   ├── vector.py
│   │   ├── graph.py
│   │   ├── hybrid.py
│   │   ├── query_decomposition.py
│   │   └── compare.py
│   ├── cache/
│   │   └── redis_cache.py
│   ├── graph_extraction.py
│   ├── catalog.py
│   ├── analysis.py
│   ├── reporter.py
│   └── observability.py
├── data/
│   ├── intermediate/
│   ├── cleaned/
│   ├── chroma/
│   ├── catalog.sqlite
│   └── corpus_version.json
├── reports/
├── evaluations/
│   ├── dataset.json
│   └── evaluate.py
├── tests/
│   └── fixtures/
├── plan/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
└── README.md
```

---

## Phase 0: Environment and infrastructure

### Step 1: Project dependencies
- **Goal:** Declare all runtime and dev dependencies.
- **Files:** `pyproject.toml`
- **Notes:** Runtime: `docling`, `langgraph`, `langchain-core`, `langchain-openai`, `sentence-transformers`, `chromadb`, `neo4j`, `redis`, `httpx`, `mlflow`, `pydantic-settings`, `python-dotenv`. Dev group: `pytest`. Use `uv add` for each.
- **Verify:** `uv sync` completes. `uv run python -c "import docling, langgraph, chromadb, neo4j, redis, mlflow"` succeeds.

### Step 2: Folder skeleton
- **Goal:** Create the target directories.
- **Files:** `app/` subpackages, `data/intermediate/`, `data/cleaned/`, `reports/`, `evaluations/`, `tests/fixtures/`.
- **Notes:** Add an empty `__init__.py` to each package under `app/`. Add `.gitkeep` to empty data folders. Add `data/` and `reports/` to `.gitignore`.
- **Verify:** `find app -name __init__.py` lists every package.

### Step 3: Docker Compose
- **Goal:** Provide Neo4j, MLflow, and Redis services.
- **Files:** `docker-compose.yml`
- **Notes:** Use the user's three services as given. Replace `NEO4J_AUTH=neo4j/your_secure_password` with a password of 8+ characters, and use the same value in `.env` as `NEO4J_PASSWORD`. Use the same Redis password in `REDIS_URL`.
- **Verify:** `docker compose config` prints the merged config without errors.

### Step 4: Environment template
- **Goal:** Document all settings.
- **Files:** `.env.example`
- **Notes:** Keys: `NEO4J_URI=bolt://localhost:7687`, `NEO4J_USER=neo4j`, `NEO4J_PASSWORD`, `REDIS_URL=redis://:PASSWORD@localhost:6379/0`, `MLFLOW_TRACKING_URI=http://localhost:5001`, `LLM_BASE_URL` (MLflow AI Gateway URL), `LLM_API_KEY`, `LLM_MODEL=nemotron-3-ultra` (gateway endpoint name), `EMBEDDING_MODEL=all-MiniLM-L6-v2`, `DATA_DIR=./data`, `CATALOG_PATH=./data/catalog.sqlite`, `CHROMA_PATH=./data/chroma`, `REPORTS_DIR=./reports`, `CACHE_TTL_SECONDS=600`, `MAX_RETRIES=2`, `CHUNK_SIZE=1200`, `CHUNK_OVERLAP=150`, `TOP_K=5`, `DOCLING_OCR=false`, `REMOVE_REFERENCES=true`.
- **Verify:** Copy to `.env` locally and confirm the file loads in Step 6. Status: `.env` exists and includes `CATALOG_PATH`. `.env.example` does not exist yet and must be created from the keys above, including `CATALOG_PATH`. Passwords and `LLM_API_KEY` in `.env` are still placeholders.

### Step 5: Service health checks
- **Goal:** Confirm the containers work from the host.
- **Files:** none
- **Notes:** Run `docker compose up -d`. Check MLflow `http://localhost:5001/health`, `redis-cli -a <password> ping`, and Neo4j Browser at `http://localhost:7474` with `RETURN 1`. Confirm an MLflow artifact upload works from the host. Done: the MLflow command uses `--default-artifact-root mlflow-artifacts:/` (proxied uploads). `--serve-artifacts` alone did not fix host uploads. Use new experiments after the change, because old experiments keep their original artifact location.
- **Verify:** All four checks pass. Done.

---

## Phase 1: Configuration and providers

### Step 6: Settings
- **Goal:** Load all configuration from `.env` into one object.
- **Files:** `app/config.py`
- **Notes:** `Settings(BaseSettings)` with the keys from Step 4 and typed defaults. Expose a cached `get_settings()`. Create `DATA_DIR`, `CHROMA_PATH`, and `REPORTS_DIR` on demand.
- **Verify:** `uv run python -c "from app.config import get_settings; print(get_settings().chunk_size)"` prints `1200`.

### Step 7: Providers
- **Goal:** Build the one configured LLM and one embedding model.
- **Files:** `app/providers.py`
- **Notes:** `get_llm()` returns a `ChatOpenAI` with the base URL, key, and model from settings. `get_embedder()` returns an object with `embed_documents(list[str])` and `embed_query(str)` backed by `SentenceTransformer(EMBEDDING_MODEL)`. Cache both.
- **Verify:** Embedding a two-string list returns two vectors of equal length.

---

## Phase 2: Docling ingestion to intermediate Markdown (Day 1)

### Step 8: URL validation and document ID
- **Goal:** Accept only valid sources and derive a stable ID.
- **Files:** `app/ingestion/validate.py`
- **Notes:** Accept `http` and `https` only. `doc_id = sha256(normalized_url)[:16]`, where normalization lowercases the scheme and host, drops fragments, and removes a trailing slash. Return `SourceInfo(url, doc_id, is_wikipedia)`. Wikipedia means the host ends with `wikipedia.org`.
- **Verify:** Unit test: `ftp://x` is rejected, and the same URL with and without a trailing slash gives the same `doc_id`.

### Step 9: Docling extraction
- **Goal:** Convert a web page or PDF to a Docling document.
- **Files:** `app/ingestion/docling_extractor.py`
- **Notes:** Build one `DocumentConverter` per process and cache it. Use `PdfPipelineOptions(do_ocr=DOCLING_OCR)` for PDFs. `convert(source)` accepts a URL or a local path. Return the `DoclingDocument` and the title (`doc.name` or the URL).
- **Verify:** Converting a local HTML fixture returns a document with at least one heading.

### Step 10: Markdown export with page markers
- **Goal:** Produce Markdown that keeps PDF page numbers.
- **Files:** `app/ingestion/docling_extractor.py`
- **Notes:** Iterate `doc.iterate_items()`. For each item, read `prov[0].page_no` when present. Emit `<!-- page: N -->` when the page changes. Headings become `#` lines, using `doc.export_to_markdown` for each item's text if needed. Confirm the API against the installed Docling version.
- **Verify:** Converting the PDF fixture yields `<!-- page: 1 -->` and any later page markers.

### Step 11: Intermediate store
- **Goal:** Save raw Markdown and metadata under `data/intermediate/`.
- **Files:** `app/ingestion/intermediate.py`
- **Notes:** Write `{doc_id}.md` and `{doc_id}.json` with `title`, `source_url`, `source_type` (`web` or `pdf`), `is_wikipedia`, `source_sha256` (hash of the fetched source bytes), `content_sha256` (hash of the extracted Markdown), `converted_at`, `docling_version`. Write atomically (temp file, then rename).
- **Verify:** The files exist with valid JSON after converting a fixture.

### Step 12: Skip unchanged conversions
- **Goal:** Avoid re-running Docling on identical content.
- **Files:** `app/ingestion/intermediate.py`, `app/ingestion/validate.py`
- **Notes:** Every ingest fetches the source bytes and computes `source_sha256`. If `{doc_id}.json` exists, its `source_sha256` matches, and `--force` is not set, skip Docling and return status `skipped`. Otherwise run Docling, write the new `.md` and `.json`, and compare the new `content_sha256` with the previous value. Return a `content_changed` flag, which is true only when `content_sha256` differs. `content_changed` is the trigger for bumping the corpus version (Step 37). A source change that produces identical Markdown does not bump the version.
- **Verify:** A second run without `--force` reports "skipped" and makes no Docling call (check with a mock). A run after the source bytes change reconverts and sets `content_changed` only if the Markdown actually changed.

### Step 13: Phase 2 checkpoint
- **Verify:** Ingest one Wikipedia URL and one PDF URL. Check that both `.md` files have headings. The PDF file has page markers. The second run skips conversion.

---

## Phase 3: Cleaning and chunking (Day 1)

### Step 14: Cleaner
- **Goal:** Remove boilerplate from the Markdown.
- **Files:** `app/ingestion/cleaner.py`
- **Notes:** Remove `[edit]` links, image placeholders, citation markers such as `[1]` and `[citation needed]`, empty headings, and three or more blank lines in a row. When `REMOVE_REFERENCES=true`, drop sections titled References, See also, External links, Notes, and Further reading. Write the result to `data/cleaned/{doc_id}.md`.
- **Verify:** Unit test on a string with each pattern. Each pattern is removed and the surrounding text is kept.

### Step 15: Chunker
- **Goal:** Split cleaned Markdown into chunks with heading and page context.
- **Files:** `app/ingestion/chunker.py`
- **Notes:** First split on Markdown headings, keeping the heading path (for example `"History > Early life"`). Then split long sections on paragraph boundaries to `CHUNK_SIZE` characters with `CHUNK_OVERLAP`. Track the current page from the `<!-- page: N -->` markers. Build the section tree from heading levels: `section_id = sha256(doc_id + section_path)[:16]`. Set `chapter_no` from the level-1 heading number, falling back to the order of level-1 headings. `chunk_id = f"{doc_id}:{index}"`, where `index` is the zero-based position within the document.
- **Verify:** No chunk exceeds `CHUNK_SIZE` plus the overlap. No chunk is empty. Chunk IDs are unique and repeat exactly when the same input is chunked again.

### Step 16: Chunk metadata
- **Goal:** Attach provenance to every chunk.
- **Files:** `app/ingestion/chunker.py`
- **Notes:** Fields: `document_id`, `chunk_id`, `title`, `source_url`, `source_type`, `heading_path`, `page_number` (or `null`), `chunk_index`, `content_sha256`, `section_id`, `chapter_no`. Chroma metadata accepts only scalar values, so store `null` page numbers and missing chapter numbers as `-1`.
- **Verify:** Every chunk dict has all eleven keys.

### Step 17: Phase 3 checkpoint
- **Verify:** Print the chunk count and three sample chunks for each document. Check the sample text and metadata by eye.

---

## Phase 4: Vector store (Day 1)

### Step 18: Chroma upsert
- **Goal:** Store chunk embeddings idempotently.
- **Files:** `app/retrieval/vector.py`
- **Notes:** `chromadb.PersistentClient(path=CHROMA_PATH)`. Collection `research_chunks_<slug>` with cosine distance, where `slug` is `EMBEDDING_MODEL` with every non-alphanumeric character replaced by `_` (for example `research_chunks_all_MiniLM_L6_v2`). Chroma collection names must be 3–63 characters of letters, digits, underscores, or hyphens. Each embedding model gets its own collection, so switching models never mixes vector dimensions. Before upserting, delete all records where `document_id` equals the current document ID. Then `upsert` with `ids=chunk_id`, embeddings from Step 7, and metadata from Step 16.
- **Verify:** Ingesting the same document twice leaves the collection count unchanged.

### Step 19: Chroma search
- **Goal:** Return the top matching chunks.
- **Files:** `app/retrieval/vector.py`
- **Notes:** `search(query, top_k, document_ids=None)`. When `document_ids` is given, use `where={"document_id": {"$in": ids}}`. Return a list of `{chunk_id, score, text, metadata}` with `score = 1 - distance`.
- **Verify:** A query about a topic in the Wikipedia article returns chunks with that document's `document_id` first.

### Step 20: Phase 4 checkpoint
- **Verify:** Run three queries, one per document and one across both. Confirm the document filter works and the counts are stable on re-ingest.

---

## Phase 5: Graph store (Day 2)

### Step 21: Neo4j client and constraints
- **Goal:** Connect to Neo4j and create uniqueness constraints.
- **Files:** `app/retrieval/graph.py`
- **Notes:** Create a driver from `NEO4J_URI`, `NEO4J_USER`, and `NEO4J_PASSWORD`. Run `CREATE CONSTRAINT <name> IF NOT EXISTS FOR (n:Label) REQUIRE n.id IS UNIQUE` for `Document`, `Chunk`, `Topic`, `Entity`, `Claim`, `Methodology`, and `Section`.
- **Verify:** Running the setup twice produces no error. `SHOW CONSTRAINTS` lists seven constraints.

### Step 22: Graph extraction
- **Goal:** Extract entities, topics, claims, methodology, and relations per chunk.
- **Files:** `app/graph_extraction.py`
- **Notes:** A pydantic schema: `entities[{name, type}]`, `topics[str]`, `claims[{text}]`, `methodology[str]`, `relations[{source, target, type}]`. Call the LLM through `with_structured_output`. Batch chunks. Normalize names (lowercase, collapse whitespace) and build IDs as `sha256(type + ":" + normalized_name)[:16]`. Each claim keeps `chunk_id` and `document_id`.
- **Verify:** On a fixture chunk, the output parses into the schema and each claim has a `chunk_id`.

### Step 23: Graph writer
- **Goal:** Write the graph with provenance, without LLM-generated Cypher.
- **Files:** `app/retrieval/graph.py`
- **Notes:** Use fixed, parameterized `MERGE` statements. Create `Document`, `Section`, and `Chunk` nodes, `HAS_SECTION` (Document to Section), `HAS_SUBSECTION` (Section to Section), `IN_SECTION` (Chunk to Section), `CONTAINS`, `DISCUSSES` (Chunk to Topic), `MENTIONS` (Chunk to Entity), `EVIDENCED_BY` (Claim to Chunk), `SUPPORTS`, `CONTRADICTS`, and `RELATED_TO` between entities, and `Methodology` nodes linked by `DISCUSSES`. Claim IDs are `sha256(document_id + chunk_id + normalized_text)[:16]`. Before writing, delete the document's `Chunk` nodes with `DETACH DELETE`, and delete the `Claim` nodes whose `EVIDENCED_BY` edges point to those chunks. Then write fresh data. `Document` and `Section` nodes are keyed by deterministic IDs, so `MERGE` updates them in place. `Entity`, `Topic`, and `Methodology` nodes are shared across documents, so they are never deleted on re-ingest. Orphaned shared nodes are harmless.
- **Verify:** Re-ingesting one document leaves node counts for that document unchanged, including `Claim` counts.

### Step 24: Graph retrieval
- **Goal:** Return graph evidence through fixed query templates.
- **Files:** `app/retrieval/graph.py`
- **Notes:** Templates: `topics_for_documents(doc_ids)`, `claims_for_topic(topic, doc_ids)`, `methodology_for_documents(doc_ids)`. Each returns `chunk_id` values for provenance. Depth and limit are parameters with hard maximums (depth 2, limit 50).
- **Verify:** `claims_for_topic` returns chunk IDs that exist in Chroma.

### Step 25: Phase 5 checkpoint
- **Verify:** Label counts per document in Neo4j Browser. Every `Claim` node has at least one `EVIDENCED_BY` edge.

### Step 25A: SQLite catalog
- **Goal:** Store document, section tree, and chunk-to-section metadata locally.
- **Files:** `app/catalog.py`
- **Notes:** Tables: `documents(doc_id, source, content_hash, title, ingested_at)`, `sections(section_id, doc_id, parent_id, level, number, title, chapter_no, page_start, page_end, order_idx)`, `section_chunks(section_id, chunk_id, order_idx)`. Chunk text stays in Chroma. `replace_document(doc_id, ...)` replaces that document's rows in one transaction.
- **Verify:** Re-ingesting one document leaves row counts unchanged. Other documents are untouched.

### Step 25B: Recursive comparison
- **Goal:** Compare chapters or whole documents section by section.
- **Files:** `app/retrieval/compare.py`
- **Notes:** Select a chapter subtree with `WITH RECURSIVE` over `sections`. `compare(nodeA, nodeB)` matches children by normalized title first, then by embedding similarity above a threshold, and recurses into matched pairs. Matched leaves compare chunk text. Report unmatched sections as "only in A" or "only in B". Supports the same document (chapters 1-3 vs 4-6) and two documents (A vs B).
- **Verify:** A fixture with two chapters where one is renamed matches correctly.

### Step 25C: Phase 5A checkpoint
- **Verify:** Compare chapters 1-3 vs 4-6 of one document. Then compare document A (already indexed) with a newly ingested document B. Neither run re-ingests A.

---

## Phase 6: Hybrid retrieval and LangGraph workflows (Day 2)

### Step 26: Hybrid fusion
- **Goal:** Merge vector and graph results.
- **Files:** `app/retrieval/hybrid.py`
- **Notes:** Reciprocal rank fusion with `k=60`. Deduplicate by `chunk_id`. Keep the source list (`vector`, `graph`, or both) on each result.
- **Verify:** Unit test with two ranked lists that share one ID. The shared ID appears once and ranks first.

### Step 27: Query decomposition
- **Goal:** Split complex questions into sub-queries.
- **Files:** `app/retrieval/query_decomposition.py`
- **Notes:** The LLM returns `{sub_queries: [{query, document_refs}]}`. On a parse failure, return the original question as the only sub-query.
- **Verify:** A two-document comparison yields two sub-queries. A bad LLM response yields one sub-query.

### Step 28: Ingestion graph
- **Goal:** Run ingestion as a LangGraph workflow.
- **Files:** `app/graphs/ingestion.py`
- **Notes:** Nodes: `validate`, `convert`, `write_intermediate`, `clean`, `chunk`, `write_catalog`, `embed_upsert`, `extract_graph`, `write_neo4j`, `bump_version`. Route errors to `END` with `status="failed"` and the message. `bump_version` runs only when content changed.
- **Verify:** Running the graph on the Wikipedia URL ends with `status="ok"`. A bad URL ends with `status="failed"`.

### Step 29: Research graph
- **Goal:** Run the question-answering workflow.
- **Files:** `app/graphs/research.py`
- **Notes:** State: `question`, `request_type`, `document_refs`, `chapter_refs`, `sub_queries`, `evidence`, `analysis`, `verification`, `retries`, `report_path`. Nodes: `intent_classifier` (seven request types from master plan section 4), `planner`, `resolve_docs` (also resolves chapter ranges through the catalog), `decompose`, `vector_retrieve`, `graph_retrieve`, `fuse`, `analyze`, `verify`, `reporter`. Chapter comparison routes to `compare.py` instead of plain vector and graph retrieval. After `verify`, retry through `decompose` while `retries < MAX_RETRIES` and the verifier says evidence is insufficient. Routing: `simple` goes to `vector_retrieve`, `relational` goes to `graph_retrieve` then `vector_retrieve`, `compare` goes to `compare.py`, and `ingest` goes to the ingestion graph (Step 28). An unclear label falls back to `simple`. Request-type mapping: single-document Q&A → `simple`; multi-document synthesis, methodology alignment, and contradiction analysis → `relational`; cross-document comparison and chapter comparison → `compare`; ingest a document → `ingest`. This mapping matches the routing table in master plan section 6. Explicit cues ("compare", "chapter N of", "difference between", "ingest") are checked by rules before the LLM classifier.
- **Verify:** A fake LLM run walks the full path and ends at `reporter`.

### Step 30: Analysis
- **Goal:** Answer from evidence only, with citations.
- **Files:** `app/analysis.py`
- **Notes:** The prompt includes only fusion output and a `PROMPT_VERSION` constant. Each claim must cite a `chunk_id`. Output is a pydantic model: `answer`, `claims[{text, chunk_ids}]`, `comparison_rows`.
- **Verify:** Every returned `chunk_id` appears in the evidence list.

### Step 31: Verification
- **Goal:** Check that the answer is supported.
- **Files:** `app/analysis.py`
- **Notes:** Deterministic check first: every cited `chunk_id` must exist in evidence. Then an LLM judge returns `{supported_claims, unsupported_claims, conflicts, missing_evidence, sufficient}`.
- **Verify:** A claim citing an unknown ID is flagged as unsupported without calling the LLM.

### Step 32: Reporter
- **Goal:** Write the Markdown report.
- **Files:** `app/reporter.py`
- **Notes:** Filename `reports/{UTC timestamp}_{slug}_{uuid8}.md`. Sections follow master plan section 9, and empty sections are omitted. Each citation shows title, source URL, heading path, and page number when set. The reporter returns the file path.
- **Verify:** The report has a Question section and at least one citation that resolves to a chunk ID.

### Step 33: CLI
- **Goal:** Expose ingestion and questions from the command line.
- **Files:** `app/main.py`
- **Notes:** `argparse` with `ingest <url> [--force]` and `ask "<question>"`. `ask` prints the report path and the summary. The existing placeholder `main()` is replaced.
- **Verify:** `uv run python -m app.main --help` lists both commands.

### Step 34: Phase 6 checkpoint
- **Verify:** Ask one cross-document question and one methodology question. Each produces a report with valid citations. Ask a question the documents cannot answer. The report includes a Limitations section.

---

## Phase 7: Redis caching (Day 3)

### Step 35: Cache client
- **Goal:** Wrap Redis with safe fallbacks.
- **Files:** `app/cache/redis_cache.py`
- **Notes:** `get_json(key)` and `set_json(key, value, ttl)`. Any `redis.RedisError` or connection error is logged as a warning, and the call returns `None` or does nothing.
- **Verify:** With Redis stopped, `get_json` returns `None` without raising.

### Step 36: Versioned keys
- **Goal:** Build deterministic keys.
- **Files:** `app/cache/redis_cache.py`
- **Notes:** `rag:{corpus_version}:vec:{sha256(json(query_normalized, document_ids, top_k, EMBEDDING_MODEL))}` and `rag:{corpus_version}:graph:{sha256(json(template, params, depth, limit))}`. Use `sort_keys=True` in the JSON.
- **Verify:** Reordering `document_ids` gives the same key. Changing `top_k` gives a different key.

### Step 37: Corpus version
- **Goal:** Track content changes.
- **Files:** `app/cache/corpus_version.py`
- **Notes:** Store `{ "version": int }` in `data/corpus_version.json`. Increment only when a document's `content_sha256` changes. Write atomically.
- **Verify:** Re-ingesting unchanged content leaves the version unchanged.

### Step 38: Wire the cache
- **Goal:** Check Redis before each database query.
- **Files:** `app/retrieval/vector.py`, `app/retrieval/graph.py`
- **Notes:** Check, then query on miss, then set with `CACHE_TTL_SECONDS`. Record hit or miss for observability.
- **Verify:** The same query twice gives a miss then a hit.

### Step 39: Phase 7 checkpoint
- **Verify:** Stop Redis and run `ask`. It completes. Restart Redis. Re-ingest changed content and confirm the next query misses.

---

## Phase 8: MLflow tracing and evaluation (Day 3)

### Step 40: Observability
- **Goal:** Trace each node without blocking the run.
- **Files:** `app/observability.py`
- **Notes:** Set `MLFLOW_TRACKING_URI` and experiment `research-agent`. A decorator wraps each graph node and records duration, error, and counts. If MLflow is unreachable, log a warning and continue.
- **Verify:** With MLflow stopped, `ask` still completes and logs a warning.

### Step 41: Run logging
- **Goal:** Record each research run.
- **Files:** `app/observability.py`, `app/graphs/research.py`
- **Notes:** Params: `chunk_size`, `chunk_overlap`, `top_k`, `embedding_model`, `llm_model`, `PROMPT_VERSION`, `strategy`, `docling_version`. Metrics: vector and graph result counts, cache hits and misses, retries, duration, and duration per retrieval path (vector, graph, compare). Artifact: the report file.
- **Verify:** One run appears in the MLflow UI with params, metrics, and the report artifact.

### Step 42: Evaluation dataset
- **Goal:** Create the test questions.
- **Files:** `evaluations/dataset.json`
- **Notes:** 10 to 20 entries with `id`, `question`, `type` (single, comparison, methodology, insufficient), `expected_docs`, `expected_keywords`, and `expect_insufficient`.
- **Verify:** The file parses as JSON and every `type` is one of the four values.

### Step 43: Evaluation runner
- **Goal:** Compare retrieval strategies.
- **Files:** `evaluations/evaluate.py`
- **Notes:** Strategies: `vector`, `graph`, `hybrid`, `hybrid_decomposition`. Metrics: retrieval relevance (share of results from expected documents), citation correctness (cited chunks exist and belong to expected documents), keyword completeness, unsupported-claim rate. Log to experiment `research-agent-eval`, one MLflow run per strategy.
- **Verify:** The script runs end to end and creates four MLflow runs.

### Step 44: Phase 8 checkpoint
- **Verify:** Compare the four runs in the MLflow UI and confirm the metrics are present.

---

## Phase 9: Tests and documentation (Day 3)

### Step 45: Fixtures
- **Goal:** Make tests run offline.
- **Files:** `tests/fixtures/sample.html`, `tests/fixtures/sample.pdf`
- **Notes:** A small HTML page with headings and a references section. A two-page PDF with headings. Docling accepts local paths.
- **Verify:** Both files open and Docling converts them.

### Step 46: Unit tests
- **Goal:** Cover the pure logic.
- **Files:** `tests/test_validate.py`, `tests/test_cleaner.py`, `tests/test_chunker.py`, `tests/test_cache_keys.py`, `tests/test_cache_fallback.py`, `tests/test_reporter.py`, `tests/test_ingestion_idempotency.py`, `tests/test_catalog.py`, `tests/test_compare.py`
- **Notes:** Use fake embeddings and a fake Docling output for idempotency. Mock Redis for the fallback test.
- **Verify:** `uv run pytest -m "not integration"` passes with no Docker running.

### Step 47: Integration tests
- **Goal:** Test the real services.
- **Files:** `tests/test_integration.py`
- **Notes:** Mark with `@pytest.mark.integration`. Cover real Docling on fixtures, Chroma, Neo4j, and Redis.
- **Verify:** `uv run pytest -m integration` passes with `docker compose up -d`.

### Step 48: README
- **Goal:** Document setup and usage.
- **Files:** `README.md`
- **Notes:** Prerequisites, `uv sync`, `docker compose up -d`, `cp .env.example .env`, ingest and ask examples, sample questions, and how to run `evaluations/evaluate.py`.
- **Verify:** A fresh clone can follow the README to ingest and ask.

---

## Phase 10: Optional

### Step 49: HyDE
- **Goal:** Improve vector recall for hard questions.
- **Files:** `app/retrieval/hyde.py`
- **Notes:** Generate a hypothetical answer for search only. Never cite it, and never store it.
- **Verify:** Evaluation shows the same or better retrieval relevance on the dataset.

### Step 50: Reranking
- **Goal:** Reorder fused results.
- **Files:** `app/retrieval/hybrid.py`
- **Notes:** Add a reranker step after fusion, only if evaluation shows a gain.
- **Verify:** Evaluation shows an improvement over the fusion-only run.

---

## Definition of done

1. A Wikipedia URL and a PDF URL can be ingested, and their Markdown appears in `data/intermediate/`.
2. Chunks and embeddings are stored in Chroma.
3. Entities, topics, claims, and relations are stored in Neo4j with provenance.
4. Single-document and cross-document questions work.
5. Methodology alignment uses evidence from the relevant documents.
6. Redis caches vector and graph retrieval results and falls back when unavailable.
7. Each research run produces a Markdown report with citations.
8. MLflow records traces and evaluation runs.
9. The project runs locally with the README steps.
10. SQLite is used only as the local catalog (Steps 25A–25B). No other relational database or observability service is required.
