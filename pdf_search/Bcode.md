## 1. Virtual environment

```bash
cd /path/to/this/project
python3 -m venv env
source env/bin/activate
python -m pip install --upgrade pip setuptools wheel
```

## 2. Install packages

```bash
pip install -r requirements.txt
```

```text
psycopg2-binary
pgvector
python-dotenv
PyMuPDF
nltk
numpy
sentence-transformers
scikit-learn
```

## 3. Two sides of pgvector

**Postgres side** allows:

```sql
embedding vector(384)
```

**Python side**: the `pgvector` package is installed (it is in `requirements.txt`), and queries cast parameters explicitly:

```python
1 - (c.embedding <=> %s::vector) AS score
```

Together they solve the Python <-> PostgreSQL vector problem.

> Note: this project does **not** call `register_vector(conn)` anywhere. Every vector is passed as a plain Python list (`embedding.tolist()`) which `psycopg2` adapts as an array, and the `::vector` cast converts it on the Postgres side.

### Connection test

Before doing anything else, make sure Python can actually talk to `pdf_vector`. This uses the real settings instead of hard-coded credentials:

```bash
python - <<'PY'
import psycopg2

from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD

conn = psycopg2.connect(
    host=DB_HOST,
    port=DB_PORT,
    dbname=DB_NAME,
    user=DB_USER,
    password=DB_PASSWORD,
)

print("Connected to PostgreSQL!")

with conn.cursor() as cursor:
    cursor.execute(
        "SELECT extversion FROM pg_extension WHERE extname = 'vector';"
    )
    row = cursor.fetchone()
    print(f"pgvector {row[0]}" if row else "pgvector extension is not enabled")

conn.close()
PY
```

```text
Connected to PostgreSQL!
pgvector 0.8.6
```

### Why an adapter matters (background)

Without an adapter, `psycopg2` sees a numpy array and says:

```text
can't adapt type 'numpy.ndarray'
```

That is what `register_vector()` would fix - it teaches `psycopg2` how to turn a numpy array into a PostgreSQL `vector`. This project sidesteps the problem:

```text
numpy array
    |
    v
.tolist()  ->  plain Python list
    |
    v
psycopg2 adapts it as an array
    |
    v
SQL cast  ::vector
    |
    v
PostgreSQL vector(384)
```

---

## 4. Project layout

```text
.
├── env/                    # Python virtual environment
├── index.py                # entry points: ingest_and_embed(), search()
├── setup_db.py             # applies schema.sql
├── schema.sql
├── requirements.txt
├── .gitignore
├── README.md
├── config/
│   └── settings.py         # DB + model settings, .env overrides
├── pdf/                    # put your PDFs here
└── src/
    ├── __init__.py
    ├── pdf_processor.py        # PDFStorage: discover + validate PDFs
    ├── paper_processor.py      # register PDFs in the papers table
    ├── pdf_extractor.py        # PyMuPDF text extraction
    ├── text_chunker.py         # sentence-aware chunking (768/128)
    ├── embeddings.py           # EmbeddingGenerator (all-MiniLM-L6-v2)
    ├── embedding_pipeline.py   # chunks -> embeddings -> paper_chunks
    ├── search.py               # vector / keyword / hybrid search
    └── database.py             # empty placeholder
```

There is no `scripts/` folder - everything runnable lives at the project root (`index.py`, `setup_db.py`), so all commands below assume you are in the project root.

---

## 5. Create the directories and files

Already present, but for reference this is what creates the current layout:

```bash
mkdir -p config
mkdir -p src
mkdir -p pdf

touch src/__init__.py
touch config/settings.py
touch src/pdf_processor.py
touch src/paper_processor.py
touch src/pdf_extractor.py
touch src/text_chunker.py
touch src/embeddings.py
touch src/embedding_pipeline.py
touch src/search.py
touch src/database.py
touch index.py
touch setup_db.py
touch schema.sql
touch requirements.txt
```

---

## 6. `.env`

There is no `.env` in the repository yet (it is git-ignored). Create one:

```bash
cat > .env <<'EOF'
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pdf_vector
DB_USER=furba
DB_PASSWORD=YOUR_POSTGRES_PASSWORD

# Model Configuration
EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_BATCH_SIZE=32
EMBEDDING_DEVICE=cpu

# Storage Configuration
PDF_STORAGE_PATH=pdf
CACHE_PATH=cache
LOG_PATH=logs
EOF
```

(Use `nano .env` if you prefer an editor. Put your real password in - never commit it.)

There are **no ArXiv settings** (`ARXIV_MAX_RESULTS`, `ARXIV_RATE_LIMIT` do not exist in this project).

## 7. `.gitignore`

```bash
nano .gitignore
```

```gitignore
env/
__pycache__/
*.py[cod]
.env
cache/
logs/
pdf/*.pdf
*.log
```

## 8. `config/settings.py`

```bash
nano config/settings.py
```

```python
import os
from pathlib import Path

from dotenv import load_dotenv

BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(BASE_DIR / ".env")

DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = int(os.getenv("DB_PORT", "5432"))
DB_NAME = os.getenv("DB_NAME", "pdf_vector")
DB_USER = os.getenv("DB_USER", "furba")
DB_PASSWORD = os.getenv("DB_PASSWORD")

EMBEDDING_MODEL = os.getenv("EMBEDDING_MODEL", "all-MiniLM-L6-v2")
EMBEDDING_BATCH_SIZE = int(os.getenv("EMBEDDING_BATCH_SIZE", "32"))
EMBEDDING_DEVICE = os.getenv("EMBEDDING_DEVICE", "cpu")

PDF_STORAGE_PATH = BASE_DIR / os.getenv("PDF_STORAGE_PATH", "pdf")
CACHE_PATH = BASE_DIR / os.getenv("CACHE_PATH", "cache")
LOG_PATH = BASE_DIR / os.getenv("LOG_PATH", "logs")
```

Differences from the original:

- `DB_NAME` defaults to `pdf_vector`, not `ragdb`.
- No `ARXIV_*` settings.
- `PDF_STORAGE_PATH` defaults to `pdf/`, not `data/pdfs`.

---

## 9. Verify the environment

The original tutorial used `scripts/verify.py`. There is no `scripts/` folder here - run this from the project root instead:

```bash
python - <<'PY'
import importlib

PACKAGES = {
    "psycopg2-binary": "psycopg2",
    "pgvector": "pgvector",
    "python-dotenv": "dotenv",
    "PyMuPDF": "fitz",
    "nltk": "nltk",
    "numpy": "numpy",
    "sentence-transformers": "sentence_transformers",
    "scikit-learn": "sklearn",
}

print("=" * 50)
print("Environment Verification")
print("=" * 50)
print("Checking Python packages...")

failed = []
for name, module in PACKAGES.items():
    try:
        importlib.import_module(module)
        print(f"  OK  {name}")
    except ImportError as error:
        print(f"  FAIL {name}: {error}")
        failed.append(name)

print("\nChecking PostgreSQL...")
try:
    import psycopg2
    from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD

    conn = psycopg2.connect(
        host=DB_HOST, port=DB_PORT, dbname=DB_NAME,
        user=DB_USER, password=DB_PASSWORD,
    )
    print("  OK  PostgreSQL connection")

    with conn.cursor() as cur:
        cur.execute("SELECT extversion FROM pg_extension WHERE extname = 'vector';")
        row = cur.fetchone()
    if row:
        print(f"  OK  pgvector {row[0]}")
    else:
        print("  FAIL pgvector extension is not enabled")
        failed.append("pgvector extension")
    conn.close()
except Exception as error:
    print(f"  FAIL PostgreSQL: {error}")
    failed.append("PostgreSQL")

print("\nChecking embedding model...")
try:
    from config.settings import EMBEDDING_MODEL, EMBEDDING_DEVICE
    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer(EMBEDDING_MODEL, device=EMBEDDING_DEVICE)
    print(f"  Model: {EMBEDDING_MODEL}")
    print(f"  Device: {EMBEDDING_DEVICE}")
    print(f"  OK  Embedding dimension: {model.get_sentence_embedding_dimension()}")
except Exception as error:
    print(f"  FAIL Model: {error}")
    failed.append("model")

print("\n" + "=" * 50)
print("ALL CHECKS PASSED" if not failed else f"FAILED: {failed}")
PY
```

```text
==================================================
Environment Verification
==================================================
Checking Python packages...
  OK  psycopg2-binary
  OK  pgvector
  OK  python-dotenv
  OK  PyMuPDF
  OK  nltk
  OK  numpy
  OK  sentence-transformers
  OK  scikit-learn

Checking PostgreSQL...
  OK  PostgreSQL connection
  OK  pgvector 0.8.6

Checking embedding model...
  Model: all-MiniLM-L6-v2
  Device: cpu
  OK  Embedding dimension: 384

==================================================
ALL CHECKS PASSED
```

The pipeline we are building:

```text
                  local PDF (pdf/)
                       |
                       v
                    PyMuPDF
                       |
                       v
                  text chunks
                       |
                       v
             all-MiniLM-L6-v2
                       |
                       v
              384-dimensional
                  vector
                       |
                       v
            PostgreSQL/pgvector
```

---

## 10. Enable the PostgreSQL extensions

As a PostgreSQL administrator (see README):

```bash
sudo -u postgres psql -d pdf_vector -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

`schema.sql` also creates both extensions with `IF NOT EXISTS`, but creating
them up front avoids permission errors when running as a normal user:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

```sql
\dx
```

```text
        Name    | Version | Description
----------------+---------+------------------------------------
 pg_trgm        | 1.6     | text similarity measurement and...
 plpgsql        | 1.0     | PL/pgSQL procedural language
 vector         | 0.8.6   | vector data type and ivfflat and...
```

Note: `btree_gin` from the original tutorial is **not** needed - this schema
has no GIN indexes over arrays or other btree-gin types.

---

## 11. `schema.sql`

Rather than putting SQL directly inside `setup_db.py`, the project keeps a
separate `schema.sql` at the project root:

```text
.
├── config/
│   └── settings.py
├── src/
│   └── ...
├── pdf/
├── schema.sql          <- applied by setup_db.py
├── setup_db.py
├── index.py
├── .env
└── .gitignore
```

```bash
nano schema.sql
```

```sql
-- Minimal schema for user-provided PDFs and semantic search.
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE IF NOT EXISTS papers (
    id SERIAL PRIMARY KEY,
    filename TEXT UNIQUE NOT NULL,
    title TEXT NOT NULL,
    pdf_path TEXT NOT NULL,
    pdf_processed BOOLEAN DEFAULT FALSE,
    embedding_generated BOOLEAN DEFAULT FALSE,
    processing_error TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS paper_chunks (
    id SERIAL PRIMARY KEY,
    paper_id INTEGER NOT NULL REFERENCES papers(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    chunk_text TEXT NOT NULL,
    chunk_tokens INTEGER,
    embedding vector(384),
    section_name VARCHAR(255),
    page_number INTEGER,
    char_start INTEGER,
    char_end INTEGER,
    has_math BOOLEAN DEFAULT FALSE,
    has_code BOOLEAN DEFAULT FALSE,
    has_references BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (paper_id, chunk_index)
);

CREATE INDEX IF NOT EXISTS idx_chunks_embedding
    ON paper_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
CREATE INDEX IF NOT EXISTS idx_chunks_paper_id ON paper_chunks (paper_id);
CREATE INDEX IF NOT EXISTS idx_chunks_text_trgm
    ON paper_chunks USING GIN (chunk_text gin_trgm_ops);
```

What changed versus the original tutorial's schema:

- `papers` is keyed by **`filename`** (unique), not `arxiv_id` - there are no
  `abstract`, `authors[]`, `categories[]`, `pdf_url`, `doi`, ... columns.
- No `pdf_downloaded` flag (nothing is downloaded), no `language` column.
- Dropped entirely: `authors`, `paper_authors`, `categories`, `search_history`,
  `processing_queue`.
- Kept: `vector(384)`, the HNSW cosine index, and the `pg_trgm` GIN index.

---

## 12. `setup_db.py`

At the project root (not `scripts/setup_db.py`), and no `sys.path` hack is
needed because it is already at the root:

```bash
nano setup_db.py
```

```python
"""Initialize the configured PostgreSQL database schema."""

from pathlib import Path

import psycopg2

from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD


schema = (Path(__file__).parent / "schema.sql").read_text()

with psycopg2.connect(
    host=DB_HOST,
    port=DB_PORT,
    dbname=DB_NAME,
    user=DB_USER,
    password=DB_PASSWORD,
) as connection:
    with connection.cursor() as cursor:
        cursor.execute(schema)

print(f"Schema initialized in database: {DB_NAME}")
```

```bash
python setup_db.py
```

```text
Schema initialized in database: pdf_vector
```

---

## 13. Inspect the database

```bash
psql -h localhost -U furba -d pdf_vector
```

```sql
\dt
```

```text
              List of tables
 Schema |    Name      | Type  |  Owner
--------+--------------+-------+--------
 public | paper_chunks  | table | furba
 public | papers        | table | furba
(2 rows)
```

```sql
\d papers
```

```text
                    Table "public.papers"
       Column       |           Type           | Collation | Nullable |              Default
--------------------+--------------------------+-----------+----------+-------------------------
 id                 | integer                  |           | not null | nextval('papers_id_seq'::regclass)
 filename           | text                     |           | not null |
 title              | text                     |           | not null |
 pdf_path           | text                     |           | not null |
 pdf_processed      | boolean                  |           |          | false
 embedding_generated | boolean                 |           |          | false
 processing_error   | text                     |           |          |
 created_at         | timestamp without time zone |        |          | CURRENT_TIMESTAMP
 updated_at         | timestamp without time zone |        |          | CURRENT_TIMESTAMP
Indexes:
    "papers_pkey" PRIMARY KEY, btree (id)
    "papers_filename_key" UNIQUE CONSTRAINT, btree (filename)
```

`filename` is unique, so the same PDF file is never stored twice.

```sql
\d paper_chunks
```

```text
                 Table "public.paper_chunks"
      Column      |         Type          | Collation | Nullable |                  Default
------------------+-----------------------+-----------+----------+---------------------------
 id               | integer               |           | not null | nextval('paper_chunks_id_seq'::regclass)
 paper_id         | integer               |           | not null |
 chunk_index      | integer               |           | not null |
 chunk_text       | text                  |           | not null |
 chunk_tokens     | integer               |           |          |
 embedding        | vector(384)           |           |          |
 section_name     | character varying(255)|           |          |
 page_number      | integer               |           |          |
 char_start       | integer               |           |          |
 char_end         | integer               |           |          |
 has_math         | boolean               |           |          | false
 has_code         | boolean               |           |          | false
 has_references   | boolean               |           |          | false
 created_at       | timestamp without time zone |     |          | CURRENT_TIMESTAMP
Indexes:
    "paper_chunks_pkey" PRIMARY KEY, btree (id)
    "idx_chunks_embedding" hnsw (embedding vector_cosine_ops) WITH (m='16', ef_construction='64')
    "idx_chunks_paper_id" btree (paper_id)
    "idx_chunks_text_trgm" gin (chunk_text gin_trgm_ops)
    "paper_chunks_paper_id_chunk_index_key" UNIQUE CONSTRAINT, btree (paper_id, chunk_index)
Foreign-key constraints:
    "paper_chunks_paper_id_fkey" FOREIGN KEY (paper_id) REFERENCES papers(id) ON DELETE CASCADE
```

```text
                    ┌──────────────────┐
                    │     papers       │
                    │──────────────────│
                    │ id               │
                    │ filename (unique)│
                    │ title            │
                    │ pdf_path         │
                    │ pdf_processed    │
                    │ embedding_generated
                    └────────┬─────────┘
                             │
                         paper_id
                             │
                ┌────────────▼────────────┐
                │      paper_chunks       │
                │─────────────────────────│
                │ id                      │
                │ paper_id                │
                │ chunk_index             │
                │ chunk_text              │
                │ embedding vector(384)   │
                │ section_name            │
                │ page_number             │
                └─────────────────────────┘
```

```text
pdf_vector
│
├── papers          <- PDF filename, title, path, processing status
└── paper_chunks    <- text chunks + 384-dim embeddings
```

There are intentionally **no** `authors`, `categories`, `search_history`, or
`processing_queue` tables: the project indexes local PDFs, so all it knows
about a file is its name, its path, and the text inside it.

---

## 14. How a paper becomes searchable

```text
Paper
│
├── Abstract
├── Introduction
├── Related Work
├── Methods
├── Experiments
├── Results
└── Conclusion
```

```text
paper_chunks

chunk 0 → "The paper introduces..."
chunk 1 → "Previous research..."
chunk 2 → "Our proposed method..."
chunk 3 → "The experiment shows..."
chunk 4 → "The results indicate..."
```

```text
chunk text
    │
    ▼
all-MiniLM-L6-v2
    │
    ▼
[0.12, -0.43, 0.08, ... 384 numbers ...]
    │
    ▼
PostgreSQL vector(384)
```

Chunking targets (already implemented in `src/text_chunker.py`):

```text
Target chunk size:    768 tokens
Overlap:              128 tokens
```

---

## 15. HNSW index (already implemented)

```sql
CREATE INDEX idx_chunks_embedding
ON paper_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

```text
idx_chunks_embedding
hnsw (embedding vector_cosine_ops)
WITH (m='16', ef_construction='64')
```

```text
                    HNSW
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      chunk         chunk        chunk
       ↓             ↓             ↓
    vector         vector       vector
       \             |            /
        \            |           /
         └───────────┼───────────┘
                     │
                fast search
```

Instead of comparing your query against every vector one by one, HNSW builds
a graph that lets PostgreSQL navigate toward nearby vectors.

`filename` must be **unique**, so the same PDF file is never stored twice.

---

## 16. Two search paths, combined

```text
SEMANTIC SEARCH
      │
      ▼
embedding vector(384)
      │
      ▼
HNSW index
      │
      ▼
similar chunks
```

```text
KEYWORD / TEXT SIMILARITY
      │
      ▼
chunk_text
      │
      ▼
pg_trgm
      │
      ▼
GIN index
```

```text
                  USER QUERY
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     Semantic search          Text search
          │                       │
       pgvector                pg_trgm
          │                       │
          └───────────┬───────────┘
                      ▼
          0.7 * vector + 0.3 * keyword
                      │
                      ▼
             ranked results (SearchMode.HYBRID)
```

---

## 17. Processing status flags

The original pipeline had a download step (`pdf_downloaded`). This project
has none - you drop files into `pdf/` yourself:

```text
PDF placed in pdf/
     │
     │  PaperProcessor.ingest_pdfs()
     ▼
paper row created
(pdf_processed = FALSE, embedding_generated = FALSE)
     │
     │  EmbeddingPipeline: extract + chunk
     ▼
pdf_processed = TRUE
     │
     │  store vectors in paper_chunks
     ▼
embedding_generated = TRUE
```

Failed papers get `processing_error` filled in instead of raising.

---

## 18. The database at a glance

```text
                         PostgreSQL
┌──────────────────────────────────────────────────────┐
│                                                       │
│  papers                                               │
│    │  filename (unique)                               │
│    │  title, pdf_path                                 │
│    │  pdf_processed / embedding_generated             │
│    │                                                  │
│    └── paper_chunks                                   │
│           ├── chunk_text                              │
│           ├── chunk_tokens                            │
│           ├── section/page metadata                   │
│           │                                           │
│           └── embedding vector(384)                   │
│                  │                                    │
│              HNSW index                               │
│                                                       │
│  pgvector ✓   pg_trgm ✓                               │
└──────────────────────────────────────────────────────┘
```
