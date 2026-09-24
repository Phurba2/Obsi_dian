```bash
mkdir -p ~/arxiv_search
cd ~/arxiv_search
```

---

```bash
python3 --version
```

You should have Python 3.8+.


---

```bash
python3 -m venv .venv
```

This creates:

```text
arxiv_search/
└── .venv/
```

`.venv` directory contains isolated Python environment for this project.

---

```bash
source .venv/bin/activate
```


```text
                 Python
                   │
          ┌────────┴────────┐
          │                 │
       System            .venv
       Python          project Python
                           │
                     our packages
```

---

```bash
python -m pip install --upgrade pip
```

---

```bash
pip install \
  arxiv==2.1.0 \
  PyMuPDF==1.23.8 \
  psycopg2-binary==2.9.9 \
  sentence-transformers==2.2.2 \
  requests==2.31.0 \
  numpy==1.24.3 \
  tqdm==4.66.1 \
  python-dotenv==1.0.0 \
  pgvector
```


---

# Two pgvector

Postgres side allow:

```sql
embedding vector(384)
```

Python side allow:

```python
from pgvector.psycopg2 import register_vector

register_vector(conn)
```

Together they solve Python ↔ PostgreSQL vector problem.

---

Before doing embeddings or downloading ArXiv papers, let's make sure Python can actually talk to your `ragdb`.

Create:

```bash
nano test_db.py
```


```python
import psycopg2
from pgvector.psycopg2 import register_vector


conn = psycopg2.connect(
    dbname="ragdb",
    user="furba",
    password="YOUR_PASSWORD",
    host="localhost",
    port=5432,
)

register_vector(conn)

print("Connected to PostgreSQL!")
print("pgvector registered!")

conn.close()
```

```bash
python test_db.py
```

```text
Connected to PostgreSQL!
pgvector registered!
```

---

# Why `register_vector()` matters

Without it:

```python
embedding = numpy_array

cursor.execute(
    "INSERT INTO paper_chunks (embedding) VALUES (%s)",
    (embedding,)
)
```

`psycopg2` sees:

```text
numpy.ndarray
```

and essentially says:

```text
"I don't know how to turn this
Python object into PostgreSQL vector."
```

Hence:

```text
can't adapt type 'numpy.ndarray'
```

After:

```python
register_vector(conn)
```

pgvector Python package teaches `psycopg2` how to handle vectors:

```text
numpy array
    │
    ▼
pgvector Python adapter
    │
    ▼
psycopg2
    │
    ▼
PostgreSQL
    │
    ▼
vector(...)
```

---


```python
                 ArXiv RAG Project
                       │
        ┌──────────────┴──────────────┐
        │                             │
     Python                         PostgreSQL
        │                             │
   ┌────┴─────┐                  ┌────┴─────┐
   │          │                  │          │
ArXiv      PyMuPDF          PostgreSQL   pgvector
   │          │                  │          │
papers      text              data      embeddings
             │                            │
             └──────────┬─────────────────┘
                        │
                  next: embedding
                     pipeline
```

---

| pip install \\                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| arxiv==2.1.0 \\                                                                                                                                              |
| PyMuPDF==1.23.8 \\                                                                                                                                           |
| psycopg2-binary==2.9.9 \\                                                                                                                                    |
| sentence-transformers==2.2.2 \\                                                                                                                              |
| requests==2.31.0 \\                                                                                                                                          |
| numpy==1.24.3 \\                                                                                                                                             |
| tqdm==4.66.1 \\                                                                                                                                              |
| python-dotenv==1.0.0 \\                                                                                                                                      |
| pgvector                                                                                                                                                     |
| Collecting arxiv==2.1.0                                                                                                                                      |
| Downloading arxiv-2.1.0-py3-none-any.whl.metadata (6.1 kB)                                                                                                   |
| Collecting PyMuPDF==1.23.8                                                                                                                                   |
| Downloading PyMuPDF-1.23.8-cp312-none-manylinux2014_x86_64.whl.metadata (3.4 kB)                                                                             |
| Collecting psycopg2-binary==2.9.9                                                                                                                            |
| Downloading psycopg2_binary-2.9.9-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl.metadata (4.4 kB)                                               |
| Collecting sentence-transformers==2.2.2                                                                                                                      |
| Downloading sentence-transformers-2.2.2.tar.gz (85 kB)                                                                                                       |
| Installing build dependencies ... done                                                                                                                       |
| Getting requirements to build wheel ... done                                                                                                                 |
| Preparing metadata (pyproject.toml) ... done                                                                                                                 |
| Collecting requests==2.31.0                                                                                                                                  |
| Downloading requests-2.31.0-py3-none-any.whl.metadata (4.6 kB)                                                                                               |
| Collecting numpy==1.24.3                                                                                                                                     |
| Downloading numpy-1.24.3.tar.gz (10.9 MB)                                                                                                                    |
| ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 10.9/10.9 MB 15.3 MB/s  0:00:00                                                                                     |
| Installing build dependencies ... done                                                                                                                       |
| Getting requirements to build wheel ... done                                                                                                                 |
| ERROR: Exception:                                                                  |

---

**old versions** and `numpy==1.24.3` does not have a normal wheel for Python 3.12, so `pip` tries to build it from source.

```text
Downloading numpy-1.24.3.tar.gz
```

instead of a `.whl` file.

Then the build environment fails with:

```text
Cannot import 'setuptools.build_meta'
```

## Easiest solution

use Python 3.12 environment and install compatible current packages.

Upgrade build tools:

```bash
python -m pip install --upgrade pip setuptools wheel
```

---

# Install packages without old version pins

```bash
pip install \
  arxiv \
  PyMuPDF \
  psycopg2-binary \
  sentence-transformers \
  requests \
  numpy \
  tqdm \
  python-dotenv \
  pgvector
```

This lets `pip` choose versions that have Python 3.12 support.

---

# installation failed, remove venv **pip cache** and partially installed packages, then start fresh.


```bash
rm -rf ~/arxiv_search/env
```


---

## Delete pip's download/cache

```bash
python3 -m pip cache purge
```

```text
Files removed: ...
```

removes cached packages such as :

```text
numpy-1.24.3.tar.gz
sentence-transformers-2.2.2.tar.gz
```

---

```bash
cd ~/arxiv_search
ls -la
```

---

```bash
python3 -m venv env
```

```bash
source env/bin/activate
```


```text
(env) furba@fu:~/arxiv_search$
```

---

```bash
python -m pip install --upgrade pip setuptools wheel
```

---


```bash
python --version
pip --version
```


```text
CLEAN
  │
  ├── old env              → deleted
  ├── failed packages     → deleted
  ├── pip cache           → purged
  │
  ▼
NEW env
  │
  ├── Python 3.12
  ├── fresh pip
  ├── fresh setuptools
  └── fresh wheel
        │
        ▼
   install dependencies
```

---

| ls -la |
|--------|
| total 8 |
| drwxrwxr-x  2 furba furba 4096 Sep 22 13:42 . |
| drwxr-x--- 35 furba furba 4096 Sep 22 13:33 .. |
| furba@fu:~/arxiv_search$ |

---

| python --version |
|------------------|
| Python 3.12.3 |
| (env) furba@fu:~/arxiv_search$ pip --version |
| pip 24.0 from /home/furba/arxiv_search/env/lib/python3.12/site-packages/pip (python 3.12) |
| (env) furba@fu:~/arxiv_search$ |

---

```text
Python 3.12.3
pip 24.0
```


```text
(env) furba@fu:~/arxiv_search$
```

Let's upgrade the packaging tools first.

```bash
python -m pip install --upgrade pip setuptools wheel
```


```bash
pip --version
```


---

This time, **don't use the old version pins**.

```bash
pip install \
  arxiv \
  PyMuPDF \
  psycopg2-binary \
  sentence-transformers \
  requests \
  numpy \
  tqdm \
  python-dotenv \
  pgvector
```

This allows pip to select versions compatible with **Python 3.12.3**.


```text
numpy==1.24.3
        │
        ▼
No suitable Python 3.12 wheel
        │
        ▼
pip tries to compile NumPy
        │
        ▼
build fails
```


```text
Python 3.12.3
      │
      ▼
pip chooses compatible versions
      │
      ├── NumPy
      ├── sentence-transformers
      ├── PyMuPDF
      ├── psycopg2
      ├── pgvector
      └── others
```

---

**don't put your PostgreSQL password in source code**. We'll keep it in `.env`, and we'll use `furba` / `ragdb` database that we already created.


```text
arxiv_search/
│
├── env/                    # Python virtual environment
│
├── config/
│   └── settings.py         # Application settings
│
├── .env                    # Passwords/configuration
│
├── data/
│   ├── pdfs/
│   │   ├── 2024/
│   │   └── 2025/
│   ├── cache/
│   └── logs/
│
├── src/
│   ├── __init__.py
│   ├── arxiv_client.py
│   ├── pdf_processor.py
│   ├── embeddings.py
│   ├── database.py
│   └── search.py
│
└── scripts/
    ├── setup_db.py
    ├── fetch_papers.py
    └── build_index.py
```

---

# Create directories

```text
(env) furba@fu:~/arxiv_search$
```


```bash
mkdir -p config
mkdir -p data/pdfs/2024
mkdir -p data/pdfs/2025
mkdir -p data/cache
mkdir -p data/logs
mkdir -p src
mkdir -p scripts
```


```bash
touch src/__init__.py
touch config/settings.py
touch src/arxiv_client.py
touch src/pdf_processor.py
touch src/embeddings.py
touch src/database.py
touch src/search.py
touch scripts/setup_db.py
touch scripts/fetch_papers.py
touch scripts/build_index.py
```

---

```bash
nano .env
```

Put this inside:

```dotenv
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ragdb
DB_USER=furba
DB_PASSWORD=YOUR_POSTGRES_PASSWORD

# ArXiv Configuration
ARXIV_MAX_RESULTS=100
ARXIV_RATE_LIMIT=3

# Model Configuration
EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_BATCH_SIZE=32
EMBEDDING_DEVICE=cpu

# Storage Configuration
PDF_STORAGE_PATH=./data/pdfs
CACHE_PATH=./data/cache
LOG_PATH=./data/logs
```

---

```bash
nano .gitignore
```

Put:

```gitignore
.env
env/
__pycache__/
*.pyc

data/pdfs/
data/cache/
data/logs/
```

This prevents you from accidentally committing your password and potentially huge PDF files to Git.

---

```bash
nano config/settings.py
```

Put:

```python
import os
from pathlib import Path

from dotenv import load_dotenv


# Project root directory
BASE_DIR = Path(__file__).resolve().parent.parent

# Load .env
load_dotenv(BASE_DIR / ".env")


# Database
DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = int(os.getenv("DB_PORT", "5432"))
DB_NAME = os.getenv("DB_NAME", "ragdb")
DB_USER = os.getenv("DB_USER", "furba")
DB_PASSWORD = os.getenv("DB_PASSWORD")


# ArXiv
ARXIV_MAX_RESULTS = int(os.getenv("ARXIV_MAX_RESULTS", "100"))
ARXIV_RATE_LIMIT = float(os.getenv("ARXIV_RATE_LIMIT", "3"))


# Embedding model
EMBEDDING_MODEL = os.getenv(
    "EMBEDDING_MODEL",
    "all-MiniLM-L6-v2",
)

EMBEDDING_BATCH_SIZE = int(
    os.getenv("EMBEDDING_BATCH_SIZE", "32")
)

EMBEDDING_DEVICE = os.getenv(
    "EMBEDDING_DEVICE",
    "cpu",
)


# Storage
PDF_STORAGE_PATH = BASE_DIR / os.getenv(
    "PDF_STORAGE_PATH",
    "data/pdfs",
)

CACHE_PATH = BASE_DIR / os.getenv(
    "CACHE_PATH",
    "data/cache",
)

LOG_PATH = BASE_DIR / os.getenv(
    "LOG_PATH",
    "data/logs",
)
```

---

# Why do we need `settings.py`?

Think of it as a bridge:

```text
.env
 │
 │  DB_PASSWORD=...
 │  DB_NAME=ragdb
 │
 ▼
settings.py
 │
 │  DB_PASSWORD
 │  DB_NAME
 │
 ├──────────────┐
 ▼              ▼
database.py    embeddings.py
```

`.env` contains the actual configuration.

`settings.py` reads that configuration and turns it into Python variables.

Your other programs don't need to know where the configuration came from.

---

```bash
nano scripts/verify.py
```

Put:

```python
#!/usr/bin/env python3

"""Verify the ArXiv search environment."""

import sys


def check_imports():
    """Check required Python packages."""

    packages = {
        "arxiv": "arxiv",
        "PyMuPDF": "fitz",
        "psycopg2": "psycopg2",
        "sentence-transformers": "sentence_transformers",
        "requests": "requests",
        "numpy": "numpy",
        "tqdm": "tqdm",
        "python-dotenv": "dotenv",
        "pgvector": "pgvector",
    }

    print("Checking Python packages...")

    failed = []

    for name, module in packages.items():
        try:
            __import__(module)
            print(f"  OK  {name}")
        except ImportError as e:
            print(f"  FAIL {name}: {e}")
            failed.append(name)

    return len(failed) == 0


def check_postgresql():
    """Check PostgreSQL and pgvector."""

    import psycopg2

    from config.settings import (
        DB_HOST,
        DB_PORT,
        DB_NAME,
        DB_USER,
        DB_PASSWORD,
    )

    print("\nChecking PostgreSQL...")

    try:
        conn = psycopg2.connect(
            host=DB_HOST,
            port=DB_PORT,
            dbname=DB_NAME,
            user=DB_USER,
            password=DB_PASSWORD,
        )

        print("  OK  PostgreSQL connection")

        cursor = conn.cursor()

        cursor.execute(
            """
            SELECT extversion
            FROM pg_extension
            WHERE extname = 'vector';
            """
        )

        result = cursor.fetchone()

        if result:
            print(f"  OK  pgvector {result[0]}")
        else:
            print("  FAIL pgvector extension is not enabled")
            conn.close()
            return False

        cursor.close()
        conn.close()

        return True

    except Exception as e:
        print(f"  FAIL PostgreSQL: {e}")
        return False


def check_model_download():
    """Check that the embedding model can be loaded."""

    print("\nChecking embedding model...")

    from config.settings import (
        EMBEDDING_MODEL,
        EMBEDDING_DEVICE,
    )

    print(f"  Model: {EMBEDDING_MODEL}")
    print(f"  Device: {EMBEDDING_DEVICE}")

    try:
        from sentence_transformers import SentenceTransformer

        model = SentenceTransformer(
            EMBEDDING_MODEL,
            device=EMBEDDING_DEVICE,
        )

        dimension = model.get_sentence_embedding_dimension()

        print("  OK  Model loaded")
        print(f"  OK  Embedding dimension: {dimension}")

        return True

    except Exception as e:
        print(f"  FAIL Model: {e}")
        return False


def main():
    """Run all checks."""

    print("=" * 50)
    print("ArXiv Search Environment Verification")
    print("=" * 50)

    results = [
        check_imports(),
        check_postgresql(),
        check_model_download(),
    ]

    print("\n" + "=" * 50)

    if all(results):
        print("ALL CHECKS PASSED")
        return 0

    print("SOME CHECKS FAILED")
    return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

```bash
python scripts/verify.py
```


```text
==================================================
ArXiv Search Environment Verification
==================================================
Checking Python packages...
  OK  arxiv
  OK  PyMuPDF
  OK  psycopg2
  OK  sentence-transformers
  OK  requests
  OK  numpy
  OK  tqdm
  OK  python-dotenv
  OK  pgvector

Checking PostgreSQL...
  OK  PostgreSQL connection
  OK  pgvector 0.8.6

Checking embedding model...
  Model: all-MiniLM-L6-v2
  Device: cpu
  OK  Model loaded
  OK  Embedding dimension: 384

==================================================
ALL CHECKS PASSED
```


```text
                   ArXiv PDF
                       │
                       ▼
                  PyMuPDF
                       │
                       ▼
                   text chunks
                       │
                       ▼
          all-MiniLM-L6-v2
                       │
                       ▼
              384-dimensional
                  vector
                       │
                       ▼
            PostgreSQL/pgvector
```

---

| (env) furba@fu:~/arxiv_search$ python scripts/verify.py |
|----------------------------------------------------------|
| ================================================== |
| ArXiv Search Environment Verification |
| ================================================== |
| Checking Python packages... |
|   OK  arxiv |
| warning: The `fitz` API is deprecated and will be removed in future. Use `import pymupdf` instead. |
|   OK  PyMuPDF |
|   OK  psycopg2 |
|   OK  sentence-transformers |
|   OK  requests |
|   OK  numpy |
|   OK  tqdm |
|   OK  python-dotenv |
|   OK  pgvector |
| Traceback (most recent call last): |
|   File "/home/furba/arxiv_search/scripts/verify.py", line 150, in <module> |
|     sys.exit(main()) |
|              ^^^^^^ |
|   File "/home/furba/arxiv_search/scripts/verify.py", line 135, in main |
|     check_postgresql(), |
|     ^^^^^^^^^^^^^^^^^^ |
|   File "/home/furba/arxiv_search/scripts/verify.py", line 43, in check_postgresql |
|     from config.settings import ( |
| ModuleNotFoundError: No module named 'config' |
| (env) furba@fu:~/arxiv_search$ |

---

```bash
python scripts/verify.py
```

Python starts the script from:

```text
~/arxiv_search/scripts/verify.py
```

So it doesn't automatically treat:

```text
~/arxiv_search/
```

as an import location for:

```python
from config.settings import ...
```

Your project looks roughly like:

```text
arxiv_search/
├── config/
│   └── settings.py
├── scripts/
│   └── verify.py   ← Python is running this
├── src/
└── .env
```

We can fix this cleanly by telling `verify.py` where the project root is.

## Step 1 — Edit `scripts/verify.py`

replace:

```python
import sys
```

with:

```python
import sys
from pathlib import Path

# Add the project root (arxiv_search/) to Python's import path
PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))
```

```python
Path(__file__).resolve()
```

```text
/home/furba/arxiv_search/scripts/verify.py
```

```python
.parent
```

```text
/home/furba/arxiv_search/scripts
```

```python
.parent
```

```text
/home/furba/arxiv_search
```

```text
/home/furba/arxiv_search
        │
        ├── config/
        │   └── settings.py   ← Python can find this
        │
        ├── scripts/
        └── src/
```


```bash
python scripts/verify.py
```

----

```bash
sudo -u postgres psql -d ragdb
```

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;
```

```sql
\dx
```

```text
Name       | Version | Description
-----------+---------+--------------------------------
btree_gin  | ...     | support for indexing common...
pg_trgm    | ...     | text similarity measurement...
vector     | 0.8.6   | vector data type and ivfflat...
```

```sql
\q
```

---

Rather than putting SQL directly into `setup_db.py` creating a separate SQL file.

```text
arxiv_search/
├── config/
│   └── settings.py
│
├── data/
│   ├── pdfs/
│   ├── cache/
│   └── logs/
│
├── src/
│   ├── ...
│
├── scripts/
│   ├── setup_db.py
│   └── verify.py
│
├── schema.sql          ← NEW
├── .env
└── .gitignore
```

---

```bash
nano schema.sql
```


```sql
-- =========================================================
-- ArXiv Research Paper Database
-- =========================================================

-- Enable required PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;


-- =========================================================
-- Main papers table
-- =========================================================

CREATE TABLE IF NOT EXISTS papers (
    id SERIAL PRIMARY KEY,
    arxiv_id VARCHAR(50) UNIQUE NOT NULL,
    title TEXT NOT NULL,
    abstract TEXT,
    authors TEXT[],
    categories TEXT[],
    primary_category VARCHAR(50),
    published_date DATE,
    updated_date DATE,
    pdf_url TEXT,
    comment TEXT,
    journal_ref TEXT,
    doi VARCHAR(100),

    -- Processing status
    pdf_downloaded BOOLEAN DEFAULT FALSE,
    pdf_processed BOOLEAN DEFAULT FALSE,
    embedding_generated BOOLEAN DEFAULT FALSE,
    processing_error TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


-- Indexes for common paper queries

CREATE INDEX IF NOT EXISTS idx_papers_published_date
ON papers (published_date DESC);

CREATE INDEX IF NOT EXISTS idx_papers_categories
ON papers USING GIN (categories);

CREATE INDEX IF NOT EXISTS idx_papers_authors
ON papers USING GIN (authors);


-- =========================================================
-- Paper chunks table
-- =========================================================

CREATE TABLE IF NOT EXISTS paper_chunks (
    id SERIAL PRIMARY KEY,

    paper_id INTEGER
        REFERENCES papers(id)
        ON DELETE CASCADE,

    chunk_index INTEGER NOT NULL,

    chunk_text TEXT NOT NULL,

    chunk_tokens INTEGER,

    -- all-MiniLM-L6-v2 produces 384-dimensional vectors
    embedding vector(384),

    -- Metadata
    section_name VARCHAR(255),
    page_number INTEGER,
    char_start INTEGER,
    char_end INTEGER,

    -- Quality indicators
    has_math BOOLEAN DEFAULT FALSE,
    has_code BOOLEAN DEFAULT FALSE,
    has_references BOOLEAN DEFAULT FALSE,

    language VARCHAR(10) DEFAULT 'en',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- A paper cannot have two chunks with the same position
    UNIQUE (paper_id, chunk_index)
);


-- Vector similarity index

CREATE INDEX IF NOT EXISTS idx_chunks_embedding
ON paper_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);


-- Find chunks belonging to a paper

CREATE INDEX IF NOT EXISTS idx_chunks_paper_id
ON paper_chunks (paper_id);


-- Full-text similarity using trigram

CREATE INDEX IF NOT EXISTS idx_chunks_text_trgm
ON paper_chunks
USING GIN (chunk_text gin_trgm_ops);


-- =========================================================
-- Authors table
-- =========================================================

CREATE TABLE IF NOT EXISTS authors (
    id SERIAL PRIMARY KEY,

    name TEXT NOT NULL,

    -- Normalized version for matching
    normalized_name TEXT,

    affiliation TEXT,
    orcid VARCHAR(50),
    email VARCHAR(255),

    UNIQUE (normalized_name)
);


CREATE INDEX IF NOT EXISTS idx_authors_name_trgm
ON authors
USING GIN (name gin_trgm_ops);


-- =========================================================
-- Paper ↔ Author relationship
-- =========================================================

CREATE TABLE IF NOT EXISTS paper_authors (
    paper_id INTEGER
        REFERENCES papers(id)
        ON DELETE CASCADE,

    author_id INTEGER
        REFERENCES authors(id)
        ON DELETE CASCADE,

    author_position INTEGER,

    is_corresponding BOOLEAN DEFAULT FALSE,

    PRIMARY KEY (paper_id, author_id)
);


CREATE INDEX IF NOT EXISTS idx_paper_authors_author_id
ON paper_authors (author_id);


CREATE INDEX IF NOT EXISTS idx_paper_authors_paper_id_position
ON paper_authors (paper_id, author_position);


-- =========================================================
-- ArXiv categories
-- =========================================================

CREATE TABLE IF NOT EXISTS categories (
    code VARCHAR(20) PRIMARY KEY,

    name TEXT NOT NULL,

    description TEXT,

    parent_category VARCHAR(20)
);


CREATE INDEX IF NOT EXISTS idx_categories_parent_category
ON categories (parent_category);


-- =========================================================
-- Search history
-- =========================================================

CREATE TABLE IF NOT EXISTS search_history (
    id SERIAL PRIMARY KEY,

    query_text TEXT NOT NULL,

    query_embedding vector(384),

    result_count INTEGER,

    execution_time_ms INTEGER,

    filters JSONB,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


CREATE INDEX IF NOT EXISTS idx_search_history_created_at
ON search_history (created_at DESC);


-- Vector index for search history

CREATE INDEX IF NOT EXISTS idx_search_history_query_embedding
ON search_history
USING hnsw (query_embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);


-- =========================================================
-- Processing queue
-- =========================================================

CREATE TABLE IF NOT EXISTS processing_queue (
    id SERIAL PRIMARY KEY,

    paper_id INTEGER
        REFERENCES papers(id)
        ON DELETE CASCADE,

    operation VARCHAR(50) NOT NULL,

    status VARCHAR(20) DEFAULT 'pending',

    priority INTEGER DEFAULT 0,

    retry_count INTEGER DEFAULT 0,

    error_message TEXT,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    started_at TIMESTAMP,

    completed_at TIMESTAMP
);


-- Queue lookup indexes

CREATE INDEX IF NOT EXISTS idx_queue_status_priority
ON processing_queue (status, priority DESC);


CREATE INDEX IF NOT EXISTS idx_queue_paper_id
ON processing_queue (paper_id);


CREATE INDEX IF NOT EXISTS idx_queue_operation_status
ON processing_queue (operation, status);
```


---

```bash
nano scripts/setup_db.py
```

```python
#!/usr/bin/env python3

from pathlib import Path

import psycopg2

from config.settings import (
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_USER,
    DB_PASSWORD,
)


def main():
    """Create the ArXiv database schema."""

    project_root = Path(__file__).resolve().parent.parent
    schema_file = project_root / "schema.sql"

    print("Connecting to PostgreSQL...")

    conn = psycopg2.connect(
        host=DB_HOST,
        port=DB_PORT,
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
    )

    print("Connected.")

    schema = schema_file.read_text()

    with conn.cursor() as cursor:
        cursor.execute(schema)

    conn.commit()
    conn.close()

    print("Database schema created successfully.")


if __name__ == "__main__":
    main()
```

---

```text
~/arxiv_search
```

```bash
python scripts/setup_db.py
```

```text
Connecting to PostgreSQL...
Connected.
Database schema created successfully.
```

---

```bash
psql -d ragdb
```

```sql
\dt
```

```text
papers
paper_chunks
authors
paper_authors
categories
search_history
processing_queue
```

```sql
\d papers
```

```sql
\d paper_chunks
```

---

```text
                    ┌─────────────────┐
                    │     papers      │
                    │─────────────────│
                    │ id              │
                    │ arxiv_id        │
                    │ title           │
                    │ abstract        │
                    │ authors[]       │
                    │ categories[]    │
                    │ published_date  │
                    └────────┬────────┘
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

---

| python scripts/setup_db.py |
|----------------------------|
| Traceback (most recent call last): |
|   File "/home/furba/arxiv_search/scripts/setup_db.py", line 7, in <module> |
|     from config.settings import ( |
| ModuleNotFoundError: No module named 'config' |
| (env) furba@fu:~/arxiv_search$ |

---

```bash
nano scripts/setup_db.py
```

change:

```python
#!/usr/bin/env python3

from pathlib import Path

import psycopg2
```

to:

```python
#!/usr/bin/env python3

import sys
from pathlib import Path

# Add the project root (arxiv_search/) to Python's import path
PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

import psycopg2
```

```bash
python scripts/setup_db.py
```

---

| ragdb=> \dt |
|-------------|
|               List of tables |
|  Schema \|       Name       \| Type  \| Owner |
| --------+------------------+-------+------- |
|  public \| authors          \| table \| furba |
|  public \| categories       \| table \| furba |
|  public \| paper_authors    \| table \| furba |
|  public \| paper_chunks     \| table \| furba |
|  public \| papers           \| table \| furba |
|  public \| processing_queue \| table \| furba |
|  public \| search_history   \| table \| furba |
| (7 rows) |
|  |
| ragdb=> |

---

```
+--------+------------------+-------+-------+
| Schema |       Name       | Type  | Owner |
+--------+------------------+-------+-------+
| public | authors          | table | furba |
| public | categories       | table | furba |
| public | paper_authors    | table | furba |
| public | paper_chunks     | table | furba |
| public | papers           | table | furba |
| public | processing_queue | table | furba |
| public | search_history   | table | furba |
+--------+------------------+-------+-------+
```

```text
ragdb
│
├── papers              ← paper metadata
├── paper_chunks        ← text chunks + 384-dim embeddings
├── authors             ← author information
├── paper_authors       ← paper ↔ author relationship
├── categories          ← ArXiv categories
├── search_history      ← previous searches
└── processing_queue    ← PDF/embedding processing jobs
```

```text
ragdb=>
```

```sql
\d papers
```

```sql
\d paper_chunks
```

verify extensions:

```sql
\dx
```

| furba@fu:~$ psql -d ragdb               |
| --------------------------------------- |
| psql (18.6 (Ubuntu 18.6-1.pgdg24.04+2)) |
| Type "help" for help.                   |
|                                         |
| ragdb=> \dt                             |
| List of tables                          |
|                                         |
| ragdb=> \d papers                       |
| Table "public.papers"                   |
|                                         |
| ragdb=> \dx                             |
| List of installed extensions            |
|                                         |
| ragdb=>                                 |

Here are the database table definitions and extensions formatted cleanly in ASCII tables:

  

### Table `public.papers`

```
+---------------------+-----------------------------+-----------+----------+------------------------------------+
|       Column        |            Type             | Collation | Nullable |              Default               |
+---------------------+-----------------------------+-----------+----------+------------------------------------+
| id                  | integer                     |           | not null | nextval('papers_id_seq'::regclass) |
| arxiv_id            | character varying(50)       |           | not null |                                    |
| title               | text                        |           | not null |                                    |
| abstract            | text                        |           |          |                                    |
| authors             | text[]                      |           |          |                                    |
| categories          | text[]                      |           |          |                                    |
| primary_category    | character varying(50)       |           |          |                                    |
| published_date      | date                        |           |          |                                    |
| updated_date        | date                        |           |          |                                    |
| pdf_url             | text                        |           |          |                                    |
| comment             | text                        |           |          |                                    |
| journal_ref         | text                        |           |          |                                    |
| doi                 | character varying(100)      |           |          |                                    |
| pdf_downloaded      | boolean                     |           |          | false                              |
| pdf_processed       | boolean                     |           |          | false                              |
| embedding_generated | boolean                     |           |          | false                              |
| processing_error    | text                        |           |          |                                    |
| created_at          | timestamp without time zone |           |          | CURRENT_TIMESTAMP                  |
| updated_at          | timestamp without time zone |           |          | CURRENT_TIMESTAMP                  |
+---------------------+-----------------------------+-----------+----------+------------------------------------+
```

- **Indexes:**
    - `"papers_pkey" PRIMARY KEY, btree (id)`
    - `"idx_papers_authors" gin (authors)`
    - `"idx_papers_categories" gin (categories)`
    - `"idx_papers_published_date" btree (published_date DESC)`
    - `"papers_arxiv_id_key" UNIQUE CONSTRAINT, btree (arxiv_id)`

- **Referenced by:**
    - `TABLE "paper_authors" CONSTRAINT "paper_authors_paper_id_fkey" FOREIGN KEY (paper_id) REFERENCES papers(id) ON DELETE CASCADE`
    - `TABLE "paper_chunks" CONSTRAINT "paper_chunks_paper_id_fkey" FOREIGN KEY (paper_id) REFERENCES papers(id) ON DELETE CASCADE`
    - `TABLE "processing_queue" CONSTRAINT "processing_queue_paper_id_fkey" FOREIGN KEY (paper_id) REFERENCES papers(id) ON DELETE CASCADE`

### Table `public.paper_chunks`

```
+----------------+-----------------------------+-----------+----------+------------------------------------------+
|     Column     |            Type             | Collation | Nullable |                 Default                  |
+----------------+-----------------------------+-----------+----------+------------------------------------------+
| id             | integer                     |           | not null | nextval('paper_chunks_id_seq'::regclass) |
| paper_id       | integer                     |           |          |                                          |
| chunk_index    | integer                     |           | not null |                                          |
| chunk_text     | text                        |           | not null |                                          |
| chunk_tokens   | integer                     |           |          |                                          |
| embedding      | vector(384)                 |           |          |                                          |
| section_name   | character varying(255)      |           |          |                                          |
| page_number    | integer                     |           |          |                                          |
| char_start     | integer                     |           |          |                                          |
| char_end       | integer                     |           |          |                                          |
| has_math       | boolean                     |           |          | false                                    |
| has_code       | boolean                     |           |          | false                                    |
| has_references | boolean                     |           |          | false                                    |
| language       | character varying(10)       |           |          | 'en'::character varying                  |
| created_at     | timestamp without time zone |           |          | CURRENT_TIMESTAMP                        |
+----------------+-----------------------------+-----------+----------+------------------------------------------+
```

- **Indexes:**
    - `"paper_chunks_pkey" PRIMARY KEY, btree (id)`
    - `"idx_chunks_embedding" hnsw (embedding vector_cosine_ops) WITH (m='16', ef_construction='64')`
    - `"idx_chunks_paper_id" btree (paper_id)`
    - `"idx_chunks_text_trgm" gin (chunk_text gin_trgm_ops)`
    - `"paper_chunks_paper_id_chunk_index_key" UNIQUE CONSTRAINT, btree (paper_id, chunk_index)`
- **Foreign-key constraints:**
    - `"paper_chunks_paper_id_fkey" FOREIGN KEY (paper_id) REFERENCES papers(id) ON DELETE CASCADE`

### Installed Extensions (`\dx`)

```
+-----------+---------+-----------------+------------+-----------------------------------------------------------------+
|   Name    | Version | Default version |   Schema   |                            Description                           |
+-----------+---------+-----------------+------------+-----------------------------------------------------------------+
| btree_gin | 1.3     | 1.3             | public     | support for indexing common datatypes in GIN                     
| pg_trgm   | 1.6     | 1.6             | public     | text similarity measurement and index searching based on trigrams
| plpgsql   | 1.0     | 1.0             | pg_catalog | PL/pgSQL procedural language                                    
| vector    | 0.8.6   | 0.8.6           | public     | vector data type and ivfflat and hnsw access methods             
+-----------+---------+-----------------+------------+-----------------------------------------------------------------+
```

---

```text
papers
├── id
├── arxiv_id
├── title
├── abstract
├── authors[]
├── categories[]
├── primary_category
├── published_date
├── updated_date
├── pdf_url
├── comment
├── journal_ref
├── doi
│
├── pdf_downloaded
├── pdf_processed
└── embedding_generated
```

`arxiv_id` must be  **unique**, so we don't store same ArXiv paper twice.

---

```text
paper_chunks
├── paper_id
├── chunk_index
├── chunk_text
├── chunk_tokens
├── embedding       ← vector(384)
├── section_name
├── page_number
├── char_start
├── char_end
├── has_math
├── has_code
└── has_references
```

And PostgreSQL confirms:

```text
embedding | vector(384)
```

That's exactly what we expect from:

```text
all-MiniLM-L6-v2
        ↓
    384 numbers
        ↓
  vector(384)
```

also have the HNSW index:

```text
idx_chunks_embedding
hnsw (embedding vector_cosine_ops)
```

So PostgreSQL is ready for **vector similarity search**.

---

We now have both:

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

---

### extensions are all enabled


```text
vector     0.8.6
pg_trgm    1.6
btree_gin  1.3
plpgsql    1.0
```

Most importantly:

```text
vector
pg_trgm
btree_gin
```

are working.

---

# One architectural point to understand

You might wonder why we have both:

```text
papers.authors
papers.categories
```

and separate:

```text
authors
paper_authors
categories
```

At first this looks redundant.

The reason is that they serve different purposes.

### Simple paper search

The array is convenient:

```text
papers
authors = ["John Smith", "Jane Doe"]
categories = ["cs.AI", "cs.LG"]
```

You can quickly filter papers.

### Detailed relational information

The normalized tables allow us to store additional information:

```text
authors
├── name
├── normalized_name
├── affiliation
├── ORCID
└── email
```

And:

```text
paper_authors
├── paper_id
├── author_id
├── author_position
└── is_corresponding
```

So we can eventually answer things like:

> Show papers by this author, preserving their position in the author list.

---

```text
Target chunk size:    768 tokens
Overlap:              128 tokens
```

# HNSW already implemented


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

Instead of comparing your query against every vector one by one, HNSW builds a graph that lets PostgreSQL navigate toward nearby vectors.

---

```sql
m = 16
```

number of connections a vector can have in HNSW graph.

```text
             chunk
           /   |   \
          /    |    \
       chunk chunk chunk  # m = 3 
          \    |    /
           \   |   /
             chunk
```


---

# `ef_construction = 64`

controls how much searching PostgreSQL does while **building the HNSW index**.

---
### Paper date

```text
idx_papers_published_date
```

B-tree:

```text
published_date DESC
```

Useful for:

```text
papers after 2020
papers from 2024
latest papers
```

### Authors

```text
idx_papers_authors
```

GIN index on:

```text
authors[]
```

### Categories

```text
idx_papers_categories
```

GIN index on:

```text
categories[]
```

### Chunk text

```text
idx_chunks_text_trgm
```

GIN + trigram:

```text
chunk_text gin_trgm_ops
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
               combine results
                      │
                      ▼
                ranked results
```

---

Our `papers` table contains:

```text
pdf_downloaded
pdf_processed
embedding_generated
processing_error
```

processing pipeline.

```text
Paper arrives
     │
     ▼
pdf_downloaded = FALSE
     │
     │ download
     ▼
pdf_downloaded = TRUE
     │
     │ extract text
     ▼
pdf_processed = TRUE
     │
     │ create embeddings
     ▼
embedding_generated = TRUE
```

we can inspect a paper and know where it is in the pipeline.

```text
Paper A
────────────────────────
PDF downloaded       ✓
PDF processed        ✓
Embedding generated  ✓
```

versus:

```text
Paper B
────────────────────────
PDF downloaded       ✓
PDF processed        ✗
Embedding generated  ✗
```

Our Python program can then say:

> Paper B already has its PDF, so don't download it again. Process the PDF instead.

---

```text
                         PostgreSQL
┌───────────────────────────────────────────────────────┐
│                                                       │
│  papers                                               │
│    │                                                  │
│    ├────────── authors                                │
│    │              │                                   │
│    │        paper_authors                             │
│    │                                                  │
│    └────────── paper_chunks                           │
│                       │                               │
│                       ├── chunk_text                  │
│                       ├── chunk_tokens                │
│                       ├── section/page metadata       │
│                       │                               │
│                       └── embedding vector(384)       │
│                                  │                    │
│                              HNSW index               │
│                                                       │
│  categories                                           │
│  search_history                                       │
│  processing_queue                                     │
│                                                       │
│  pgvector ✓   pg_trgm ✓   btree_gin ✓                 │
└───────────────────────────────────────────────────────┘
```

|Component|Status|
|---|---|
|PostgreSQL|✓|
|pgvector|✓|
|`vector(384)`|✓|
|HNSW|✓|
|`pg_trgm`|✓|
|GIN metadata indexes|✓|
|Paper status flags|✓|
|Chunk fields|✓|
|768/128 chunking|**Later**|
|Chunking implementation|**Later**|
|Index monitoring|**Later**|
|ArXiv API client|**Next**|

---

