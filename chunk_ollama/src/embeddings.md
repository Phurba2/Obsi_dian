# 1. `import logging`

```
import logging
```

Imports Python's built-in logging system.

You later use:

```
logger.info(...)
```

to report what the program is doing.

For example:

```
INFO: Loading embedding model: all-MiniLM-L6-v2
INFO: Embedding dimension: 384
```

---

# 2. `from typing import List`

```
from typing import List
```

`List` is used for type hints.

You later write:

```
texts: List[str]
```

This means:

```
texts should be:

[
    "text one",
    "text two",
    "text three"
]
```

In other words:

```
List[str]
   │
   ├── List
   │
   └── strings
```

---

```
import numpy as np
```



Your embeddings are numerical arrays, so NumPy is very useful.

For example:

```
[
    0.123,
    -0.456,
    0.789
]
```

can be represented as a NumPy array:

```
numpy.ndarray
       │
       ▼
[0.123, -0.456, 0.789]
```

---



```
from sentence_transformers import SentenceTransformer
```



`SentenceTransformer` loads an embedding model.

---



```
from config.settings import (    EMBEDDING_BATCH_SIZE,    EMBEDDING_DEVICE,    EMBEDDING_MODEL,)
```



---

# 7. Create the logger

```
logger = logging.getLogger(__name__)
```

This creates a logger for this module.

`__name__` represents the current module name.

So conceptually:

```
embeddings.py
     │
     ▼
  __name__
     │
     ▼
  logger
```

Then:

```
logger.info(...)
```

can write messages.

---



```
class EmbeddingGenerator:
```







Think of it as a machine:

```
                    EmbeddingGenerator
                           │
             ┌─────────────┴─────────────┐
             │                           │
          input                        output
             │                           │
             ▼                           ▼
        "hello world"          [0.12, -0.45, ...]
```

The class knows:

- which model to use
- which device to use
- how to clean text
- how to generate embeddings
- how to generate query embeddings

---



```
_instance = None
```

This creates a class variable.

Initially:

```
_instance
   │
   ▼
 None
```

The purpose is to make `EmbeddingGenerator` behave like a **singleton**.

The idea is:

```
                    EmbeddingGenerator
                           │
                           ▼
                 ONE model instance
                           │
              ┌────────────┼────────────┐
              │            │            │
           index.py    ollama_rag.py   ask.py
              │            │            │
              └────────────┼────────────┘
                           │
                           ▼
                    same model object
```

Why?

Because loading a model can consume significant memory.

You don't want:

```
Model #1 → 400 MB
Model #2 → 400 MB
Model #3 → 400 MB
```

Instead:

```
             ONE MODEL
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
     user1    user2    user3
```

---

# 10. `__new__`

```
def __new__(cls, *args, **kwargs):
```

This is a special Python method.

Most beginners learn:

```
__init__
```

first.

But:

```
__new__
```

is responsible for **creating the object itself**.

Very simplified:

```
EmbeddingGenerator()
       │
       ▼
   __new__()
       │
       ▼
 object created
       │
       ▼
   __init__()
       │
       ▼
 object initialized
```

---

# 11. What is `cls`?

```
def __new__(cls, ...)
```

`cls` means:

> the class itself.

For this code:

```
cls
 │
 ▼
EmbeddingGenerator
```

Compare:

```
self
```

which means:

```
this object
```

while:

```
cls
```

means:

```
this class
```

---

# 12. `*args` and `**kwargs`

```
def __new__(cls, *args, **kwargs):
```

These allow the function to accept arbitrary positional and keyword arguments.

For example:

```
EmbeddingGenerator(
    "all-MiniLM-L6-v2",
    device="cpu"
)
```

could result in arguments being collected into:

```
args
 │
 └── positional arguments

kwargs
 │
 └── named arguments
```

You don't actually use them here.

They are accepted so the singleton implementation remains flexible.

---


```
if cls._instance is None:
```


---

# 14. Create the object

```
cls._instance = super(
    EmbeddingGenerator,
    cls
).__new__(cls)
```

This is the complicated-looking part.

The important concept is:

```
super(...)
```

means:

> Go to the parent implementation.

And:

```
.__new__(cls)
```

actually creates the object.

You can think of it as:

```
Need an object
     │
     ▼
Ask Python's normal object creation system
     │
     ▼
Create EmbeddingGenerator object
```

The result is stored:

```
cls._instance
```

---

# 15. `_initialized = False`

```
cls._instance._initialized = False
```

```
_initialized = False
```

> This object hasn't loaded its model yet.

---

# 16. Return the singleton

```
return cls._instance
```

Now the object is returned.

The important idea is:

```
First call:

EmbeddingGenerator()
      │
      ▼
No instance
      │
      ▼
Create instance
      │
      ▼
Store instance
      │
      ▼
Return it


Second call:

EmbeddingGenerator()
      │
      ▼
Instance already exists
      │
      ▼
Return same instance
```

---

# 17. `__init__`

```
def __init__(
    self,
    model_name: str = EMBEDDING_MODEL,
    device: str = EMBEDDING_DEVICE,
):
```

This initializes the object.

You can call:

```
EmbeddingGenerator()
```

and it uses:

```
model_name = all-MiniLM-L6-v2
device     = cpu
```

because those are the defaults from `settings.py`.


---

```
if self._initialized:
    return
```

---


```
logger.info(
    "Loading embedding model: %s",
    model_name,
)
```

This prints :

```
Loading embedding model: all-MiniLM-L6-v2
```

`%s` is replaced with:

```
model_name
```

---


---

# 22. Load the actual model

```
self.model = SentenceTransformer(
    model_name,
    device=device,
)
```

This is where the actual embedding model gets loaded.


The model is now stored in:

```
self.model
```


---

# 23. Get embedding dimension

```
self.embedding_dimension = (
    self.model.get_sentence_embedding_dimension()
)
```

This asks:

> How many numbers are in each embedding vector?

For `all-MiniLM-L6-v2`, this is typically:

```
384
```

So:

```
"PostgreSQL is a database"
             │
             ▼
       embedding model
             │
             ▼
[0.12, -0.43, 0.55, ..., 0.91]
             │
             └── 384 numbers
```

Therefore:

```
self.embedding_dimension
```

is:

```
384
```

This matters because your pgvector column must have the matching dimension.

```
Model
  │
  │ 384 dimensions
  ▼
pgvector
  │
  └── vector(384)
```

---

# 24. Log dimension

```
logger.info(
    "Embedding dimension: %d",
    self.embedding_dimension,
)
```

`%d` means integer.

Output:

```
Embedding dimension: 384
```

---

# 25. Mark initialization complete

```
self._initialized = True
```

Now:

```
_initialized
      │
      ▼
    True
```

The model is ready.

---

# 26. `generate_embeddings()`

```
def generate_embeddings(
    self,
    texts: List[str],
    show_progress: bool = True,
) -> np.ndarray:
```

This is the main function.

Input:

```
List[str]
```

meaning:

```
[
    "text one",
    "text two",
    "text three"
]
```

Output:

```
np.ndarray
```

meaning a NumPy array.

Conceptually:

```
                    texts
                      │
                      ▼
            ┌─────────────────┐
            │ generate_       │
            │ embeddings()    │
            └────────┬────────┘
                     │
                     ▼
               NumPy array
                     │
                     ▼
             vectors
```

---


```
if not texts:
```



---

# 28. Return empty array

```
return np.empty(
    (0, self.embedding_dimension),
    dtype=np.float32,
)
```

If your embedding dimension is:

```
384
```

this creates an empty array shaped:

```
(0, 384)
```

Meaning:

```
0 rows
384 columns
```

Visual:

```
        384 dimensions
<-------------------------------->

(no text)
```

This avoids trying to run the model when there is nothing to embed.

---

# 29. Clean the texts

```
cleaned_texts = [
    self._preprocess_text(text)
    for text in texts
]
```

This is a list comprehension.

If:

```
texts = [
    "  Hello   world ",
    "PostgreSQL    is great"
]
```

then each text is sent through:

```
self._preprocess_text(text)
```

Result:

```
cleaned_texts = [
    "Hello world",
    "PostgreSQL is great"
]
```

So:

```
Raw text
   │
   ▼
_preprocess_text()
   │
   ▼
Clean text
```

---

# 30. The actual embedding

```
embeddings = self.model.encode(
    cleaned_texts,
    batch_size=EMBEDDING_BATCH_SIZE,
    show_progress_bar=show_progress,
    convert_to_numpy=True,
    normalize_embeddings=True,
)
```


---


```
show_progress_bar=show_progress
```

you may see something like:

```
Batches: 100%|████████████| 32/32
```

For query embedding you later use:

```
show_progress=False
```

because you don't need a progress bar for one query.

---



```
convert_to_numpy=True
```

tells Sentence Transformers:

> Return the embeddings as NumPy arrays.

So instead of a PyTorch tensor, you get:

```
numpy.ndarray
```


---


```
normalize_embeddings=True
```

This normalizes each vector to unit length.

Conceptually:

```
Before:

[10, 20, 30, ...]

       │
       ▼
   normalize

       │
       ▼

[0.267, 0.535, 0.802, ...]
```

This is useful for similarity search.

In particular, normalized vectors make **cosine similarity** especially convenient.

```
Document
   │
   ▼
Embedding
   │
   ▼
Normalize
   │
   ▼
pgvector
   │
   │
   │ similarity search
   ▼
Relevant chunks
```

---

# 35. Return float32

```
return embeddings.astype(
    np.float32
)
```

This converts the embedding data type to:

```
float32
```

Why?

Because embeddings don't usually need 64-bit floating point precision.

`float32` uses less memory:

```
float64 → 8 bytes
float32 → 4 bytes
```

So for large numbers of vectors, this can save memory.

---

```
def _preprocess_text(self, text: str,) -> str:
```

```
raw text
   │
   ▼
clean text
   │
   ▼
embedding model
```

---

# 37. Empty text check

```
if not text:
    return ""
```

---

```
text = " ".join(
    text.split()
)
```


Suppose you have:

```
"Hello     world

this   is   RAG"
```

`text.split()` breaks it into words:

```
[
    "Hello",
    "world",
    "this",
    "is",
    "RAG"
]
```

Then:

```
" ".join(...)
```

puts them back together with exactly one space:

```
"Hello world this is RAG"
```

---

# 39. `.strip()`

```
return text.strip()
```

```
"   Hello world   "
        │
        ▼
"Hello world"
```

---

```
def generate_query_embedding(
    self,
    query: str,
) -> np.ndarray:
```


```
DOCUMENT SIDE                    QUERY SIDE
─────────────                    ───────────

Paper chunks                     User question
     │                                  │
     ▼                                  ▼
EmbeddingGenerator             EmbeddingGenerator
     │                                  │
     ▼                                  ▼
Document vector                  Query vector
     │                                  │
     └──────────────┬───────────────────┘
                    │
                    ▼
              similarity search
                    │
                    ▼
             relevant chunks
```

---

# 41. Generate the query embedding

```
embeddings = self.generate_embeddings(
    [query],
    show_progress=False,
)
```


```
[query]
```

Why brackets?

Because `generate_embeddings()` expects:

```
List[str]
```

not a single string.

So:

```
query = "Who runs the company?"
```

becomes:

```
[
    "Who runs the company?"
]
```

Then it goes through the same embedding pipeline.

---

# 42. `return embeddings[0]`

```
return embeddings[0]
```

Because you gave the function one query:

```
[
    "Who runs the company?"
]
```

the result has one embedding:

```
embeddings
   │
   ▼
[
   [0.12, -0.43, 0.55, ...]
]
```

`embeddings[0]` takes the first vector:

```
[0.12, -0.43, 0.55, ...]
```

So the final API is convenient:

```
query_vector = generator.generate_query_embedding(
    "Who runs the company?"
)
```

instead of dealing with a list containing one vector.

---


```
                 EmbeddingGenerator
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
     generate_embeddings()   generate_query_embedding()
              │                   │
              ▼                   ▼
        document text        user question
              │                   │
              ▼                   ▼
       document vectors       query vector
              │                   │
              └─────────┬─────────┘
                        │
                        ▼
                  similarity search
                        │
                        ▼
                 relevant chunks
```

---

# How this connects to your previous `EmbeddingPipeline`

Earlier you had:

```
embeddings = self.embedding_generator.generate_embeddings(
    [chunk["text"] for chunk in chunks],
    show_progress=True
)
```

Now you can see exactly what happens:

```
paper
 │
 ▼
MarkdownExtractor
 │
 ▼
TextChunker
 │
 ▼
chunks
 │
 │  ["chunk 1", "chunk 2", "chunk 3", ...]
 ▼
EmbeddingGenerator
 │
 ▼
generate_embeddings()
 │
 ▼
SentenceTransformer
 │
 ▼
all-MiniLM-L6-v2
 │
 ▼
normalized vectors
 │
 ▼
PostgreSQL / pgvector
```

Then, when the user asks:

```
"Who runs the company?"
```

the query side does:

```
"Who runs the company?"
          │
          ▼
generate_query_embedding()
          │
          ▼
all-MiniLM-L6-v2
          │
          ▼
query vector
          │
          ▼
pgvector similarity search
          │
          ▼
most relevant chunks
          │
          ▼
Ollama / llama3.2:3b
          │
          ▼
final answer
```

---

```
DOCUMENT                         QUERY
────────                         ─────

"John is the CEO..."             "Who runs the company?"
        │                                │
        ▼                                ▼
 all-MiniLM-L6-v2                 all-MiniLM-L6-v2
        │                                │
        ▼                                ▼
 document vector                   query vector
        │                                │
        └──────────────┬─────────────────┘
                       ▼
                compare vectors
                       │
                       ▼
                semantic similarity
```
