# PDF semantic search - build notes

The original tutorial has been edited to match the files that actually exist in this project.

Main corrections:

- **No ArXiv client and no PDF downloader.** PDFs are supplied by you in `pdf/`; `src/pdf_processor.py` is a `PDFStorage` class that lists and validates them.
- **Database is `pdf_vector`** (not `ragdb`), storage folder is `pdf/` (not `data/pdfs/YYYY/`).
- **No `scripts/` folder.** Entry points live in `index.py`.
- **`papers` is keyed by `filename`** - no `arxiv_id`, no authors tables, no `processing_queue`.
- **Paper-level result grouping is already implemented** in `src/search.py` (`_group()`).

## Project layout

```text
.
├── index.py              # entry points: ingest_and_embed(), search()
├── setup_db.py           # applies schema.sql
├── schema.sql
├── requirements.txt
├── config/
│   └── settings.py       # DB + model settings, .env overrides
├── pdf/                  # put your PDFs here
└── src/
    ├── __init__.py
    ├── pdf_processor.py        # PDFStorage: discover + validate PDFs
    ├── paper_processor.py      # register PDFs in the papers table
    ├── pdf_extractor.py        # PyMuPDF text extraction
    ├── text_chunker.py         # sentence-aware chunking
    ├── embeddings.py           # EmbeddingGenerator (all-MiniLM-L6-v2)
    ├── embedding_pipeline.py   # chunks -> embeddings -> paper_chunks
    ├── search.py               # vector / keyword / hybrid search
    └── database.py             # empty placeholder
```

## Setup

```bash
pip install -r requirements.txt
python setup_db.py
```

Drop one or more PDFs into `pdf/`. The full chain is:

```text
pdf/*.pdf
   |  PDFStorage (list + validate)
   v
PaperProcessor.ingest_pdfs()
   |
   v
papers table (filename, title, pdf_path)
   |  PDFExtractor (PyMuPDF)
   v
clean structured text
   |  TextChunker
   v
chunks
   |  EmbeddingGenerator (all-MiniLM-L6-v2, 384 dims)
   v
paper_chunks (PostgreSQL + pgvector, HNSW index)
   |
   v
vector / keyword / hybrid search
```

---

## 1. `src/pdf_processor.py`

```bash
nano src/pdf_processor.py
```

```python
import logging
from pathlib import Path
from typing import Dict

from config.settings import PDF_STORAGE_PATH

logger = logging.getLogger(__name__)


class PDFStorage:
    """Discover and validate PDFs supplied by the user."""

    def __init__(self, storage_path: str = str(PDF_STORAGE_PATH)):
        self.storage_path = Path(storage_path)
        self.storage_path.mkdir(parents=True, exist_ok=True)

    def list_pdfs(self):
        return sorted(self.storage_path.rglob("*.pdf"))

    @staticmethod
    def validate_pdf(pdf_path: Path) -> bool:
        try:
            return (
                pdf_path.is_file()
                and pdf_path.stat().st_size >= 100
                and pdf_path.open("rb").read(5) == b"%PDF-"
            )
        except OSError:
            return False

    def get_storage_stats(self) -> Dict[str, object]:
        files = [path for path in self.list_pdfs() if self.validate_pdf(path)]
        total_bytes = sum(path.stat().st_size for path in files)
        return {
            "total_files": len(files),
            "total_bytes": total_bytes,
            "total_mb": round(total_bytes / (1024 * 1024), 2),
        }


# Backwards-compatible name for callers that only use storage statistics.
PDFDownloader = PDFStorage
```

Verify storage statistics:

```bash
python - <<'PY'
from src.pdf_processor import PDFStorage

stats = PDFStorage().get_storage_stats()

print("PDF storage statistics:")
print(f"  Files: {stats['total_files']}")
print(f"  Size:  {stats['total_mb']} MB")
PY
```

With an empty `pdf/` folder:

```text
PDF storage statistics:
  Files: 0
  Size:  0.0 MB
```

---

## 2. `src/paper_processor.py`

Registers every valid PDF found in `pdf/` as a row in `papers`.

```bash
nano src/paper_processor.py
```

```python
import logging
from pathlib import Path
from typing import Optional

import psycopg2
from psycopg2.extras import RealDictCursor

from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
from src.pdf_processor import PDFStorage

logger = logging.getLogger(__name__)


class PaperProcessor:
    """Register PDFs already present in the configured PDF folder."""

    def __init__(self, db_config: dict, pdf_storage: Optional[PDFStorage] = None):
        self.db_config = db_config
        self.pdf_storage = pdf_storage or PDFStorage()

    def _get_connection(self):
        return psycopg2.connect(**self.db_config)

    def ingest_pdfs(self) -> dict:
        pdfs = [p for p in self.pdf_storage.list_pdfs() if self.pdf_storage.validate_pdf(p)]
        conn = self._get_connection()
        new = 0
        existing = 0
        try:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                for pdf_path in pdfs:
                    filename = pdf_path.name
                    title = pdf_path.stem.replace("_", " ")
                    cursor.execute(
                        """
                        INSERT INTO papers (filename, title, pdf_path)
                        VALUES (%s, %s, %s)
                        ON CONFLICT (filename) DO UPDATE SET
                            title = EXCLUDED.title,
                            pdf_path = EXCLUDED.pdf_path,
                            updated_at = CURRENT_TIMESTAMP
                        RETURNING (xmax = 0) AS inserted
                        """,
                        (filename, title, str(pdf_path)),
                    )
                    if cursor.fetchone()["inserted"]:
                        new += 1
                    else:
                        existing += 1
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()
        return {"found": len(pdfs), "new": new, "existing": existing, "failed": 0}


def create_default_processor() -> PaperProcessor:
    return PaperProcessor(
        db_config={
            "host": DB_HOST,
            "port": DB_PORT,
            "dbname": DB_NAME,
            "user": DB_USER,
            "password": DB_PASSWORD,
        }
    )
```

Register the PDFs:

```bash
python - <<'PY'
from src.paper_processor import create_default_processor

print(create_default_processor().ingest_pdfs())
PY
```

```text
{'found': 2, 'new': 2, 'existing': 0, 'failed': 0}
```

Note: `title` is derived from the filename (`stem.replace("_", " ")`), so name your PDFs sensibly if you want readable titles.

The chain so far:

```text
pdf/*.pdf
   |
   v
PDFStorage (list + validate)
   |
   v
PaperProcessor.ingest_pdfs()
   |
   v
papers table (filename, title, pdf_path)
```

Next: extract clean structured text from each PDF.

---

## 3. `src/pdf_extractor.py`

Install the extraction dependency first:

```bash
pip install nltk
```

```bash
nano src/pdf_extractor.py
```

```python
import logging
import re
from dataclasses import dataclass
from pathlib import Path
from typing import Dict, List

import fitz


logger = logging.getLogger(__name__)


@dataclass
class ExtractedPage:
    """Structured representation of extracted page content."""

    page_num: int
    text: str
    blocks: List[Dict]
    has_columns: bool
    has_math: bool
    has_tables: bool
    confidence: float


class PDFExtractor:
    """Advanced PDF text extraction for academic papers."""

    def __init__(self):
        pass

    def extract_paper_text(
        self,
        pdf_path: str,
    ) -> Dict[str, object]:
        """Extract complete text from a PDF."""

        pdf_path = Path(pdf_path)

        if not pdf_path.exists():
            raise FileNotFoundError(
                f"PDF not found: {pdf_path}"
            )

        pages: List[ExtractedPage] = []

        document = fitz.open(pdf_path)

        try:
            for page_index, page in enumerate(document):
                page_num = page_index + 1

                extracted_page = self._extract_page(
                    page,
                    page_num,
                )

                pages.append(extracted_page)

        finally:
            document.close()

        full_text = "\n\n".join(
            page.text
            for page in pages
            if page.text.strip()
        )

        full_text = self._clean_extracted_text(
            full_text
        )

        sections = self._identify_sections(
            full_text
        )

        confidence = self._calculate_confidence(
            pages
        )

        return {
            "text": full_text,
            "pages": pages,
            "sections": sections,
            "page_count": len(pages),
            "confidence": confidence,
        }

    def _extract_page(
        self,
        page,
        page_num: int,
    ) -> ExtractedPage:
        """Extract text from one page with layout analysis."""

        raw_blocks = page.get_text(
            "blocks",
            sort=True,
        )

        blocks = []

        for block in raw_blocks:

            if len(block) < 5:
                continue

            x0, y0, x1, y1, text = block[:5]

            text = self._extract_block_text(
                {
                    "x0": x0,
                    "y0": y0,
                    "x1": x1,
                    "y1": y1,
                    "text": text,
                }
            )

            if not text:
                continue

            if self._is_noise(text):
                continue

            block_type = self._classify_block(
                text
            )

            blocks.append(
                {
                    "x0": x0,
                    "y0": y0,
                    "x1": x1,
                    "y1": y1,
                    "text": text,
                    "type": block_type,
                }
            )

        has_columns = self._detect_columns(
            blocks
        )

        if has_columns:
            ordered_blocks = self._extract_multicolumn(
                blocks
            )
        else:
            ordered_blocks = self._extract_singlecolumn(
                blocks
            )

        text = self._combine_blocks(
            ordered_blocks
        )

        has_math = any(
            block["type"] == "math"
            for block in ordered_blocks
        )

        has_tables = self._detect_tables(
            ordered_blocks
        )

        confidence = self._calculate_page_confidence(
            text,
            has_columns,
            page_num,
        )

        return ExtractedPage(
            page_num=page_num,
            text=text,
            blocks=ordered_blocks,
            has_columns=has_columns,
            has_math=has_math,
            has_tables=has_tables,
            confidence=confidence,
        )

    def _detect_columns(
        self,
        blocks: List[Dict],
    ) -> bool:
        """Detect whether a page probably has multiple columns."""

        if len(blocks) < 4:
            return False

        page_width = max(
            block["x1"]
            for block in blocks
        )

        if page_width <= 0:
            return False

        left_blocks = 0
        right_blocks = 0

        midpoint = page_width / 2

        for block in blocks:

            center_x = (
                block["x0"] + block["x1"]
            ) / 2

            if center_x < midpoint:
                left_blocks += 1
            else:
                right_blocks += 1

        return (
            left_blocks >= 2
            and right_blocks >= 2
        )

    def _extract_multicolumn(
        self,
        blocks: List[Dict],
    ) -> List[Dict]:
        """Order blocks from left column to right column."""

        columns = self._group_by_columns(
            blocks
        )

        ordered = []

        for column in columns:
            column.sort(
                key=lambda block: block["y0"]
            )

            ordered.extend(column)

        return ordered

    def _extract_singlecolumn(
        self,
        blocks: List[Dict],
    ) -> List[Dict]:
        """Order blocks vertically."""

        return sorted(
            blocks,
            key=lambda block: (
                block["y0"],
                block["x0"],
            ),
        )

    def _extract_block_text(
        self,
        block: Dict,
    ) -> str:
        """Clean text from an individual block."""

        text = block.get(
            "text",
            "",
        )

        text = text.replace(
            "\x00",
            "",
        )

        text = re.sub(
            r"[ \t]+",
            " ",
            text,
        )

        text = re.sub(
            r"\n{3,}",
            "\n\n",
            text,
        )

        return text.strip()

    def _group_by_columns(
        self,
        blocks: List[Dict],
    ) -> List[List[Dict]]:
        """Group blocks into approximate columns."""

        if not blocks:
            return []

        page_left = min(
            block["x0"]
            for block in blocks
        )

        page_right = max(
            block["x1"]
            for block in blocks
        )

        midpoint = (
            page_left + page_right
        ) / 2

        left = []
        right = []

        for block in blocks:

            center_x = (
                block["x0"] + block["x1"]
            ) / 2

            if center_x < midpoint:
                left.append(block)
            else:
                right.append(block)

        columns = []

        if left:
            columns.append(left)

        if right:
            columns.append(right)

        return columns

    def _combine_blocks(
        self,
        blocks: List[Dict],
    ) -> str:
        """Combine blocks into continuous text."""

        parts = []

        for block in blocks:

            text = block["text"].strip()

            if not text:
                continue

            parts.append(text)

        return "\n\n".join(parts)

    def _is_noise(
        self,
        text: str,
    ) -> bool:
        """Detect obvious headers, footers, and page numbers."""

        text = text.strip()

        if not text:
            return True

        # A block consisting only of a page number.
        if re.fullmatch(
            r"(page\s*)?\d+",
            text,
            flags=re.IGNORECASE,
        ):
            return True

        # Very short standalone noise.
        if len(text) <= 2:
            return True

        return False

    def _classify_block(
        self,
        text: str,
    ) -> str:
        """Classify a text block."""

        stripped = text.strip()

        # Common academic section headings.
        if re.match(
            r"^(abstract|introduction|background|"
            r"related work|methodology|methods|"
            r"experiments|results|discussion|"
            r"conclusion|conclusions|references)"
            r"\s*$",
            stripped,
            flags=re.IGNORECASE,
        ):
            return "heading"

        # Numbered headings such as:
        # 1 Introduction
        # 2.1 Dataset
        if re.match(
            r"^\d+(\.\d+)*\.?\s+\S+",
            stripped,
        ):
            if len(stripped) < 150:
                return "heading"

        # Simple mathematical-looking blocks.
        math_patterns = [
            r"\$.*\$",
            r"\\frac",
            r"\\sum",
            r"\\alpha",
            r"\\beta",
            r"∑",
            r"∫",
            r"≤",
            r"≥",
            r"≈",
        ]

        if any(
            re.search(pattern, stripped)
            for pattern in math_patterns
        ):
            return "math"

        # Figure/table captions.
        if re.match(
            r"^(figure|fig\.|table)\s+\d+",
            stripped,
            flags=re.IGNORECASE,
        ):
            return "caption"

        return "body"

    def _detect_tables(
        self,
        blocks: List[Dict],
    ) -> bool:
        """Heuristically detect tables."""

        table_keywords = re.compile(
            r"\b(table|column|row)\b",
            flags=re.IGNORECASE,
        )

        for block in blocks:

            if block["type"] == "caption":
                if re.match(
                    r"^table",
                    block["text"],
                    flags=re.IGNORECASE,
                ):
                    return True

            if table_keywords.search(
                block["text"]
            ):
                return True

        return False

    def _identify_sections(
        self,
        text: str,
    ) -> List[Dict]:
        """Identify likely document sections."""

        lines = text.splitlines()

        sections = []

        current_section = "Unknown"
        current_start = 0

        heading_pattern = re.compile(
            r"^(?:"
            r"\d+(?:\.\d+)*\.?\s+)?"
            r"(abstract|introduction|background|"
            r"related work|methods?|methodology|"
            r"experiments?|results?|discussion|"
            r"conclusions?|references)"
            r"$",
            flags=re.IGNORECASE,
        )

        position = 0

        for line in lines:

            stripped = line.strip()

            if not stripped:
                position += len(line) + 1
                continue

            if heading_pattern.match(
                stripped
            ):
                if current_start < position:
                    sections.append(
                        {
                            "name": current_section,
                            "start": current_start,
                            "end": position,
                        }
                    )

                current_section = stripped
                current_start = position

            position += len(line) + 1

        if current_start < len(text):
            sections.append(
                {
                    "name": current_section,
                    "start": current_start,
                    "end": len(text),
                }
            )

        return sections

    def _clean_extracted_text(
        self,
        text: str,
    ) -> str:
        """Clean extracted text for downstream processing."""

        # Remove excessive whitespace.
        text = re.sub(
            r"[ \t]+",
            " ",
            text,
        )

        # Normalize excessive blank lines.
        text = re.sub(
            r"\n{3,}",
            "\n\n",
            text,
        )

        # Fix spaces before punctuation.
        text = re.sub(
            r"\s+([,.;:!?])",
            r"\1",
            text,
        )

        return text.strip()

    def _calculate_page_confidence(
        self,
        text: str,
        has_columns: bool,
        page_num: int,
    ) -> float:
        """Calculate a simple page extraction confidence score."""

        if not text.strip():
            return 0.0

        score = 1.0

        # Very little text may indicate a problematic page.
        if len(text) < 100:
            score -= 0.25

        # Replacement characters often indicate encoding issues.
        replacement_count = text.count("�")

        if replacement_count:
            score -= min(
                0.5,
                replacement_count * 0.05,
            )

        # Column detection is not itself bad,
        # but adds complexity.
        if has_columns:
            score -= 0.05

        return max(
            0.0,
            min(1.0, score),
        )

    def _calculate_confidence(
        self,
        pages: List[ExtractedPage],
    ) -> float:
        """Calculate overall extraction confidence."""

        if not pages:
            return 0.0

        return round(
            sum(
                page.confidence
                for page in pages
            ) / len(pages),
            3,
        )
```

Extract one PDF:

```bash
python - <<'PY'
from pathlib import Path

from src.pdf_extractor import PDFExtractor

pdf_files = sorted(Path("pdf").rglob("*.pdf"))

if not pdf_files:
    print("No PDF files found. Put a PDF in pdf/ first.")
    raise SystemExit(0)

pdf_path = pdf_files[0]
print(f"Testing PDF:\n{pdf_path}\n")

result = PDFExtractor().extract_paper_text(str(pdf_path))

print("=" * 60)
print("Extraction results")
print("=" * 60)
print(f"Pages:      {result['page_count']}")
print(f"Confidence: {result['confidence']}")
print(f"Sections:   {len(result['sections'])}")

print("\nDetected sections:")
for section in result["sections"]:
    print(f"  - {section['name']}")

print("\nFirst 2000 characters:")
print("-" * 60)
print(result["text"][:2000])
PY
```

Now we turn:

```text
paper.pdf
   |
   v
PDFExtractor
   |
   v
clean structured text
   |
   v
TextChunker
   |
   v
chunks
   |
   v
later: embeddings
```

---

## 4. `src/text_chunker.py`

```bash
nano src/text_chunker.py
```

```python
import logging
from typing import Dict, List, Optional

import nltk
from nltk.tokenize import sent_tokenize


logger = logging.getLogger(__name__)


# Download sentence tokenizer data if necessary.
try:
    nltk.data.find(
        "tokenizers/punkt_tab"
    )
except LookupError:
    nltk.download(
        "punkt_tab",
        quiet=True,
    )


class TextChunker:
    """Intelligent text chunking for academic papers."""

    def __init__(
        self,
        target_chunk_size: int = 768,
        min_chunk_size: int = 256,
        max_chunk_size: int = 1024,
        overlap_size: int = 128,
    ):
        self.target_chunk_size = target_chunk_size
        self.min_chunk_size = min_chunk_size
        self.max_chunk_size = max_chunk_size
        self.overlap_size = overlap_size

    def chunk_paper(
        self,
        text: str,
        sections: List[Dict],
        preserve_sections: bool = True,
    ) -> List[Dict]:
        """
        Chunk paper text intelligently.

        Returns a list of chunks with metadata.
        """

        if not text.strip():
            return []

        if not preserve_sections or not sections:
            return self._chunk_text(text)

        chunks = []

        for section in sections:

            start = section["start"]
            end = section["end"]

            section_text = text[
                start:end
            ].strip()

            if not section_text:
                continue

            section_name = section.get(
                "name",
                "Unknown",
            )

            section_chunks = self._chunk_text(
                section_text,
                section_name=section_name,
            )

            chunks.extend(
                section_chunks
            )

        # Add global chunk indexes.
        for index, chunk in enumerate(
            chunks
        ):
            chunk["chunk_index"] = index

        return chunks

    def _chunk_text(
        self,
        text: str,
        section_name: Optional[str] = None,
    ) -> List[Dict]:
        """Chunk text using sentence boundaries."""

        sentences = sent_tokenize(
            text
        )

        chunks = []

        current_sentences = []
        current_tokens = 0

        for sentence in sentences:

            sentence_tokens = self._count_tokens(
                sentence
            )

            # Extremely long individual sentence.
            if sentence_tokens > self.max_chunk_size:

                if current_sentences:
                    chunks.append(
                        self._make_chunk(
                            current_sentences,
                            section_name,
                        )
                    )

                    current_sentences = []
                    current_tokens = 0

                chunks.append(
                    {
                        "text": sentence.strip(),
                        "token_count": sentence_tokens,
                        "section_name": section_name,
                    }
                )

                continue

            would_exceed = (
                current_tokens
                + sentence_tokens
                > self.max_chunk_size
            )

            if (
                would_exceed
                and current_sentences
            ):

                chunks.append(
                    self._make_chunk(
                        current_sentences,
                        section_name,
                    )
                )

                overlap = self._get_overlap_sentences(
                    current_sentences,
                    self.overlap_size,
                )

                current_sentences = overlap

                current_tokens = sum(
                    self._count_tokens(sentence)
                    for sentence in current_sentences
                )

            current_sentences.append(
                sentence
            )

            current_tokens += sentence_tokens

            # Once we're around the target,
            # allow the next sentence to decide
            # whether the chunk should close.
            if (
                current_tokens
                >= self.target_chunk_size
            ):
                continue

        if current_sentences:
            chunks.append(
                self._make_chunk(
                    current_sentences,
                    section_name,
                )
            )

        # Merge tiny trailing chunks.
        chunks = self._merge_small_chunks(
            chunks
        )

        return chunks

    def _get_overlap_sentences(
        self,
        sentences: List[str],
        target_tokens: int,
    ) -> List[str]:
        """Get sentences from the end for overlap."""

        overlap = []
        token_count = 0

        for sentence in reversed(
            sentences
        ):
            sentence_tokens = self._count_tokens(
                sentence
            )

            if (
                token_count
                + sentence_tokens
                > target_tokens
            ):
                break

            overlap.insert(
                0,
                sentence,
            )

            token_count += sentence_tokens

        return overlap

    def _make_chunk(
        self,
        sentences: List[str],
        section_name: Optional[str],
    ) -> Dict:
        """Create a chunk dictionary."""

        text = " ".join(
            sentence.strip()
            for sentence in sentences
        )

        return {
            "text": text,
            "token_count": self._count_tokens(
                text
            ),
            "section_name": section_name,
        }

    def _count_tokens(
        self,
        text: str,
    ) -> int:
        """
        Estimate token count.

        This is intentionally simple for now.
        The actual embedding tokenizer may use
        a somewhat different token count.
        """

        if not text.strip():
            return 0

        return len(
            text.split()
        )

    def _merge_small_chunks(
        self,
        chunks: List[Dict],
    ) -> List[Dict]:
        """Merge chunks that are too small."""

        if not chunks:
            return []

        result = []

        for chunk in chunks:

            if (
                result
                and chunk["token_count"]
                < self.min_chunk_size
                and result[-1]["token_count"]
                + chunk["token_count"]
                <= self.max_chunk_size
            ):
                previous = result[-1]

                previous["text"] = (
                    previous["text"]
                    + " "
                    + chunk["text"]
                )

                previous["token_count"] = (
                    self._count_tokens(
                        previous["text"]
                    )
                )

            else:
                result.append(chunk)

        return result
```

---

## 5. `src/embeddings.py`

```bash
nano src/embeddings.py
```

```python
import logging
from typing import List

import numpy as np
import torch
from sentence_transformers import SentenceTransformer
from config.settings import (
    EMBEDDING_BATCH_SIZE,
    EMBEDDING_DEVICE,
    EMBEDDING_MODEL,
)


logger = logging.getLogger(__name__)


class EmbeddingGenerator:
    """Generate embeddings using Sentence Transformers."""

    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super(
                EmbeddingGenerator,
                cls
            ).__new__(cls)

            cls._instance._initialized = False

        return cls._instance

    def __init__(
        self,
        model_name: str = EMBEDDING_MODEL,
        device: str = EMBEDDING_DEVICE,
    ):
        if self._initialized:
            return

        logger.info(
            "Loading embedding model: %s",
            model_name,
        )

        self.model_name = model_name
        self.device = device

        self.model = SentenceTransformer(
            model_name,
            device=device,
        )

        self.embedding_dimension = (
            self.model.get_sentence_embedding_dimension()
        )

        logger.info(
            "Embedding dimension: %d",
            self.embedding_dimension,
        )

        self._initialized = True

    def generate_embeddings(
        self,
        texts: List[str],
        show_progress: bool = True,
    ) -> np.ndarray:
        """Generate embeddings for a list of texts."""

        if not texts:
            return np.empty(
                (0, self.embedding_dimension),
                dtype=np.float32,
            )

        cleaned_texts = [
            self._preprocess_text(text)
            for text in texts
        ]

        embeddings = self.model.encode(
            cleaned_texts,
            batch_size=EMBEDDING_BATCH_SIZE,
            show_progress_bar=show_progress,
            convert_to_numpy=True,
            normalize_embeddings=True,
        )

        return embeddings.astype(
            np.float32
        )

    def _preprocess_text(
        self,
        text: str,
    ) -> str:
        """Clean text before embedding generation."""

        if not text:
            return ""

        # Normalize whitespace.
        text = " ".join(
            text.split()
        )

        return text.strip()

    def generate_query_embedding(
        self,
        query: str,
    ) -> np.ndarray:
        """Generate an embedding for a search query."""

        embeddings = self.generate_embeddings(
            [query],
            show_progress=False,
        )

        return embeddings[0]
```

Note: newer versions of `sentence-transformers` rename
`get_sentence_embedding_dimension()` to `get_embedding_dimension()`;
the `FutureWarning` you may see is harmless.

Quick check:

```bash
python - <<'PY'
from src.embeddings import EmbeddingGenerator

generator = EmbeddingGenerator()

texts = [
    "Transformers use self-attention mechanisms.",
    "PostgreSQL can store vector embeddings.",
    "Cats are domestic animals.",
]

embeddings = generator.generate_embeddings(texts, show_progress=False)

print(f"Shape:  {embeddings.shape}")
print(f"Dtype:  {embeddings.dtype}")
print(f"Dim:    {generator.embedding_dimension}")

query_embedding = generator.generate_query_embedding(
    "How do transformer models work?"
)
print(f"Query:  {query_embedding.shape}")
PY
```

```text
Shape:  (3, 384)
Dtype:  float32
Dim:    384
Query:  (384,)
```

---

## 6. `src/embedding_pipeline.py`

Extracts chunks for every registered paper that still needs embeddings,
then stores them (with vectors) in `paper_chunks`.

```bash
nano src/embedding_pipeline.py
```

```python
import logging
import re
from typing import Dict, List

import numpy as np
import psycopg2
from psycopg2.extras import execute_batch

from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, EMBEDDING_BATCH_SIZE
from src.embeddings import EmbeddingGenerator
from src.pdf_extractor import PDFExtractor
from src.text_chunker import TextChunker

logger = logging.getLogger(__name__)


class EmbeddingPipeline:
    def __init__(self, db_config: dict, embedding_generator: EmbeddingGenerator, batch_size: int = EMBEDDING_BATCH_SIZE):
        self.db_config = db_config
        self.embedding_generator = embedding_generator
        self.batch_size = batch_size

    def _get_connection(self):
        return psycopg2.connect(**self.db_config)

    def process_paper(self, paper_id: int, chunks: List[Dict]) -> Dict[str, object]:
        if not chunks:
            return {"paper_id": paper_id, "chunks": 0, "embedded": 0, "status": "empty"}
        embeddings = self.embedding_generator.generate_embeddings([chunk["text"] for chunk in chunks], show_progress=True)
        conn = self._get_connection()
        try:
            with conn.cursor() as cursor:
                cursor.execute("DELETE FROM paper_chunks WHERE paper_id = %s", (paper_id,))
                rows = []
                for index, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
                    text = chunk["text"]
                    rows.append((paper_id, index, text, chunk.get("token_count"), embedding.tolist(),
                                 chunk.get("section_name"), chunk.get("page_number"), chunk.get("char_start"),
                                 chunk.get("char_end"), self._detect_math(text), self._detect_code(text),
                                 self._detect_references(text)))
                execute_batch(cursor, """
                    INSERT INTO paper_chunks
                    (paper_id, chunk_index, chunk_text, chunk_tokens, embedding,
                     section_name, page_number, char_start, char_end, has_math, has_code, has_references)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, rows, page_size=self.batch_size)
                cursor.execute("""
                    UPDATE papers SET embedding_generated = TRUE, pdf_processed = TRUE,
                    processing_error = NULL, updated_at = CURRENT_TIMESTAMP WHERE id = %s
                """, (paper_id,))
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()
        return {"paper_id": paper_id, "chunks": len(chunks), "embedded": len(embeddings), "status": "completed"}

    @staticmethod
    def _detect_math(text):
        return any(re.search(pattern, text, re.IGNORECASE) for pattern in [r"\\frac", r"\\sum", r"\\int", r"∑", r"∫", r"≤", r"≥", r"≈", r"\bEquation\s+\d+"])

    @staticmethod
    def _detect_code(text):
        return any(re.search(pattern, text, re.IGNORECASE) for pattern in [r"\bdef\s+\w+\(", r"\bclass\s+\w+", r"\bimport\s+\w+", r"```"])

    @staticmethod
    def _detect_references(text):
        return any(re.search(pattern, text, re.IGNORECASE) for pattern in [r"\[\d+\]", r"\(\w+\s+et al\.,?\s+\d{4}\)", r"\bdoi:\s*10\."])

    def process_pending_papers(self, limit: int = 10) -> Dict[str, object]:
        conn = self._get_connection()
        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT id, pdf_path FROM papers
                    WHERE embedding_generated = FALSE
                    ORDER BY id LIMIT %s
                """, (limit,))
                papers = cursor.fetchall()
        finally:
            conn.close()

        processed = failed = 0
        for paper_id, pdf_path in papers:
            try:
                extracted = PDFExtractor().extract_paper_text(pdf_path)
                chunks = TextChunker().chunk_paper(text=extracted["text"], sections=extracted["sections"])
                self.process_paper(paper_id, chunks)
                processed += 1
            except Exception as error:
                failed += 1
                logger.exception("Failed to process paper %d: %s", paper_id, error)
                self._record_error(paper_id, str(error))
        return {"requested": limit, "found": len(papers), "processed": processed, "failed": failed}

    def _record_error(self, paper_id: int, error: str):
        conn = self._get_connection()
        try:
            with conn.cursor() as cursor:
                cursor.execute("UPDATE papers SET processing_error = %s, updated_at = CURRENT_TIMESTAMP WHERE id = %s", (error, paper_id))
            conn.commit()
        finally:
            conn.close()


def create_default_pipeline():
    return EmbeddingPipeline(
        db_config={"host": DB_HOST, "port": DB_PORT, "dbname": DB_NAME, "user": DB_USER, "password": DB_PASSWORD},
        embedding_generator=EmbeddingGenerator(),
    )
```

Note: `process_pending_papers()` reads `pdf_path` straight from the
`papers` row - there is no `arxiv_id` lookup and no `data/pdfs/YYYY/`
folder to search.

---

## 7. `src/search.py`

```bash
nano src/search.py
```

```python
import logging
from dataclasses import dataclass
from enum import Enum
from typing import Dict, List, Optional

import psycopg2
from psycopg2.extras import RealDictCursor
from src.embeddings import EmbeddingGenerator

logger = logging.getLogger(__name__)


class SearchMode(Enum):
    VECTOR = "vector"
    HYBRID = "hybrid"
    KEYWORD = "keyword"


@dataclass
class SearchResult:
    paper_id: int
    filename: str
    title: str
    score: float
    matched_chunks: List[Dict]


class PaperSearchEngine:
    def __init__(self, db_config: dict, embedding_generator: EmbeddingGenerator):
        self.db_config = db_config
        self.embedding_generator = embedding_generator

    def _get_connection(self):
        return psycopg2.connect(**self.db_config)

    def search(self, query: str, mode=SearchMode.VECTOR, limit=10, filters: Optional[Dict] = None):
        if not query or limit <= 0:
            return []
        if mode == SearchMode.VECTOR:
            return self._vector_search(query, limit)
        if mode == SearchMode.KEYWORD:
            return self._keyword_search(query, limit)
        if mode == SearchMode.HYBRID:
            return self._combine_results(self._vector_search(query, limit * 2), self._keyword_search(query, limit * 2), limit)
        raise ValueError(f"Unknown search mode: {mode}")

    def _vector_search(self, query, limit):
        embedding = self.embedding_generator.generate_query_embedding(query)
        sql = """
            SELECT p.id AS paper_id, p.filename, p.title,
                   1 - (c.embedding <=> %s::vector) AS score,
                   c.id AS chunk_id, c.chunk_index, c.chunk_text,
                   c.section_name, c.page_number
            FROM paper_chunks c JOIN papers p ON p.id = c.paper_id
            WHERE c.embedding IS NOT NULL
            ORDER BY c.embedding <=> %s::vector LIMIT %s
        """
        with self._get_connection() as conn, conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(sql, (embedding.tolist(), embedding.tolist(), max(limit * 10, 50)))
            return self._group(cur.fetchall(), limit)

    def _keyword_search(self, query, limit):
        sql = """
            SELECT p.id AS paper_id, p.filename, p.title,
                   similarity(c.chunk_text, %s) AS score,
                   c.id AS chunk_id, c.chunk_index, c.chunk_text,
                   c.section_name, c.page_number
            FROM paper_chunks c JOIN papers p ON p.id = c.paper_id
            WHERE c.chunk_text %% %s
            ORDER BY similarity(c.chunk_text, %s) DESC LIMIT %s
        """
        with self._get_connection() as conn, conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(sql, (query, query, query, max(limit * 10, 50)))
            return self._group(cur.fetchall(), limit)

    def _group(self, rows, limit):
        grouped = {}
        for row in rows:
            item = grouped.setdefault(row["paper_id"], {
                "paper_id": row["paper_id"], "filename": row["filename"],
                "title": row["title"], "score": float(row["score"]), "matched_chunks": []
            })
            item["score"] = max(item["score"], float(row["score"]))
            item["matched_chunks"].append({
                "chunk_id": row["chunk_id"], "chunk_index": row["chunk_index"],
                "text": row["chunk_text"], "section_name": row["section_name"],
                "page_number": row["page_number"], "score": float(row["score"])
            })
        results = sorted(grouped.values(), key=lambda x: x["score"], reverse=True)
        for result in results:
            result["matched_chunks"] = sorted(result["matched_chunks"], key=lambda x: x["score"], reverse=True)[:3]
        return [SearchResult(**result) for result in results[:limit]]

    def _combine_results(self, vectors, keywords, limit):
        combined = {r.paper_id: [r, r.score, 0.0] for r in vectors}
        for result in keywords:
            if result.paper_id in combined:
                combined[result.paper_id][2] = result.score
            else:
                combined[result.paper_id] = [result, 0.0, result.score]
        output = []
        for result, vector_score, keyword_score in combined.values():
            result.score = 0.7 * vector_score + 0.3 * keyword_score
            output.append(result)
        return sorted(output, key=lambda x: x.score, reverse=True)[:limit]

    def find_similar_papers(self, paper_id, limit=10):
        with self._get_connection() as conn, conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("SELECT embedding FROM paper_chunks WHERE paper_id = %s AND embedding IS NOT NULL ORDER BY id LIMIT 1", (paper_id,))
            row = cur.fetchone()
            if not row:
                return []
            cur.execute("""
                SELECT p.id AS paper_id, p.filename, p.title,
                       1 - (c.embedding <=> %s::vector) AS score,
                       c.id AS chunk_id, c.chunk_index, c.chunk_text,
                       c.section_name, c.page_number
                FROM paper_chunks c JOIN papers p ON p.id = c.paper_id
                WHERE c.embedding IS NOT NULL AND p.id != %s
                ORDER BY c.embedding <=> %s::vector LIMIT %s
            """, (row["embedding"], paper_id, row["embedding"], max(limit * 10, 50)))
            return self._group(cur.fetchall(), limit)
```

---

## 8. `index.py` - entry points

There is no `scripts/` folder. Everything is wired together in `index.py`:

```bash
nano index.py
```

```python
"""Minimal entry points for local PDF indexing and semantic search."""

from config.settings import DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
from src.embeddings import EmbeddingGenerator
from src.paper_processor import create_default_processor
from src.embedding_pipeline import EmbeddingPipeline
from src.search import PaperSearchEngine, SearchMode

DB_CONFIG = {
    "host": DB_HOST,
    "port": DB_PORT,
    "dbname": DB_NAME,
    "user": DB_USER,
    "password": DB_PASSWORD,
}


def ingest_and_embed(limit: int = 100):
    """Register PDFs in pdf/, extract chunks, and generate embeddings."""
    registration = create_default_processor().ingest_pdfs()
    pipeline = EmbeddingPipeline(DB_CONFIG, EmbeddingGenerator())
    processing = pipeline.process_pending_papers(limit=limit)
    return {"registration": registration, "processing": processing}


def search(query: str, mode: SearchMode = SearchMode.HYBRID, limit: int = 5):
    """Search indexed PDFs and return ranked results with matching chunks."""
    return PaperSearchEngine(DB_CONFIG, EmbeddingGenerator()).search(
        query=query,
        mode=mode,
        limit=limit,
    )
```

Run the whole pipeline (register + extract + chunk + embed):

```bash
python - <<'PY'
from index import ingest_and_embed

print(ingest_and_embed())
PY
```

```text
{'registration': {'found': 2, 'new': 2, 'existing': 0, 'failed': 0},
 'processing': {'requested': 100, 'found': 2, 'processed': 2, 'failed': 0}}
```

---

## 9. Verify the database

Before testing search, make sure the vectors are really there:

```bash
psql -h localhost -U furba -d pdf_vector -c "SELECT COUNT(*) AS chunks FROM paper_chunks;"
```

```bash
psql -h localhost -U furba -d pdf_vector -c "SELECT COUNT(*) AS embedded_chunks FROM paper_chunks WHERE embedding IS NOT NULL;"
```

```bash
psql -h localhost -U furba -d pdf_vector -c "SELECT id, paper_id, chunk_index, section_name, vector_dims(embedding) AS dimensions FROM paper_chunks LIMIT 10;"
```

You should get something similar to:

```text
 id | paper_id | chunk_index | section_name | dimensions
----+----------+-------------+--------------+-----------
  1 |     1    |      0      | Introduction |    384
  2 |     1    |      1      | Introduction |    384
  3 |     1    |      2      | Methods      |    384
 ...
```

```text
chunks > 0
embedded_chunks > 0
dimensions = 384
```

---

## 10. Test search

```bash
python - <<'PY'
from index import search
from src.search import SearchMode

query = "What does a LoRA-generating hypernetwork produce as output on a user's device?"

print("\nSearching...")
print("=" * 70)
print(f"Query: {query}")

results = search(query, mode=SearchMode.VECTOR, limit=5)

print("\nResults:")
print("=" * 70)

for i, result in enumerate(results, start=1):
    print(f"\n{i}. {result.title}")
    print(f"   File:  {result.filename}")
    print(f"   Score: {result.score:.4f}")

    if result.matched_chunks:
        chunk = result.matched_chunks[0]
        print(f"   Section: {chunk['section_name']}")
        print(f"   Page:    {chunk['page_number']}")
        text = chunk["text"].replace("\n", " ")
        print(f"   Match:   {text[:300]}...")
PY
```

Example output (with a LoRA-hypernetwork paper in `pdf/`):

```text
Results:
======================================================================

1. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   File:  2609.24979v1.pdf
   Score: 0.5470
   Section: Introduction
   Page:    None
   Match:   Introduction Corresponding author: saugenst@google.com. Google. Preprint. LoRA-generating hypernetworks ...

2. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   File:  2609.24979v1.pdf
   Score: 0.5142
   ...
```

Note that `SearchResult` carries `filename` and `title` only - there are
no `arxiv_id`, `authors`, `abstract`, or `categories` fields, because the
project indexes local PDFs, not ArXiv metadata.

---

## Notes

### About the score

Your highest score might look like:

```text
0.5470
```

Don't worry about the number being "only" 0.547. The score is based on:

```sql
1 - (embedding <=> query_embedding)
```

and the absolute value is not a universal "percentage of relevance".

```text
0.547 != 54.7% relevant
```

It is a similarity measure useful primarily for **ranking**.

Hybrid mode combines the two signals as:

```text
score = 0.7 * vector_score + 0.3 * keyword_score
```

### Paper-level grouping is already done

The original note in this document said we were returning five chunks
from the same paper and that grouping by `paper_id` should be built next.
In this project that step is **already implemented** in
`PaperSearchEngine._group()`:

```text
            user query
                |
                v
        query embedding
                |
                v
            pgvector  (fetch max(limit * 10, 50) chunk rows)
                |
                v
        group by paper_id
                |
                v
  best score per paper + top 3 matched_chunks
                |
                v
        top `limit` papers
```

So each paper appears **once**, with its best score and up to three
`matched_chunks` attached:

```text
Paper A -> 0.82  (chunks 17, 21, 5)
Paper B -> 0.76  (chunks 5, 9)
Paper C -> 0.71  (chunks 8)
```

### Your system has reached an important milestone

```text
        pdf/*.pdf
            |
            v
      PDFStorage + PaperProcessor
            |
            v
        PDFExtractor
            |
            v
        TextChunker
            |
            v
    all-MiniLM-L6-v2
            |
       384 dimensions
            |
            v
        PostgreSQL
          pgvector
            |
            v
        HNSW index
            |
            v
      vector / keyword / hybrid search
            |
            v
   relevant papers + chunks
```

The **core semantic-search mechanism works**.

### What to build next

```text
vector search          (done)
   |
paper-level ranking    (done - _group())
   |
metadata filtering     (todo)
   |
keyword / hybrid       (done)
   |
RAG                    (todo)
   |
interactive CLI        (todo)
```

Concretely:

1. **Filters are stubbed but unused.** `PaperSearchEngine.search()`
   accepts a `filters` argument, yet `_vector_search()` and
   `_keyword_search()` never receive it. Add a WHERE-clause builder
   (e.g. filter on `filename`, `title`, or `has_math` / `has_code`
   flags on chunks - there are no category/author columns in this
   schema).
2. **Interactive CLI** on top of `index.search()`.
3. **RAG**: feed `matched_chunks` into a generator model.

### Import paths (the book vs. this project)

The book's snippets assume a flat layout:

```python
from arxiv_client import ArxivClient
```

This project uses packages, and the ArXiv client does not exist here:

```text
.
├── index.py            -> from index import search, ingest_and_embed
├── config/settings.py  -> from config.settings import DB_HOST, ...
└── src/*.py            -> from src.search import PaperSearchEngine, SearchMode
```

Rules of thumb:

- Always run Python from the project root so `config` and `src` resolve.
- Inside `src/` modules, import siblings as `from src.module import ...`
  (and `from config.settings import ...`), never as `import module`.
- The virtualenv lives in `env/`.
