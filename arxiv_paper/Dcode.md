Good test. We found a real bug in the keyword-search SQL. **Vector search is fixed; keyword and hybrid are not yet working.**

The error:

```text
IndexError: list index out of range
```

means PostgreSQL/`psycopg2` found more `%s` placeholders in the SQL than values in our `params` list.

### What happened

Our keyword SQL contains the query **three times**:

```sql
similarity(c.chunk_text, %s)
...
c.chunk_text % %s
...
ORDER BY similarity(c.chunk_text, %s)
```

So we need three copies of `query`.

But there's an important PostgreSQL detail: `%` in this expression:

```sql
c.chunk_text % %s
```

is interpreted by psycopg2 as a formatting marker because `%` is special to Python's DB driver.

We need to escape the PostgreSQL trigram operator as:

```sql
c.chunk_text %% %s
```

---

# Step 1 — Fix `_keyword_search()`

Open:

```bash
nano src/search.py
```

Find:

```python
def _keyword_search(self, query, limit, filters):
```

Replace the **entire function** with:

```python
def _keyword_search(self, query, limit, filters):

    candidate_limit = max(limit * 10, 50)

    filter_sql, filter_params = self._build_filter_clause(filters)

    sql = f"""
        SELECT
            p.id AS paper_id,
            p.arxiv_id,
            p.title,
            p.abstract,
            p.authors,
            p.published_date,
            p.categories,

            similarity(c.chunk_text, %s) AS score,

            c.id AS chunk_id,
            c.chunk_index,
            c.chunk_text,
            c.section_name,
            c.page_number

        FROM paper_chunks c

        JOIN papers p
            ON p.id = c.paper_id

        WHERE c.chunk_text %% %s
        {filter_sql}

        ORDER BY similarity(c.chunk_text, %s) DESC

        LIMIT %s
    """

    params = [
        query,
        query,
        *filter_params,
        query,
        candidate_limit,
    ]

    with self._get_connection() as conn:
        with conn.cursor(cursor_factory=RealDictCursor) as cursor:

            cursor.execute(sql, params)

            rows = cursor.fetchall()

    return self._group_results_by_paper(rows, limit)
```

### The important change

This:

```sql
WHERE c.chunk_text % %s
```

becomes:

```sql
WHERE c.chunk_text %% %s
```

Why two `%`?

Because:

```text
Python/psycopg2
       │
       ▼
     %%  ──────►  %
```

So PostgreSQL ultimately receives:

```sql
WHERE c.chunk_text % %s
```

which is the pg_trgm similarity operator.

---

# Step 2 — Check syntax

Save with:

```text
Ctrl + O
Enter
Ctrl + X
```

Then:

```bash
python -m py_compile src/search.py
```

If it gives **no output**, the syntax is good.

---

# Step 3 — Test keyword search again

Run:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m keyword -l 5
```

This time we want something like:

```text
Searching...
======================================================================

╭───┬──────────────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                        │ ArXiv ID     │ Score  │
├───┼──────────────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks for efficient… │ 2609.24979v1 │ ...    │
│ 2 │ ...                                          │ ...          │ ...    │
╰───┴──────────────────────────────────────────────┴──────────────┴────────╯
```

If keyword search works, then test hybrid:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m hybrid -l 5
```

---

## One more thing you'll probably notice

Every command currently says:

```text
Warning: You are sending unauthenticated requests to the HF Hub...
Loading weights...
```

even for keyword search.

That's because `cli.py` currently creates:

```python
EmbeddingGenerator()
```

when the CLI starts, regardless of which command you're running.

That's **not a search failure**. It's just inefficient. A keyword search doesn't need the embedding model at all.

We'll fix that after we confirm **keyword + hybrid** work. Then we can make the CLI start much faster and remove unnecessary model loading for commands that don't need embeddings.

---

```bash
# Fixed `_keyword_search()` in `src/search.py` by escaping the PostgreSQL trigram operator:
#
# WHERE c.chunk_text %% %s
#
# The query already supplies the required three `query` parameters, so psycopg2 will correctly convert `%%` to `%` before PostgreSQL executes it.

# Syntax validation passed:

python3 -m py_compile src/search.py

# I couldn’t run the database-backed keyword or hybrid searches because that requires the project’s configured PostgreSQL environment.
```

---

That's the right fix, and the syntax check confirms `src/search.py` is valid.

Since you're already inside the project's virtual environment, let's **actually test the database-backed search now** rather than stopping at syntax validation.

### 1. Test keyword search

Run:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m keyword -l 5
```

The important thing is that you should **no longer get**:

```text
IndexError: list index out of range
```

You should get a results table.

### 2. Then test hybrid

If keyword works, run:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m hybrid -l 5
```

Hybrid combines:

```text
70% semantic/vector similarity
+
30% keyword similarity
```

from our current implementation.

### 3. If you get a database error

Don't change anything yet. Paste the **complete traceback** here.

If both commands work, we'll have verified:

```text
                 Search Engine
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Vector      Keyword      Hybrid
          │           │           │
          └───────────┼───────────┘
                      ▼
              Group by paper
                      │
                      ▼
                 CLI results
```

At that point, the core search layer is working across all three modes, and we can move on to making the CLI UI cleaner and removing the unnecessary embedding-model loading for keyword searches.


---

```bash
# Database-backed tests completed using the project virtual environment.

# ### Keyword search

# Command ran successfully with no `IndexError`:

env/bin/python scripts/cli.py search \
  -q "LoRA hypernetwork on-device personalization" \
  -m keyword -l 5

# However, it returned:

# No results found.

# So the placeholder error is fixed, but keyword search currently produces no matches for this query.

# ### Hybrid search

# Command ran successfully and returned results:

# 1. LoRA-generating hypernetworks for efficien...  2609.24979v1
# 2. onPanda: Efficient Annotation of On-Policy...  2609.24983v1
# 3. Critical-State RL: Diagnosing Trainable St...  2609.24985v1

# The interactive prompt was exited with `q`.

# The embedding-model warnings still appear for both modes, as expected from the current CLI initialization behavior.
```

---

Perfect. This tells us something useful:

### Current status

|Search mode|Status|Result|
|---|---|---|
|**Vector**|Working|Returns relevant papers|
|**Keyword**|Working technically, but no matches|Needs tuning|
|**Hybrid**|Working|Returns papers|
|**Paper grouping**|Working|One paper appears once|
|**pgvector**|Working|384-dim embeddings|
|**CLI**|Working|Interactive exploration works|

The important distinction is that **keyword search is not broken anymore**. PostgreSQL executed the query successfully; it simply decided that no chunk was similar enough to the query to pass the `pg_trgm` `%` threshold.

## Why keyword search returns nothing

Your keyword query is:

```text
LoRA hypernetwork on-device personalization
```

Your SQL currently does:

```sql
WHERE c.chunk_text %% %s
```

which PostgreSQL ultimately sees as:

```sql
WHERE c.chunk_text % %s
```

The `%` operator from `pg_trgm` means roughly:

> "Is this text similar enough to this other text?"

The default similarity threshold can be fairly strict, especially when comparing a **long query** against individual chunks.

For example, your database may contain a chunk containing:

```text
LoRA-generating hypernetworks for efficient on-device LLM
generative personalization...
```

That is clearly relevant to a human, but the trigram similarity between the entire query and the chunk may not exceed the threshold.

So:

```text
Keyword search
     │
     ▼
pg_trgm similarity
     │
     ▼
Does similarity exceed threshold?
     │
    NO
     │
     ▼
No rows
```

Whereas vector search understands the **meaning**:

```text
"LoRA hypernetwork on-device personalization"
                 │
                 ▼
          embedding model
                 │
                 ▼
       semantic representation
                 │
                 ▼
     finds conceptually similar text
```

That's exactly one of the reasons a **hybrid search system** is useful.

---

# I would make one small test before changing the search code

Let's see whether PostgreSQL's trigram search can find the actual title.

Run this directly:

```bash
env/bin/python -c "
import psycopg2
from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD

conn = psycopg2.connect(
    host=DB_HOST,
    port=DB_PORT,
    dbname=DB_NAME,
    user=DB_USER,
    password=DB_PASSWORD
)

with conn.cursor() as cur:
    cur.execute('''
        SELECT
            title,
            similarity(title, %s)
        FROM papers
        ORDER BY similarity(title, %s) DESC
    ''', (
        'LoRA hypernetwork on-device personalization',
        'LoRA hypernetwork on-device personalization',
    ))

    for row in cur.fetchall():
        print(row)

conn.close()
"
```

You should get something conceptually like:

```text
('LoRA-generating hypernetworks for efficient on-device LLM generative personalization', 0.xxxx)
('onPanda: ...', 0.xxxx)
('Critical-State RL: ...', 0.xxxx)
```

The exact numbers will depend on your database.

### Why we're doing this

This separates two things:

**1. Is pg_trgm similarity working?**

and

**2. Is the `%` threshold filtering everything out?**

If the LoRA paper gets a reasonable similarity score but keyword search still returns nothing, we know the threshold is the issue.

---

## One other thing we should fix soon

You mentioned this appears for both keyword and hybrid:

```text
Loading weights...
FutureWarning...
```

That's because your CLI currently creates:

```python
EmbeddingGenerator()
```

when the CLI starts.

So even this:

```bash
python scripts/cli.py search -m keyword ...
```

loads the embedding model even though keyword search **doesn't need embeddings at all**.

Eventually we should change the CLI to initialize the embedding model **only when vector or hybrid search is requested**.

That gives us:

```text
keyword
   │
   └── PostgreSQL/pg_trgm
       (no embedding model)

vector
   │
   └── Embedding model + pgvector

hybrid
   │
   ├── Embedding model + pgvector
   └── PostgreSQL/pg_trgm
```

But **don't change that yet**.

First run the small `similarity(title, ...)` test above. That will tell us exactly what is happening with your keyword search.