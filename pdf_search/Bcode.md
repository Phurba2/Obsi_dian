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


---

## 5. Create directories and files

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

```python

DB_HOST=localhost

DB_PORT=5432

DB_NAME=pdf_vector

DB_USER=furba

DB_PASSWORD=furba

```

## `.gitignore`

## config/settings.py`

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

## 10. EnablePostgreSQL extensions

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
---

## 12. `setup_db.py`

```bash
python setup_db.py
```

```text
Schema initialized in database: pdf_vector
```

---

## 13. Inspect database

```sql
psql

\l 

\c pdf_vector
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