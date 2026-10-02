```
                .env
                 │
                 │ load settings
                 ▼
        ┌──────────────────┐
        │   config.py      │
        │                  │
        │ DB settings      │
        │ Embedding model  │
        │ Batch size       │
        │ Device           │
        │ Markdown path    │
        └────────┬─────────┘
                 │
                 ▼
        Rest of your RAG app
```

Instead of writing passwords, database names, model names, etc. directly throughout your code, you keep them here.

---

# 1. `import os`

```
import os
```

`os` is a **built-in Python module**.

It lets Python interact with the operating system.

One thing we need from it is:

```
os.getenv(...)
```

which reads an **environment variable**.

For example, if your environment contains:

```
DB_HOST=localhost
```

Python can read it:

```
os.getenv("DB_HOST")
```

and get:

```
localhost
```

Think:

```
Operating System
       │
       │ environment variables
       ▼
    DB_HOST
       │
       ▼
 Python → os.getenv()
```

---

# 2. `from pathlib import Path`

```
from pathlib import Path
```

`pathlib` is Python's standard library for working with **file and directory paths**.

You're importing:

```
Path
```

so you can write:

```
Path("something")
```

instead of manually manipulating strings like:

```
"/home/furba/project/something"
```

For example:

```
Path("markdown")
```

represents a path named:

```
markdown/
```

`Path` is especially useful because it works cleanly across operating systems.

---

# 3. `from dotenv import load_dotenv`

```
from dotenv import load_dotenv
```

This comes from the external Python package:

```
python-dotenv
```

It provides:

```
load_dotenv()
```

Its job is to read a `.env` file and load its values as environment variables.

For example, your `.env` might contain:

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pdf_vector
DB_USER=furba
DB_PASSWORD=my_secret_password

EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_BATCH_SIZE=32
EMBEDDING_DEVICE=cpu

MARKDOWN_STORAGE_PATH=markdown
```

Then:

```
load_dotenv(...)
```

makes those values available through:

```
os.getenv(...)
```

---

# 4. Finding the project directory

```
BASE_DIR = Path(__file__).resolve().parent.parent
```

This looks complicated, but it is actually just walking through directories.

Let's break it apart.

---

## `__file__`

```
__file__
```

is a special Python variable.

It represents the path of the current Python file.

For example, suppose your project is:

```
/home/furba/pdf_rag/
│
├── .env
├── config/
│   └── settings.py    ← current file
│
├── markdown/
└── ...
```

Then:

```
__file__
```

might represent:

```
/home/furba/pdf_rag/config/settings.py
```

---

## `.resolve()`

```
Path(__file__).resolve()
```

converts the path into an absolute, normalized path.

For example:

```
config/settings.py
```

becomes:

```
/home/furba/pdf_rag/config/settings.py
```

---

## `.parent`

```
Path(__file__).resolve().parent
```

means:

> Give me the directory containing this file.

So:

```
/home/furba/pdf_rag/config/settings.py
                                      │
                                      ▼
                              .parent
                                      │
                                      ▼
                         /home/furba/pdf_rag/config
```

---

## `.parent.parent`

Then:

```
Path(__file__).resolve().parent.parent
```

goes up **two directories**:

```
/home/furba/pdf_rag/config/settings.py
                  │
                  │ .parent
                  ▼
       /home/furba/pdf_rag/config
                  │
                  │ .parent
                  ▼
          /home/furba/pdf_rag
```

So:

```
BASE_DIR
```

becomes:

```
/home/furba/pdf_rag
```

### Visual

```
/home/furba/pdf_rag/
│
├── .env                 ← BASE_DIR / ".env"
│
├── config/
│   └── settings.py      ← __file__
│
├── markdown/            ← BASE_DIR / "markdown"
│
└── ...
```

That's why this variable is called:

```
BASE_DIR
```

It represents the **base/root directory of your project**.

---

# 5. Load `.env`

```
load_dotenv(BASE_DIR / ".env")
```

This tells `python-dotenv`:

> Find the `.env` file inside my project directory and load it.

The `/` here is **not division**.

Because `BASE_DIR` is a `Path`, Python's `Path` object overloads `/` to join paths.

For example:

```
BASE_DIR / ".env"
```

becomes:

```
/home/furba/pdf_rag/.env
```

So:

```
load_dotenv(BASE_DIR / ".env")
```

means:

```
             BASE_DIR
                │
                ▼
       /home/furba/pdf_rag
                │
                │ + ".env"
                ▼
       /home/furba/pdf_rag/.env
                │
                ▼
         load environment
            variables
```

---

# 6. Database host

```
DB_HOST = os.getenv("DB_HOST", "localhost")
```

This reads the environment variable:

```
DB_HOST
```

The second argument:

```
"localhost"
```

is the **default value**.

The pattern is:

```
os.getenv(NAME, DEFAULT)
```

Meaning:

```
Does DB_HOST exist?
       │
    ┌──┴──┐
   YES    NO
    │      │
    ▼      ▼
 use it  "localhost"
```

So if `.env` contains:

```
DB_HOST=192.168.1.20
```

then:

```
DB_HOST
```

becomes:

```
192.168.1.20
```

If nothing is configured:

```
DB_HOST
```

becomes:

```
localhost
```

### What is `localhost`?

`localhost` means:

> This same computer.

So your application would connect to PostgreSQL running on your own PC.

---

# 7. Database port

```
DB_PORT = int(os.getenv("DB_PORT", "5432"))
```

This is slightly different.

PostgreSQL normally uses:

```
5432
```

as its port.

But there is an important detail.

`os.getenv()` returns a **string**.

For example:

```
os.getenv("DB_PORT", "5432")
```

returns:

```
"5432"
```

Notice the quotes.

But your database library probably wants:

```
5432
```

as an integer.

So you use:

```
int(...)
```

to convert:

```
"5432"
  │
  │ int()
  ▼
 5432
```

Therefore:

```
DB_PORT = int(os.getenv("DB_PORT", "5432"))
```

means:

> Read `DB_PORT`; if it doesn't exist, use `"5432"`; then convert it into an integer.

---

# 8. Database name

```
DB_NAME = os.getenv("DB_NAME", "pdf_vector")
```

This reads:

```
DB_NAME
```

from `.env`.

If it doesn't exist, use:

```
pdf_vector
```

So your PostgreSQL database might be:

```
PostgreSQL
    │
    └── pdf_vector
```

---

# 9. Database user

```
DB_USER = os.getenv("DB_USER", "furba")
```

This determines which PostgreSQL user/role your Python application uses.

Default:

```
furba
```

So the connection might look conceptually like:

```
Python RAG
    │
    │ user = furba
    ▼
PostgreSQL
    │
    └── database = pdf_vector
```

---

# 10. Database password

```
DB_PASSWORD = os.getenv("DB_PASSWORD")
```

This is slightly different because there is **no default**.

You are saying:

> Get the `DB_PASSWORD` environment variable.

If it doesn't exist:

```
DB_PASSWORD
```

will be:

```
None
```

This is important because you generally **don't want to hard-code your password**:

```
# BADDB_PASSWORD = "mypassword123"
```

Instead:

```
.env
  │
  └── DB_PASSWORD=********
             │
             ▼
       os.getenv()
             │
             ▼
       DB_PASSWORD
```

---

# 11. Embedding model

```
EMBEDDING_MODEL = os.getenv(    "EMBEDDING_MODEL",    "all-MiniLM-L6-v2")
```

Your actual code has it on one line:

```
EMBEDDING_MODEL = os.getenv("EMBEDDING_MODEL", "all-MiniLM-L6-v2")
```

This defines which model creates your **embeddings**.

---

# 12. Embedding batch size

```
EMBEDDING_BATCH_SIZE = int(os.getenv("EMBEDDING_BATCH_SIZE", "32"))
```

This controls how many pieces of text you process at once.

Suppose you have:

```
1,000 documents
```

Instead of:

```
document 1
document 2
document 3
...
```

one at a time, you can process them in groups.

With:

```
batch size = 32
```

you get:

```
┌─────────────────────────────┐
│ batch 1                     │
│ 1 ... 32                    │
└─────────────────────────────┘

┌─────────────────────────────┐
│ batch 2                     │
│ 33 ... 64                   │
└─────────────────────────────┘

┌─────────────────────────────┐
│ batch 3                     │
│ 65 ... 96                   │
└─────────────────────────────┘

              ...
```

---

# 13. Embedding device

```
EMBEDDING_DEVICE = os.getenv("EMBEDDING_DEVICE", "cpu")
```

This determines where the embedding model runs.

---

# 14. Markdown storage path

```
MARKDOWN_STORAGE_PATH = BASE_DIR / os.getenv("MARKDOWN_STORAGE_PATH", "markdown")
```

This tells your application where Markdown files are stored.

Suppose:

```
MARKDOWN_STORAGE_PATH=markdown
```

Then:

```
os.getenv(...)
```

returns:

```
markdown
```

And:

```
BASE_DIR / "markdown"
```

might become:

```
/home/furba/pdf_rag/markdown
```

So:

```
MARKDOWN_STORAGE_PATH
```

is a `Path` object pointing to:

```
project/
└── markdown/
```

---

# Why use `BASE_DIR / ...`?

You could technically write:

```
MARKDOWN_STORAGE_PATH = "markdown"
```

But then the meaning depends on the directory from which you start the program.

For example:

```
cd /home/furba/pdf_rag
python main.py
```

might work.

But:

```
cd /tmp
python /home/furba/pdf_rag/main.py
```

could cause relative paths to behave differently.

Using:

```
BASE_DIR / "markdown"
```

makes the path anchored to your project:

```
              project root
                   │
                   ▼
        /home/furba/pdf_rag
                   │
                   └── markdown
```

Much safer.

---

# Complete configuration architecture

Your file is essentially doing this:

```
                    .env
                     │
                     │
                     ▼
            ┌─────────────────┐
            │    config.py    │
            └────────┬────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   PostgreSQL     Embeddings    Files
        │            │            │
        │            │            │
        ▼            ▼            ▼
     DB_HOST      MODEL        MARKDOWN
     DB_PORT      BATCH        PATH
     DB_NAME      DEVICE
     DB_USER
     DB_PASSWORD
```

---

# The `.env` → Python relationship

You can think of `.env` as the **settings storage**:

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pdf_vector
DB_USER=furba
DB_PASSWORD=secret

EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_BATCH_SIZE=32
EMBEDDING_DEVICE=cpu

MARKDOWN_STORAGE_PATH=markdown
```

And `config.py` as the **settings loader**:

```
              .env
               │
               │ load_dotenv()
               ▼
        Environment variables
               │
               │ os.getenv()
               ▼
          ┌────────────┐
          │ config.py  │
          └─────┬──────┘
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
       DB     Model     Files
```

Then another file can simply do:

```
from config import DB_HOST, DB_NAME, EMBEDDING_MODEL
```

instead of repeatedly reading `.env`.

---

**This `config.py` file centralizes your PostgreSQL, embedding-model, and file-storage settings, loading them from `.env` while providing sensible defaults.**