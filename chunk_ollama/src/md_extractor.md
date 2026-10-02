```
                 Markdown file
                      │
                      ▼
              ┌─────────────────┐
              │MarkdownExtractor│
              └────────┬────────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Full text             Headings
             │                   │
             │              ┌────┴────┐
             │              │ Section │
             │              │positions│
             │              └────┬────┘
             │                   │
             └──────────┬────────┘
                        ▼
              {
                "text": ...,
                "sections": [...]
              }
```

---

# 1. Import `Path`

```
from pathlib import Path
```

### `from`

Python keyword meaning:

> Get something from a module/package.

### `pathlib`

Python's built-in library for working with filesystem paths.

For example:

```
~/chunk_Ollama/markdown/paper.md
```

Instead of manually manipulating strings representing paths, Python gives us `Path`.

### `import`

Bring something into this Python file so we can use it.

### `Path`

This is a class from `pathlib`.

So:

```
from pathlib import Path
```

means:

```
pathlib
   │
   └── Path
        │
        └── now available in this file
```

We later use:

```
Path(path)
```

to turn a normal string path into a `Path` object.

---

# 2. Import `Dict` and `List`

```
from typing import Dict, List
```

These come from Python's `typing` module.

They are used for **type hints**.

---

## `List`

`List` means:

> This variable/function contains a list.

For example:

```
List[str]
```

means:

```
List
 ├── "hello"
 ├── "world"
 └── "python"
```

where every item is expected to be a string.

---

## `Dict`

`Dict` means:

> This variable/function contains a dictionary.

For example:

```
Dict[str, object]
```

roughly means:

```
dictionary
│
├── string key
│       │
│       └── object value
│
├── string key
│       │
│       └── object value
│
└── ...
```

---



```
class MarkdownExtractor:
```


```
┌─────────────────────────────┐
│     MarkdownExtractor       │
│                             │
│  extract()                  │
│      │                      │
│      ├── read Markdown      │
│      ├── find headings      │
│      ├── find sections      │
│      └── return information │
└─────────────────────────────┘
```

Later you can create an object:

```
extractor = MarkdownExtractor()
```

and then:

```
extractor.extract("paper.md")
```

---



---

# 5. Define the `extract()` method

```
def extract(self, path: str) -> Dict[str, object]:
```


---




---
---

## `self`

```
def extract(self, ...)
```

---

## `path`

```
path: str
```

`path` is the input argument.

It should contain the path to the Markdown file.

For example:

```
"markdown/company.md"
```

or:

```
"/home/furba/chunk_Ollama/markdown/company.md"
```

---

## `: str`

```
path: str
```

This is a **type hint**.

It tells us:

```
path
  ↓
should be a string
```

Example:

```
path = "markdown/test.md"
```

---

## `-> Dict[str, object]`

This is another type hint.

It says the function returns a dictionary.

Something like:

```
{
    "text": "...",
    "sections": [...]
}
```

More specifically:

```
Dict
 │
 ├── key: str
 │
 └── value: object
```

So:

```
Dict[str, object]
```

means:

> A dictionary whose keys are strings and whose values can be objects of various types.

---

```
text = Path(path).read_text(encoding="utf-8")
```

---

## `Path(path)`

Suppose:

```
path = "markdown/company.md"
```

Then:

```
Path(path)
```

creates a Path object:

```
"markdown/company.md"
          │
          ▼
     Path object
```

---

## `.read_text()`

`Path` has a method called:

```
read_text()
```

which reads a text file.

So:

```
Path(path).read_text()
```

means:

```
Find file
   │
   ▼
Open file
   │
   ▼
Read contents
   │
   ▼
Return text
```

---

## `encoding="utf-8"`

This tells Python how the file is encoded.

UTF-8 supports normal English characters as well as many other characters.

For example:

```
A B C
नेपाल
é
中
```

So:

```
encoding="utf-8"
```

means:

> Read this text file as UTF-8.

---

## Assignment to `text`

```
text = ...
```

The complete file contents are stored in:

```
text
```

For example:

```
# Introduction

This is a company.

## History

The company started in 2020.

## Products

We sell cameras.
```

becomes:

```
text = """
# Introduction

This is a company.

## History

The company started in 2020.

## Products

We sell cameras.
"""
```

---

# 7. Create an empty sections list

```
sections: List[Dict] = []
```

This creates an empty list.

```
sections
   │
   ▼
┌──────────────┐
│              │
│   EMPTY      │
│     []       │
│              │
└──────────────┘
```

Later we'll put dictionaries into this list.

For example:

```
sections = [
    {
        "name": "Introduction",
        "start": 0,
        "end": 50
    },
    {
        "name": "History",
        "start": 50,
        "end": 100
    }
]
```

---

## Why `List[Dict]`?

It tells us:

```
sections
   │
   ▼
List
 │
 ├── Dict
 ├── Dict
 ├── Dict
 └── ...
```

So:

```
List[Dict]
```

means:

> A list containing dictionaries.

---

# 8. Default heading

```
heading = "Document"
```

At the beginning, we don't know whether the document has a heading.

So we start with:

```
heading = "Document"
```

This acts as a default name.

Imagine a Markdown file starts like this:

```
This is some text before the first heading.

More text.

# Introduction
...
```

The text before `# Introduction` still needs a section name.

Therefore:

```
"No heading yet"
       │
       ▼
"Document"
```

---

# 9. Track where the current section starts

```
section_start = 0
```

This stores the character position where the current section begins.

Python strings have character indexes.

For example:

```
Hello world
01234567890
```

The first character is position:

```
0
```

So initially:

```
section_start = 0
```

means:

> The current section starts at the beginning of the document.

---

# 10. Track our current position

```
offset = 0
```

`offset` means:

> Where are we currently located in the document?

Initially we're at:

```
START
  │
  ▼
position 0
```

As we read lines, `offset` increases.

---

# 11. Loop through every line

```
for line in text.splitlines(keepends=True):
```

This is very important.

It means:

> Take the document and process it one line at a time.

---

## `text.splitlines()`

Suppose:

```
text = """# Introduction
Hello world.
## History
Started in 2020.
"""
```

Then:

```
text.splitlines()
```

produces roughly:

```
[
    "# Introduction",
    "Hello world.",
    "## History",
    "Started in 2020."
]
```

---

## But here we use:

```
splitlines(keepends=True)
```

The `keepends=True` is important.

It keeps the newline characters.

Without it:

```
"# Introduction"
"Hello world."
"## History"
```

With it:

```
"# Introduction\n"
"Hello world.\n"
"## History\n"
```

Why?

Because the program wants to keep accurate character positions.

---

# 12. Check whether the line is a heading

```
if line.lstrip().startswith("#"):
```

This asks:

> Does this line represent a Markdown heading?

Let's break it apart.

---

## `line`

Current line.

For example:

```
"## History\n"
```

---

## `.lstrip()`

`lstrip()` removes whitespace from the **left side**.

Example:

```
"   ## History".lstrip()
```

becomes:

```
"## History"
```

So this extractor also recognizes:

```
   # Introduction
```

as a heading.

---

## `.startswith("#")`

Checks whether the string begins with `#`.

Example:

```
"# Introduction".startswith("#")
```

returns:

```
True
```

while:

```
"Hello".startswith("#")
```

returns:

```
False
```

So:

```
if line.lstrip().startswith("#"):
```

means:

```
                current line
                      │
                      ▼
               remove left spaces
                      │
                      ▼
                starts with "#"
                 /          \
               YES           NO
                │             │
                ▼             ▼
          heading line     normal line
```

---

# 13. Check whether previous section has content

```
if offset > section_start:
```

This asks:

> Has the current section actually accumulated any characters?

For example:

```
section_start = 0
offset = 0
```

Then:

```
offset > section_start
```

is:

```
0 > 0
```

which is:

```
False
```

But later:

```
section_start = 0
offset = 50
```

Then:

```
50 > 0
```

is:

```
True
```

Therefore there is something to save.

---

# 14. Save the previous section

```
sections.append({
    "name": heading,
    "start": section_start,
    "end": offset
})
```

This adds a dictionary to `sections`.

Suppose:

```
heading = "Introduction"
section_start = 0
offset = 50
```

Then:

```
{    "name": "Introduction",    "start": 0,    "end": 50}
```

gets added.

Visually:

```
sections
   │
   ▼
┌─────────────────────────────────┐
│ {                               │
│   name:  "Introduction",        │
│   start: 0,                     │
│   end:   50                     │
│ }                               │
└─────────────────────────────────┘
```

---

# 15. What does `append()` do?

```
sections.append(...)
```

`append()` means:

> Add one item to the end of a list.

Example:

```
numbers = []

numbers.append(10)
numbers.append(20)
```

Now:

```
numbers
```

is:

```
[10, 20]
```

So here:

```
sections.append({...})
```

adds another section.

---

# 16. Extract the heading name

```
heading = line.lstrip().lstrip("#").strip() or "Document"
```

This line looks complicated, but it is basically a sequence of cleaning operations.

Let's use:

```
## History
```

---

## Step 1

```
line.lstrip()
```

removes spaces on the left.

```
"   ## History"
        ↓
"## History"
```

---

## Step 2

```
.lstrip("#")
```

removes `#` characters from the left.

```
"## History"
      ↓
" History"
```

---

## Step 3

```
.strip()
```

removes whitespace from both sides.

```
" History"
     ↓
"History"
```

Therefore:

```
heading = "History"
```

---

# 17. Why `or "Document"`?

The complete expression is:

```
line.lstrip().lstrip("#").strip() or "Document"
```

Suppose the heading is:

```
###
```

After removing `#`:

```
""
```

That's an empty string.

In Python, an empty string is **falsey**.

Therefore:

```
"" or "Document"
```

produces:

```
"Document"
```

So this protects against empty headings.

```
Heading text
     │
     ▼
clean it
     │
     ▼
empty?
 ┌───┴───┐
YES      NO
 │        │
 ▼        ▼
Document  actual heading
```

---

# 18. Set the beginning of the new section

```
section_start = offset
```

This is very important.

We've just found a new heading.

Therefore the new section starts at the current position.

For example:

```
Previous section
0 ------------------ 50
                    ▲
                    │
                  offset

# History
```

Now:

```
section_start = offset
```

means:

```
section_start = 50
```

So:

```
History section
50 ---------------------- ...
▲
│
section_start
```

---

# 19. Move the offset forward

```
offset += len(line)
```

This means:

```
offset = offset + len(line)
```

---

## `len(line)`

`len()` tells us how many characters are in the line.

For example:

```
len("Hello\n")
```

is:

```
6
```

because:

```
H e l l o \n
0 1 2 3 4 5
```

---

## `+=`

This:

```
offset += len(line)
```

is shorthand for:

```
offset = offset + len(line)
```

So if:

```
offset = 50
len(line) = 10
```

then:

```
offset = 60
```

---

# 20. Why is `offset` necessary?

This is probably the most important concept in this entire file.

Imagine this Markdown:

```
# Introduction

This is the introduction.

## History

The company started in 2020.

## Products

We sell cameras.
```

The program tracks:

```
character position
      │
      ▼

0
│
├── # Introduction
│
├── This is the introduction.
│
├── ## History
│
├── The company started in 2020.
│
└── ## Products
    │
    └── We sell cameras.
```

It wants to eventually produce something like:

```
[
    {
        "name": "Introduction",
        "start": 0,
        "end": 42
    },
    {
        "name": "History",
        "start": 42,
        "end": 85
    },
    {
        "name": "Products",
        "start": 85,
        "end": 110
    }
]
```

The exact numbers depend on the actual text.

---

# 21. Finish the final section

After the `for` loop:

```
if section_start < len(text):
```

This asks:

> Is there still some text after the beginning of the final section?

For example:

```
section_start = 80
len(text) = 120
```

Then:

```
80 < 120
```

is:

```
True
```

So there is a final section that needs to be added.

---

# 22. Add the final section

```
sections.append({
    "name": heading,
    "start": section_start,
    "end": len(text)
})
```

Notice something important.

Earlier we used:

```
"end": offset
```

But now we use:

```
"end": len(text)
```

Why?

Because we're at the **end of the document**.

There isn't another heading to tell us where the section ends.

Therefore:

```
Final section
     │
     ▼
section_start ─────────────────── len(text)
                                  ▲
                                  │
                             end of file
```

---

# 23. Return everything

```
return {"text": text, "sections": sections}
```

The function returns a dictionary containing two things:

```
┌──────────────────────────────────────────┐
│              RESULT                      │
├──────────────────────────────────────────┤
│ "text"     → entire Markdown document    │
│                                            │
│ "sections" → heading/position information│
└──────────────────────────────────────────┘
```

For example:

```
{
    "text": "# Introduction\nHello...",
    
    "sections": [
        {
            "name": "Introduction",
            "start": 0,
            "end": 42
        },
        {
            "name": "History",
            "start": 42,
            "end": 85
        }
    ]
}
```

---

# Complete Flow

Let's put the whole class together:

```
                 Markdown file
                       │
                       ▼
             ┌──────────────────┐
             │ Path(path)       │
             │ .read_text()     │
             └────────┬─────────┘
                      │
                      ▼
                 entire text
                      │
                      ▼
             split into lines
                      │
                      ▼
          ┌───────────────────────┐
          │       for line        │
          └───────────┬───────────┘
                      │
                      ▼
             Is it a heading?
                 /         \
               YES          NO
                │            │
                ▼            │
       save previous         │
          section            │
                │            │
                ▼            │
       extract heading       │
                │            │
                ▼            │
       start new section     │
                │            │
                └─────┬──────┘
                      │
                      ▼
               update offset
                      │
                      ▼
                next line
                      │
                      ▼
                 end of loop
                      │
                      ▼
             save final section
                      │
                      ▼
                   return
                      │
                      ▼
        ┌──────────────────────────┐
        │ text                     │
        │ sections                 │
        └──────────────────────────┘
```

---

# Example With Your RAG System

This is where this code becomes useful.

Suppose your Markdown file contains:

```
# Company

Our company is Elite.

## History

Elite started in Nepal.

## Services

We provide photography and videography.

## Contact

Contact us at example.com.
```

The extractor gives your pipeline a map:

```
Markdown
   │
   ├── # Company
   │
   ├── ## History
   │
   ├── ## Services
   │
   └── ## Contact
```

and something conceptually like:

```
{
    "text": "...entire document...",
    
    "sections": [
        {
            "name": "Company",
            "start": 0,
            "end": ...
        },
        {
            "name": "History",
            "start": ...,
            "end": ...
        },
        {
            "name": "Services",
            "start": ...,
            "end": ...
        },
        {
            "name": "Contact",
            "start": ...,
            "end": ...
        }
    ]
}
```

Then your next component, `TextChunker`, can use this information.

---

# How It Fits Into Your RAG Pipeline

Your project is essentially doing:

```
                MARKDOWN FILE
                     │
                     ▼
          ┌─────────────────────┐
          │ MarkdownExtractor   │
          │                     │
          │ Find headings       │
          │ Find boundaries     │
          └──────────┬──────────┘
                     │
                     ▼
              text + sections
                     │
                     ▼
          ┌─────────────────────┐
          │    TextChunker      │
          │                     │
          │ Split into chunks   │
          └──────────┬──────────┘
                     │
                     ▼
                  chunks
                     │
                     ▼
          ┌─────────────────────┐
          │ EmbeddingGenerator  │
          │                     │
          │ text → vector       │
          └──────────┬──────────┘
                     │
                     ▼
             384-dimensional
                 vectors
                     │
                     ▼
          ┌─────────────────────┐
          │ PostgreSQL pgvector │
          └──────────┬──────────┘
                     │
                     ▼
               similarity
                  search
                     │
                     ▼
              relevant chunks
                     │
                     ▼
          ┌─────────────────────┐
          │ Ollama              │
          │ llama3.2:3b         │
          └──────────┬──────────┘
                     │
                     ▼
                   ANSWER
```

## The key idea

`MarkdownExtractor` **does not create embeddings**.

It doesn't search.

It doesn't use Ollama.

It doesn't answer questions.

Its job is much simpler:

```
MarkdownExtractor
       │
       ├── Read the document
       ├── Find headings
       ├── Track character positions
       └── Tell TextChunker where sections are
```

So you can think of it as the **document mapper** at the beginning of your RAG pipeline:

```
              DOCUMENT
                  │
                  ▼
        ┌─────────────────┐
        │ "Where are the  │
        │  sections?"     │
        └────────┬────────┘
                 │
                 ▼
        MarkdownExtractor
                 │
                 ▼
       ┌───────────────────┐
       │ Introduction  0-50│
       │ History      50-90│
       │ Services    90-140│
       │ Contact    140-180│
       └───────────────────┘
                 │
                 ▼
             TextChunker
```

That separation is a good design: **one component reads/maps the document, another decides how to chunk it, and another converts chunks into vectors.**