ri```bash
cd project
python3 -m venv env
source env/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r requirements.txt
```

## `.env`

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=md_vector
DB_USER=furba
DB_PASSWORD=furba
```

## Connection test

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

Expected output:

```text
Connected to PostgreSQL!
pgvector 0.8.6
```

## Enable PostgreSQL extensions

```bash
sudo -u postgres psql -d md_vector -c \
  "CREATE EXTENSION IF NOT EXISTS vector;"

sudo -u postgres psql -d md_vector -c \
  "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
```

## Initialize the database

```bash
python setup_db.py
```

## Project layout

```text
.
├── env/
├── index.py
├── ask.py
├── setup_db.py
├── schema.sql
├── requirements.txt
├── .gitignore
├── .env
├── README.md
├── config/
│   └── settings.py
├── markdown/
└── src/
    ├── __init__.py
    ├── markdown_processor.py
    ├── markdown_extractor.py
    ├── text_chunker.py
    ├── embeddings.py
    ├── embedding_pipeline.py
    ├── search.py
    └── database.py
```

## Database structure

```text
md_vector
├── papers
│   ├── id
│   ├── filename
│   ├── title
│   ├── file_path
│   ├── processed
│   └── embedding_generated
└── paper_chunks
    ├── id
    ├── paper_id
    ├── chunk_index
    ├── chunk_text
    ├── chunk_tokens
    ├── embedding vector(384)
    └── section_name
```

## Inspect the database

```bash
psql -h localhost -U furba -d md_vector
```

```sql
\dt
\d papers
\d paper_chunks
```

## Index Markdown files

Place `.md` files in the `markdown/` directory, then run:

```bash
python - <<'PY'
from index import ingest_and_embed

print(ingest_and_embed())
PY
```

## Search

```bash
python ask.py "What does the document say about avoiding financial ruin?"
```

```python
from index import search
from src.search import SearchMode

vector_results = search("your question", mode=SearchMode.VECTOR)
keyword_results = search("your words", mode=SearchMode.KEYWORD)
hybrid_results = search("your question", mode=SearchMode.HYBRID)
```

The project uses:

- PostgreSQL database: `md_vector`
- `pgvector` for semantic similarity
- `pg_trgm` for keyword similarity
- `all-MiniLM-L6-v2` embeddings
- 384-dimensional vectors
- HNSW vector indexing
- Hybrid search with `0.7 * vector + 0.3 * keyword` scoring