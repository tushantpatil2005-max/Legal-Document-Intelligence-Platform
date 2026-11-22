# Legal Document Intelligence Platform

A retrieval-augmented legal document analysis system designed for contracts, corporate policies, data processing addenda, and multi-document legal collections. The implementation lives in a single self-contained module (`legal_intelligence.py`) with persistent SQLite indexing, structure-aware chunking, metadata filtering, hybrid semantic + lexical retrieval, second-stage reranking, clause discovery, and cited document summarization.

## Core Capabilities

- **Large Document Collection Indexing**: Ingests individual files or entire directories (`.txt`, `.md`, `.json`, and `.pdf` via `pypdf`) into a persistent SQLite index (`documents` and `chunks` tables with indexed metadata columns).
- **Clause-Aware Legal Chunking**: Preserves page boundaries (`\f`, `--- Page N ---`, `[Page N]`) and legal section/article headers (`Article I`, `Section 4.2`, `6. Limitation of Liability`). Windows long sections with configurable word overlap so obligations spanning adjacent sentences remain intact.
- **Metadata Filtering**: Pre-filters retrieval candidates by `doc_id`, `doc_type`, `jurisdiction`, `parties`, `clause_category`, `confidentiality`, and `effective_year` range (`year_from` / `year_to`).
- **Hybrid Semantic & Lexical Search**: Combines dense vector cosine similarity with Okapi BM25 term scoring through Reciprocal Rank Fusion (RRF). Works out of the box with a zero-dependency subword/concept projection fallback, or seamlessly upgrades to remote embeddings (`OPENAI_API_KEY`) or local `sentence-transformers`.
- **Two-Stage Reranking**: Re-scores the top-K hybrid shortlist using a cross-encoder (`USE_CROSS_ENCODER=1`), LLM pointwise reranker (when `OPENAI_API_KEY` is configured), or a legal structural reranker (section title alignment, phrase overlap, query coverage, and quantitative/temporal threshold matching). Tracks initial vs. final rank transitions.
- **Cited Answers & Evidence Provenance**: Every chunk retains `doc_id`, `doc_title`, `page_start`, `page_end`, `section_number`, `section_title`, `clause_category`, and character offsets. Answers attach inline `[n]` citations mapped to verbatim evidence spans and explicitly refuse when evidence falls below the grounding threshold.
- **Clause Extraction**: Locates standard contract provisions (Limitation of Liability, Indemnification, Termination, Confidentiality, Data Protection & Breach, Intellectual Property, Governing Law & Dispute, Payment & SLA) across filtered subsets.
- **Document Summarization**: Produces structured executive summaries covering overview, key obligations, financial/temporal thresholds, risk considerations, and section outlines, with page and section citations on every entry.

## Architecture

```text
Documents (.txt / .md / .pdf / .json)
        │
        ▼
LegalChunker (page tracking + section header detection + overlapping windows + clause tagging)
        │
        ▼
EmbeddingClient + LegalIndexStore (SQLite persistence for documents, metadata, chunks, vectors)
        │
        ▼
MetadataFilter (pre-filter by doc_type, jurisdiction, party, clause, year, confidentiality)
        │
        ▼
Stage 1 Hybrid Retrieval (Dense Cosine Similarity + Okapi BM25 ──► Reciprocal Rank Fusion)
        │
        ▼
Stage 2 LegalReranker (Cross-Encoder / LLM Pointwise / Structural Legal Reranker)
        │
        ▼
LegalSynthesizer (Cited Q&A with refusal guardrails | Clause Discovery | Document Summary)
```

## Quick Start

Python 3.10+ is required. No third-party packages are needed to run the core CLI and sample corpus.

### 1. Seed the Sample Corpus

```bash
python3 legal_intelligence.py seed
```

This indexes four realistic legal documents (Master Services Agreement, GDPR/UK GDPR Data Processing Addendum, Enterprise Information Security Policy, and Mutual NDA) into `legal_index.db`.

### 2. List Indexed Documents

```bash
python3 legal_intelligence.py list-docs
```

### 3. Ask a Cited Legal Question

```bash
python3 legal_intelligence.py query \
  "What is the liability cap for data security breaches and what claims are uncapped?"
```

With metadata filters:

```bash
python3 legal_intelligence.py query \
  "How quickly must a personal data breach be reported?" \
  --jurisdiction "European Union" \
  --doc-type "Data Processing Addendum"
```

### 4. Find Relevant Clauses

```bash
python3 legal_intelligence.py clauses "Termination" --year-from 2025
```

### 5. Summarize a Document

```bash
python3 legal_intelligence.py summarize msa-northstar-2025
```

### 6. Ingest Custom Documents

Ingest a single file or a folder of contracts:

```bash
python3 legal_intelligence.py ingest ./contracts/ \
  --doc-type "Master Services Agreement" \
  --jurisdiction "New York, US" \
  --effective-date "2026-03-01" \
  --party "Acme Corp" \
  --party "Globex LLC"
```

For structured batch ingestion, pass a `.json` file containing an array of objects with keys `title`, `text`, `doc_type`, `jurisdiction`, `effective_date`, `parties`, and `confidentiality`.

## Optional Integrations

| Capability | Environment Variable / Dependency | Behavior |
| --- | --- | --- |
| Remote Embeddings & Grounded Synthesis | `OPENAI_API_KEY`, optional `OPENAI_BASE_URL`, `EMBEDDING_MODEL`, `CHAT_MODEL` | Uses remote embedding vectors, LLM-blended reranking, and generative cited answers/summaries |
| Local Dense Embeddings | `pip install sentence-transformers` + `USE_SENTENCE_TRANSFORMERS=1` | Encodes chunks with `BAAI/bge-small-en-v1.5` (override via `ST_EMBEDDING_MODEL`) |
| Cross-Encoder Reranking | `pip install sentence-transformers` + `USE_CROSS_ENCODER=1` | Reranks candidates with `cross-encoder/ms-marco-MiniLM-L-6-v2` (override via `CROSS_ENCODER_MODEL`) |
| PDF Extraction | `pip install pypdf` | Extracts page-delimited text from PDF contracts |
| HTTP REST API | `pip install fastapi uvicorn` | Exposes `/documents`, `/ingest`, `/query`, `/clauses`, and `/summarize/{doc_id}` |

Start the HTTP server:

```bash
python3 legal_intelligence.py serve --host 0.0.0.0 --port 8000
```

## Chunking & Provenance Strategy

1. **Page Boundary Tracking**: Form feeds (`\f`) and explicit markers (`--- Page N ---`, `[Page N]`) establish exact page indices before text splitting. Chunks that span a page break record both `page_start` and `page_end`.
2. **Section Hierarchy Preservation**: Regular expressions detect numbered clauses (`6. Limitation of Liability`, `4.2.1 Subprocessors`) and formal headings (`ARTICLE IV`, `SECTION 12`). Paragraphs inherit the active section reference until the next header is encountered.
3. **Overlap Windowing**: Long sections are windowed using configurable `--chunk-words` (default `180`) and `--overlap-words` (default `35`), keeping defined terms and cross-references intact without duplicating short clauses.
4. **Automatic Clause Tagging**: Each chunk is tagged against an 8-category legal taxonomy at index time so clause-level filtering and discovery remain fast even across large collections.

## Metadata Schema

Every indexed chunk stores:

- `doc_id`, `doc_title`, `doc_type`
- `jurisdiction`, `effective_date`, `effective_year`
- `parties`, `confidentiality`
- `page_start`, `page_end`
- `section_number`, `section_title`, `clause_category`
- `chunk_index`, `char_start`, `char_end`

## Design Notes & Trade-offs

- **Pre-filtering vs. Post-filtering**: Metadata filters execute prior to BM25 and dense scoring so narrow legal queries (such as single-jurisdiction or single-counterparty audits) never lose recall to high-scoring passages from unrelated contracts.
- **RRF Fusion**: Combining dense semantic rank and BM25 lexical rank via Reciprocal Rank Fusion (`k=60`) prevents scale mismatch between cosine similarity and unbounded BM25 scores while preserving exact-match recall on defined contract terms (e.g., `Net 30`, `Super-Cap`, `Article 33`).
- **Hallucination Guardrail**: If no reranked passage clears the minimum relevance threshold (`0.22`), the system declines to synthesize an answer and reports insufficient evidence instead of extrapolating.
