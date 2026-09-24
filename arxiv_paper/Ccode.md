```text
                  ArXiv
                    │
                    │ API
                    ▼
             ArxivClient
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      Search     Fetch IDs   Recent papers
        │           │           │
        └───────────┼───────────┘
                    ▼
              ArxivPaper
                    │
                    ▼
              PostgreSQL
```

`ArxivPaper` object is clean Python representation of one paper.

---

```bash
nano src/arxiv_client.py
```

```python
import time
import logging
from typing import List, Optional, Generator
from datetime import datetime, timedelta
from dataclasses import dataclass

import arxiv

from config.settings import (
    ARXIV_MAX_RESULTS,
    ARXIV_RATE_LIMIT,
)


logger = logging.getLogger(__name__)


@dataclass
class ArxivPaper:
    """Structured representation of an ArXiv paper."""

    arxiv_id: str
    title: str
    abstract: str
    authors: List[str]
    categories: List[str]
    primary_category: str
    published_date: datetime
    updated_date: datetime
    pdf_url: str
    comment: Optional[str] = None
    journal_ref: Optional[str] = None
    doi: Optional[str] = None


class ArxivClient:
    """ArXiv API client with rate limiting and error handling."""

    def __init__(
        self,
        rate_limit_seconds: float = ARXIV_RATE_LIMIT,
        max_results_per_query: int = ARXIV_MAX_RESULTS,
    ):
        self.rate_limit_seconds = rate_limit_seconds
        self.max_results_per_query = max_results_per_query

        # Time when the previous API request was made.
        self._last_request_time = 0.0

        # The arxiv Python package handles communication with ArXiv.
        self.client = arxiv.Client(
            page_size=max_results_per_query,
            delay_seconds=rate_limit_seconds,
            num_retries=3,
        )

    def _rate_limit(self):
        """Enforce minimum time between API requests."""

        elapsed = time.time() - self._last_request_time

        if elapsed < self.rate_limit_seconds:
            wait_time = self.rate_limit_seconds - elapsed

            logger.debug(
                "Rate limiting: sleeping %.2f seconds",
                wait_time,
            )

            time.sleep(wait_time)

        self._last_request_time = time.time()

    def _convert_result(self, result: arxiv.Result) -> ArxivPaper:
        """Convert an arxiv.Result into our ArxivPaper object."""

        return ArxivPaper(
            arxiv_id=result.entry_id.split("/")[-1],
            title=result.title.strip(),
            abstract=result.summary.strip(),
            authors=[author.name for author in result.authors],
            categories=result.categories,
            primary_category=result.primary_category,
            published_date=result.published,
            updated_date=result.updated,
            pdf_url=result.pdf_url,
            comment=result.comment,
            journal_ref=result.journal_ref,
            doi=result.doi,
        )

    def search_papers(
        self,
        query: str,
        max_results: int = 100,
        sort_by: arxiv.SortCriterion = arxiv.SortCriterion.SubmittedDate,
        sort_order: arxiv.SortOrder = arxiv.SortOrder.Descending,
    ) -> Generator[ArxivPaper, None, None]:
        """
        Search for papers matching an ArXiv query.

        Examples:

            cat:cs.LG

            au:Bengio

            ti:transformer

            cat:cs.LG AND (ti:attention OR ti:transformer)
        """

        search = arxiv.Search(
            query=query,
            max_results=max_results,
            sort_by=sort_by,
            sort_order=sort_order,
        )

        logger.info("Searching ArXiv: %s", query)

        try:
            for result in self.client.results(search):
                yield self._convert_result(result)

        except Exception:
            logger.exception(
                "ArXiv search failed for query: %s",
                query,
            )
            raise

    def fetch_by_ids(
        self,
        arxiv_ids: List[str],
    ) -> List[ArxivPaper]:
        """Fetch specific papers by their ArXiv IDs."""

        if not arxiv_ids:
            return []

        search = arxiv.Search(
            id_list=arxiv_ids,
            max_results=len(arxiv_ids),
        )

        logger.info(
            "Fetching %d ArXiv papers",
            len(arxiv_ids),
        )

        papers = []

        try:
            for result in self.client.results(search):
                papers.append(self._convert_result(result))

        except Exception:
            logger.exception("Failed to fetch ArXiv papers by ID")
            raise

        return papers

    def search_recent_papers(
        self,
        categories: List[str],
        days_back: int = 7,
    ) -> Generator[ArxivPaper, None, None]:
        """Fetch papers from categories published in the last N days."""

        if not categories:
            return

        cutoff = datetime.now() - timedelta(days=days_back)

        category_query = " OR ".join(
            f"cat:{category}"
            for category in categories
        )

        logger.info(
            "Searching recent papers in %s",
            categories,
        )

        for paper in self.search_papers(
            query=f"({category_query})",
            max_results=self.max_results_per_query,
            sort_by=arxiv.SortCriterion.SubmittedDate,
            sort_order=arxiv.SortOrder.Descending,
        ):
            if paper.published_date >= cutoff:
                yield paper
            else:
                # Results are sorted newest first.
                # Once we reach papers older than our cutoff,
                # we can stop searching.
                break
```

---
## `@dataclass`

```python
@dataclass
class ArxivPaper:
```

creates a simple container for paper information.

Instead of passing around a messy dictionary:

```python
{
    "id": "...",
    "title": "...",
    "authors": [...],
    ...
}
```

we can do:

```python
paper.title
paper.abstract
paper.authors
paper.pdf_url
```

Much easier to work with.

---

```text
ArxivPaper
│
├── arxiv_id
│     └── 2401.12345
│
├── title
│     └── "Attention Is All You Need..."
│
├── abstract
│
├── authors
│     ├── Author 1
│     └── Author 2
│
├── categories
│     ├── cs.LG
│     └── cs.AI
│
├── primary_category
│     └── cs.LG
│
├── published_date
├── updated_date
└── pdf_url
```

---

useful parts of ArXiv API.


### Machine learning

```text
cat:cs.LG
```

### Papers by Bengio

```text
au:Bengio
```

### Transformer in title

```text
ti:transformer
```

### More complex search

```text
cat:cs.LG AND (ti:attention OR ti:transformer)
```

Eventually our system can expose these searches through your own interface.

---

```bash
nano scripts/test_arxiv.py
```

```python
import sys
from pathlib import Path

# Make the project root available for imports.
PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.arxiv_client import ArxivClient


def main():
    client = ArxivClient()

    print("Searching ArXiv...\n")

    papers = client.search_papers(
        query="cat:cs.LG",
        max_results=5,
    )

    for paper in papers:
        print("=" * 60)
        print(f"ArXiv ID: {paper.arxiv_id}")
        print(f"Title:    {paper.title}")
        print(f"Authors:  {', '.join(paper.authors)}")
        print(f"Category: {paper.primary_category}")
        print(f"PDF:      {paper.pdf_url}")


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_arxiv.py
```

```text
Searching ArXiv...

============================================================
ArXiv ID: ...
Title: ...
Authors: ...
Category: cs.LG
PDF: https://arxiv.org/pdf/...
============================================================
...
```

```text
src/
│
├── arxiv_client.py     ← talks to ArXiv
│
├── pdf_processor.py    ← handles PDFs
│
├── embeddings.py       ← creates vectors
│
├── database.py         ← talks to PostgreSQL
│
└── search.py           ← performs searches
```

```text
ArXiv API
   │
   │ metadata + PDF URL
   ▼
ArxivClient
   │
   ▼
PDFDownloader
   │
   ├── retry on network failure
   ├── validate PDF
   ├── organize files
   └── track storage
   │
   ▼
data/pdfs/
```

```bash
nano src/pdf_processor.py
```

```python
import hashlib
import logging
import time
from datetime import datetime
from pathlib import Path
from typing import Dict, Optional, Tuple

import requests

from config.settings import PDF_STORAGE_PATH


logger = logging.getLogger(__name__)


class PDFDownloader:
    """Manages PDF downloads with retry logic and organization."""

    def __init__(
        self,
        storage_path: str = str(PDF_STORAGE_PATH),
        max_retries: int = 3,
        timeout: int = 30,
    ):
        self.storage_path = Path(storage_path)
        self.max_retries = max_retries
        self.timeout = timeout

        # Make sure the storage directory exists.
        self.storage_path.mkdir(
            parents=True,
            exist_ok=True,
        )

    def _get_pdf_path(
        self,
        arxiv_id: str,
        published_date: datetime,
    ) -> Path:
        """Generate organized storage path for a PDF."""

        year = published_date.year

        # ArXiv IDs can contain characters such as '/'.
        # Replace them so they are safe as filenames.
        safe_id = arxiv_id.replace("/", "_")

        year_directory = self.storage_path / str(year)

        year_directory.mkdir(
            parents=True,
            exist_ok=True,
        )

        return year_directory / f"{safe_id}.pdf"

    def download_pdf(
        self,
        pdf_url: str,
        arxiv_id: str,
        published_date: datetime,
        force: bool = False,
    ) -> Tuple[bool, Optional[Path], Optional[str]]:
        """
        Download a PDF with retry logic.

        Returns:
            (success, pdf_path, error_message)
        """

        pdf_path = self._get_pdf_path(
            arxiv_id,
            published_date,
        )

        # Don't download the same PDF again unless force=True.
        if pdf_path.exists() and not force:
            if self._validate_pdf(pdf_path):
                logger.info(
                    "PDF already exists: %s",
                    pdf_path,
                )
                return True, pdf_path, None

            logger.warning(
                "Existing PDF is invalid. Re-downloading: %s",
                pdf_path,
            )

        for attempt in range(self.max_retries):
            try:
                logger.info(
                    "Downloading %s (attempt %d/%d)",
                    arxiv_id,
                    attempt + 1,
                    self.max_retries,
                )

                response = requests.get(
                    pdf_url,
                    timeout=self.timeout,
                    stream=True,
                )

                response.raise_for_status()

                # Download to a temporary file first.
                temp_path = pdf_path.with_suffix(".tmp")

                with open(temp_path, "wb") as file:
                    for chunk in response.iter_content(
                        chunk_size=8192
                    ):
                        if chunk:
                            file.write(chunk)

                response.close()

                # Validate before replacing the final file.
                if not self._validate_pdf(temp_path):
                    temp_path.unlink(missing_ok=True)

                    raise ValueError(
                        "Downloaded file is not a valid PDF"
                    )

                temp_path.replace(pdf_path)

                logger.info(
                    "Downloaded PDF: %s",
                    pdf_path,
                )

                return True, pdf_path, None

            except Exception as error:
                logger.warning(
                    "Download failed for %s: %s",
                    arxiv_id,
                    error,
                )

                # Clean up incomplete download.
                temp_path = pdf_path.with_suffix(".tmp")
                temp_path.unlink(missing_ok=True)

                # If this wasn't the final attempt, wait using
                # exponential backoff.
                if attempt < self.max_retries - 1:
                    wait_seconds = 2 ** attempt

                    logger.info(
                        "Retrying in %d seconds...",
                        wait_seconds,
                    )

                    time.sleep(wait_seconds)

                else:
                    return False, None, str(error)

        return False, None, "Download failed"

    def _validate_pdf(self, pdf_path: Path) -> bool:
        """Check whether a file is a valid PDF."""

        try:
            if not pdf_path.exists():
                return False

            # A PDF should not be empty.
            if pdf_path.stat().st_size < 100:
                return False

            with open(pdf_path, "rb") as file:
                header = file.read(5)

            # PDF files begin with %PDF-
            return header == b"%PDF-"

        except OSError:
            return False

    def get_storage_stats(self) -> Dict[str, object]:
        """Get statistics about stored PDFs."""

        total_files = 0
        total_bytes = 0

        for pdf_path in self.storage_path.rglob("*.pdf"):
            try:
                total_files += 1
                total_bytes += pdf_path.stat().st_size
            except OSError:
                pass

        total_mb = total_bytes / (1024 * 1024)

        return {
            "total_files": total_files,
            "total_bytes": total_bytes,
            "total_mb": round(total_mb, 2),
        }

    def cleanup_old_pdfs(
        self,
        days_to_keep: int = 90,
    ):
        """Remove PDFs older than the specified number of days."""

        cutoff_time = time.time() - (
            days_to_keep * 24 * 60 * 60
        )

        removed = 0

        for pdf_path in self.storage_path.rglob("*.pdf"):
            try:
                if pdf_path.stat().st_mtime < cutoff_time:
                    pdf_path.unlink()
                    removed += 1

                    logger.info(
                        "Removed old PDF: %s",
                        pdf_path,
                    )

            except OSError as error:
                logger.warning(
                    "Could not remove %s: %s",
                    pdf_path,
                    error,
                )

        return removed
```

---


---


Remember your `.env` contains:

```text
PDF_STORAGE_PATH=./data/pdfs
```

So a paper published in 2025 become:

```text
data/
└── pdfs/
    ├── 2024/
    │   └── ...
    │
    └── 2025/
        ├── 2501.12345.pdf
        ├── 2502.54321.pdf
        └── 2503.98765.pdf
```

The year comes from:

```python
published_date.year
```

and the filename comes from:

```text
arxiv_id + ".pdf"
```

This makes the collection easy to browse manually.

---

# Why we download to `.tmp` first

```python
temp_path = pdf_path.with_suffix(".tmp")
```

We don't immediately write to:

```text
2501.12345.pdf
```

Instead:

```text
Download
   │
   ▼
2501.12345.tmp
   │
   ▼
validate PDF
   │
   ├── invalid → delete .tmp
   │
   └── valid
          │
          ▼
2501.12345.pdf
```

That prevents a failed network connection from leaving a half-downloaded file pretending to be a complete PDF.

---
```python
max_retries = 3
```

```text
Attempt 1
   │
   ✗
   │
   └── wait 1 second

Attempt 2
   │
   ✗
   │
   └── wait 2 seconds

Attempt 3
   │
   ✗
   │
   └── give up
```

```python
wait_seconds = 2 ** attempt
```

```text
2^0 = 1 second
2^1 = 2 seconds
2^2 = 4 seconds
...
```

---
```python
header = file.read(5)
return header == b"%PDF-"
```

So we don't accidentally save something like:

```text
<html>
    Service unavailable...
</html>
```

as:

```text
2501.12345.pdf
```

---
```bash
nano scripts/test_pdf.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.pdf_processor import PDFDownloader


def main():
    downloader = PDFDownloader()

    stats = downloader.get_storage_stats()

    print("PDF storage statistics:")
    print(f"  Files: {stats['total_files']}")
    print(f"  Size:  {stats['total_mb']} MB")


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_pdf.py
```

```text
PDF storage statistics:
  Files: 0
  Size:  0.0 MB
```

```text
                 ArXiv API
                    │
                    ▼
              ArxivClient
                    │
              paper metadata
                    │
                    ▼
             PaperProcessor
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      PostgreSQL          processing_queue
          │                   │
          │                   ▼
          │              PDFDownloader
          │                   │
          │                   ▼
          │              data/pdfs/
          │
          ▼
       papers
```


# Create `src/paper_processor.py`

```python
import logging
import re
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Dict, List

import psycopg2
from psycopg2.extras import RealDictCursor

from config.settings import (
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_USER,
    DB_PASSWORD,
)

from src.arxiv_client import ArxivClient, ArxivPaper
from src.pdf_processor import PDFDownloader


logger = logging.getLogger(__name__)


class PaperProcessor:
    """Orchestrates the complete paper processing pipeline."""

    def __init__(
        self,
        db_config: dict,
        arxiv_client: ArxivClient,
        pdf_downloader: PDFDownloader,
        max_workers: int = 4,
    ):
        self.db_config = db_config
        self.arxiv_client = arxiv_client
        self.pdf_downloader = pdf_downloader
        self.max_workers = max_workers

    def _get_connection(self):
        """Create a new PostgreSQL connection."""

        return psycopg2.connect(
            host=self.db_config["host"],
            port=self.db_config["port"],
            dbname=self.db_config["dbname"],
            user=self.db_config["user"],
            password=self.db_config["password"],
        )

    @staticmethod
    def _normalize_author_name(name: str) -> str:
        """Create a consistent version of an author name."""

        return re.sub(
            r"\s+",
            " ",
            name.strip().lower(),
        )

    def process_papers_batch(
        self,
        query: str,
        max_papers: int = 100,
        skip_existing: bool = True,
    ) -> dict:
        """Process a batch of papers from an ArXiv search query."""

        stats = {
            "found": 0,
            "new": 0,
            "existing": 0,
            "queued": 0,
            "downloaded": 0,
            "failed": 0,
        }

        papers: List[ArxivPaper] = []

        logger.info(
            "Searching ArXiv for: %s",
            query,
        )

        for paper in self.arxiv_client.search_papers(
            query=query,
            max_results=max_papers,
        ):
            papers.append(paper)

        stats["found"] = len(papers)

        if not papers:
            logger.info("No papers found.")
            return stats

        conn = self._get_connection()

        try:
            with conn.cursor(
                cursor_factory=RealDictCursor
            ) as cursor:

                queue_items = []

                for paper in papers:

                    cursor.execute(
                        """
                        SELECT id
                        FROM papers
                        WHERE arxiv_id = %s
                        """,
                        (paper.arxiv_id,),
                    )

                    existing = cursor.fetchone()

                    if existing and skip_existing:
                        stats["existing"] += 1
                        continue

                    paper_id = self._upsert_paper(
                        cursor,
                        paper,
                    )

                    stats["new"] += 1

                    cursor.execute(
                        """
                        INSERT INTO processing_queue
                            (paper_id, operation, status, priority)
                        VALUES
                            (%s, %s, %s, %s)
                        RETURNING id, paper_id, operation
                        """,
                        (
                            paper_id,
                            "download",
                            "pending",
                            10,
                        ),
                    )

                    queue_items.append(cursor.fetchone())
                    stats["queued"] += 1

                conn.commit()

        except Exception:
            conn.rollback()
            raise

        finally:
            conn.close()

        if queue_items:
            self._process_queue(
                queue_items,
                stats,
            )

        return stats

    def _upsert_paper(
        self,
        cursor,
        paper: ArxivPaper,
    ) -> int:
        """Insert or update paper metadata and return database ID."""

        cursor.execute(
            """
            INSERT INTO papers (
                arxiv_id,
                title,
                abstract,
                authors,
                categories,
                primary_category,
                published_date,
                updated_date,
                pdf_url,
                comment,
                journal_ref,
                doi
            )
            VALUES (
                %s, %s, %s, %s, %s, %s,
                %s, %s, %s, %s, %s, %s
            )
            ON CONFLICT (arxiv_id)
            DO UPDATE SET
                title = EXCLUDED.title,
                abstract = EXCLUDED.abstract,
                authors = EXCLUDED.authors,
                categories = EXCLUDED.categories,
                primary_category = EXCLUDED.primary_category,
                published_date = EXCLUDED.published_date,
                updated_date = EXCLUDED.updated_date,
                pdf_url = EXCLUDED.pdf_url,
                comment = EXCLUDED.comment,
                journal_ref = EXCLUDED.journal_ref,
                doi = EXCLUDED.doi,
                updated_at = CURRENT_TIMESTAMP
            RETURNING id
            """,
            (
                paper.arxiv_id,
                paper.title,
                paper.abstract,
                paper.authors,
                paper.categories,
                paper.primary_category,
                paper.published_date.date(),
                paper.updated_date.date(),
                paper.pdf_url,
                paper.comment,
                paper.journal_ref,
                paper.doi,
            ),
        )

        paper_db_id = cursor.fetchone()["id"]

        # Keep the relational author tables synchronized.
        for position, author_name in enumerate(
            paper.authors
        ):
            normalized_name = self._normalize_author_name(
                author_name
            )

            cursor.execute(
                """
                INSERT INTO authors (
                    name,
                    normalized_name
                )
                VALUES (%s, %s)
                ON CONFLICT (normalized_name)
                DO UPDATE SET name = EXCLUDED.name
                RETURNING id
                """,
                (
                    author_name,
                    normalized_name,
                ),
            )

            author_id = cursor.fetchone()["id"]

            cursor.execute(
                """
                INSERT INTO paper_authors (
                    paper_id,
                    author_id,
                    author_position
                )
                VALUES (%s, %s, %s)
                ON CONFLICT (paper_id, author_id)
                DO UPDATE SET
                    author_position = EXCLUDED.author_position
                """,
                (
                    paper_db_id,
                    author_id,
                    position,
                ),
            )

        return paper_db_id

    def _process_queue(
        self,
        queue_items: List[dict],
        stats: dict,
    ):
        """Process queue items using parallel workers."""

        logger.info(
            "Processing %d queue items with %d workers",
            len(queue_items),
            self.max_workers,
        )

        with ThreadPoolExecutor(
            max_workers=self.max_workers
        ) as executor:

            futures = {
                executor.submit(
                    self._process_download,
                    item,
                ): item
                for item in queue_items
            }

            for future in as_completed(futures):

                item = futures[future]

                try:
                    success = future.result()

                    if success:
                        stats["downloaded"] += 1
                    else:
                        stats["failed"] += 1

                except Exception:
                    stats["failed"] += 1

                    logger.exception(
                        "Queue item %s failed",
                        item["id"],
                    )

    def _process_download(
        self,
        item: dict,
    ) -> bool:
        """Process a single PDF download task."""

        conn = self._get_connection()

        try:
            with conn.cursor(
                cursor_factory=RealDictCursor
            ) as cursor:

                # Mark queue item as processing.
                cursor.execute(
                    """
                    UPDATE processing_queue
                    SET
                        status = %s,
                        started_at = CURRENT_TIMESTAMP
                    WHERE id = %s
                    """,
                    (
                        "processing",
                        item["id"],
                    ),
                )

                # Get paper information.
                cursor.execute(
                    """
                    SELECT
                        id,
                        arxiv_id,
                        pdf_url,
                        published_date
                    FROM papers
                    WHERE id = %s
                    """,
                    (item["paper_id"],),
                )

                paper = cursor.fetchone()

                if not paper:
                    raise ValueError(
                        f"Paper {item['paper_id']} not found"
                    )

                conn.commit()

                success, pdf_path, error = (
                    self.pdf_downloader.download_pdf(
                        pdf_url=paper["pdf_url"],
                        arxiv_id=paper["arxiv_id"],
                        published_date=paper[
                            "published_date"
                        ],
                    )
                )

                if success:

                    cursor.execute(
                        """
                        UPDATE papers
                        SET
                            pdf_downloaded = TRUE,
                            processing_error = NULL,
                            updated_at = CURRENT_TIMESTAMP
                        WHERE id = %s
                        """,
                        (paper["id"],),
                    )

                    cursor.execute(
                        """
                        UPDATE processing_queue
                        SET
                            status = %s,
                            completed_at = CURRENT_TIMESTAMP
                        WHERE id = %s
                        """,
                        (
                            "completed",
                            item["id"],
                        ),
                    )

                    conn.commit()

                    logger.info(
                        "Successfully downloaded %s",
                        paper["arxiv_id"],
                    )

                    return True

                cursor.execute(
                    """
                    UPDATE papers
                    SET
                        processing_error = %s,
                        updated_at = CURRENT_TIMESTAMP
                    WHERE id = %s
                    """,
                    (
                        error,
                        paper["id"],
                    ),
                )

                cursor.execute(
                    """
                    UPDATE processing_queue
                    SET
                        status = %s,
                        error_message = %s,
                        retry_count = retry_count + 1
                    WHERE id = %s
                    """,
                    (
                        "failed",
                        error,
                        item["id"],
                    ),
                )

                conn.commit()

                return False

        except Exception as error:

            conn.rollback()

            logger.exception(
                "Download task failed: %s",
                error,
            )

            try:
                with conn.cursor() as cursor:
                    cursor.execute(
                        """
                        UPDATE processing_queue
                        SET
                            status = %s,
                            error_message = %s,
                            retry_count = retry_count + 1
                        WHERE id = %s
                        """,
                        (
                            "failed",
                            str(error),
                            item["id"],
                        ),
                    )

                    conn.commit()

            except Exception:
                conn.rollback()

            return False

        finally:
            conn.close()


def create_default_processor(
    max_workers: int = 4,
) -> PaperProcessor:
    """Create a PaperProcessor using project settings."""

    db_config = {
        "host": DB_HOST,
        "port": DB_PORT,
        "dbname": DB_NAME,
        "user": DB_USER,
        "password": DB_PASSWORD,
    }

    return PaperProcessor(
        db_config=db_config,
        arxiv_client=ArxivClient(),
        pdf_downloader=PDFDownloader(),
        max_workers=max_workers,
    )
```

```text
ArxivClient
    ↓
paper metadata
```

```text
ArxivClient
    ↓
PaperProcessor
    ↓
papers table
    ↓
processing_queue
    ↓
PDFDownloader
```

```text
Paper A ──┐
Paper B ──┤
Paper C ──┼──→ processing_queue
Paper D ──┤
Paper E ──┘
```

```text
id   paper_id   operation   status
1    101        download    pending
2    102        download    pending
3    103        download    pending
4    104        download    pending
5    105        download    pending
```

```text
                 processing_queue
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          Worker 1  Worker 2  Worker 3
             │         │         │
             ▼         ▼         ▼
          PDF A      PDF B      PDF C
```

If Paper C fails:

```text
A → completed
B → completed
C → failed
D → completed
E → completed
```

**The whole pipeline doesn't stop.**

That's the main reason for having queue.

---

# Why `ThreadPoolExecutor`?

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
```

This allows us to have several downloads happening at once.

```python
max_workers = 4
```

```text
Worker 1 → downloading paper A
Worker 2 → downloading paper B
Worker 3 → downloading paper C
Worker 4 → downloading paper D
```

# Why each worker gets its own database connection

```python
conn = self._get_connection()
```

```text
Thread 1 → PostgreSQL connection 1
Thread 2 → PostgreSQL connection 2
Thread 3 → PostgreSQL connection 3
Thread 4 → PostgreSQL connection 4
```

That's a much safer design for parallel processing.

---

```text
scripts/test_processor.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.paper_processor import create_default_processor


def main():
    processor = create_default_processor(
        max_workers=2,
    )

    stats = processor.process_papers_batch(
        query="cat:cs.LG",
        max_papers=3,
    )

    print("\nProcessing results:")
    print(f"  Found:      {stats['found']}")
    print(f"  New:        {stats['new']}")
    print(f"  Existing:   {stats['existing']}")
    print(f"  Queued:     {stats['queued']}")
    print(f"  Downloaded: {stats['downloaded']}")
    print(f"  Failed:     {stats['failed']}")


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_processor.py
```

For first test, **3 papers is enough**. 

We want to verify this chain first:

```text
ArXiv
  ↓
metadata
  ↓
papers table
  ↓
processing_queue
  ↓
PDF download
  ↓
data/pdfs/2024/ or data/pdfs/2025/
```

```text
ArXiv
  ↓
metadata
  ↓
PDFDownloader
  ↓
paper.pdf
```

Now we turn:

```text
paper.pdf
   ↓
PDFExtractor
   ↓
clean structured text
   ↓
TextChunker
   ↓
chunks
   ↓
later: embeddings
```

```bash
pip install nltk
```


# Create `src/pdf_extractor.py`

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

```python
raw_blocks = page.get_text("blocks", sort=True)
```

Instead of asking PyMuPDF:

> "Give me all the text."

we ask:

> "Give me the individual text blocks and their positions on the page."

```text
PDF page

┌────────────────────────────────────┐
│             TITLE                  │
│                                    │
│ ┌────────────┐   ┌────────────┐    │
│ │ Block A    │   │ Block D    │    │
│ │ Block B    │   │ Block E    │    │
│ │ Block C    │   │ Block F    │    │
│ └────────────┘   └────────────┘    │
│                                    │
│              5                     │
└────────────────────────────────────┘
```

PyMuPDF gives us coordinates approximately like:

```text
x0, y0, x1, y1, text
```

So we can determine that A/B/C belong to the left column and D/E/F belong to the right column.

---

# Why `has_columns` matters

Academic papers frequently look like:

```text
┌────────────────┬────────────────┐
│ Introduction   │ Some text      │
│ text text      │ text text      │
│ text text      │ text text      │
│ text text      │ text text      │
└────────────────┴────────────────┘
```

A naive extractor might produce:

```text
Introduction Some text text text text text...
```

when we really want:

```text
Introduction
left-column text...
right-column text...
```

```text
detect columns
      ↓
yes ─────→ group left/right
      ↓
      ↓
order each column vertically
```

For each page we return:

```python
ExtractedPage(
    page_num=1,
    text="...",
    blocks=[...],
    has_columns=True,
    has_math=False,
    has_tables=False,
    confidence=0.95,
)
```

```text
Page 1
 ├── text
 ├── blocks
 ├── columns = yes
 ├── math = yes
 ├── tables = no
 └── confidence = 0.94
```

After extraction we have a large string:

```text
The entire paper...
Introduction...
Methods...
Results...
Conclusion...
```

But an embedding model shouldn't receive an entire 15-page paper as one vector.

```text
paper
  ↓
sentences
  ↓
chunks
  ↓
embeddings
```

```text
target ≈ 768 tokens
minimum = 256
maximum = 1024
overlap = 128
```

Create:
```text
src/text_chunker.py
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
        "tokenizers/punkt"
    )
except LookupError:
    nltk.download(
        "punkt",
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

 used:
```python
len(text.split())
```

instead of the actual embedding model tokenizer.

That's intentional **for now**.

This:

```python
"Transformers are powerful models"
```

might be roughly 5 words but a tokenizer might split it into a different number of tokens.

Later, when we build:

```text
TextChunker
     ↓
Embedding model
     ↓
all-MiniLM-L6-v2
```

we can make chunk sizing use the actual tokenizer.

For now, we're learning and building the pipeline without unnecessarily coupling everything together.

---
Create:
```text
scripts/test_extractor.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.pdf_extractor import PDFExtractor


def main():
    pdf_directory = (
        PROJECT_ROOT / "data" / "pdfs"
    )

    pdf_files = list(
        pdf_directory.rglob("*.pdf")
    )

    if not pdf_files:
        print("No PDF files found.")
        print(
            "Run the paper processor first "
            "to download some papers."
        )
        return

    pdf_path = pdf_files[0]

    print(
        f"Testing PDF:\n{pdf_path}\n"
    )

    extractor = PDFExtractor()

    result = extractor.extract_paper_text(
        str(pdf_path)
    )

    print("=" * 60)
    print("Extraction results")
    print("=" * 60)

    print(
        f"Pages:      {result['page_count']}"
    )

    print(
        f"Confidence: {result['confidence']}"
    )

    print(
        f"Sections:   {len(result['sections'])}"
    )

    print("\nDetected sections:")

    for section in result["sections"]:
        print(
            f"  - {section['name']}"
        )

    print("\nFirst 2000 characters:")
    print("-" * 60)
    print(
        result["text"][:2000]
    )


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_extractor.py
```

---
Create:
```text
scripts/test_chunker.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.pdf_extractor import PDFExtractor
from src.text_chunker import TextChunker


def main():
    pdf_directory = (
        PROJECT_ROOT / "data" / "pdfs"
    )

    pdf_files = list(
        pdf_directory.rglob("*.pdf")
    )

    if not pdf_files:
        print("No PDF files found.")
        return

    pdf_path = pdf_files[0]

    extractor = PDFExtractor()

    result = extractor.extract_paper_text(
        str(pdf_path)
    )

    chunker = TextChunker(
        target_chunk_size=768,
        min_chunk_size=256,
        max_chunk_size=1024,
        overlap_size=128,
    )

    chunks = chunker.chunk_paper(
        text=result["text"],
        sections=result["sections"],
    )

    print("=" * 60)
    print("Chunking results")
    print("=" * 60)

    print(
        f"PDF: {pdf_path.name}"
    )

    print(
        f"Chunks: {len(chunks)}"
    )

    for index, chunk in enumerate(
        chunks[:5]
    ):
        print("\n" + "=" * 60)
        print(
            f"Chunk {index}"
        )
        print(
            f"Section: {chunk['section_name']}"
        )
        print(
            f"Estimated tokens: "
            f"{chunk['token_count']}"
        )
        print("-" * 60)
        print(
            chunk["text"][:1000]
        )


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_chunker.py
```

```text
                   ArXiv
                     │
                     ▼
              ┌─────────────┐
              │ ArxivClient │
              └──────┬──────┘
                     │
                  metadata
                     │
                     ▼
              ┌─────────────┐
              │ PostgreSQL  │
              │   papers    │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │PDFDownloader│
              └──────┬──────┘
                     │
                     ▼
                  PDF file
                     │
                     ▼
              ┌─────────────┐
              │ PDFExtractor│  ← WE ARE HERE
              └──────┬──────┘
                     │
                clean text
                     │
                     ▼
              ┌─────────────┐
              │ TextChunker │  ← WE ARE HERE
              └──────┬──────┘
                     │
                 768-ish
                 token chunks
                     │
                     ▼
              ┌─────────────┐
              │  Embedding  │
              │   Model     │
              └──────┬──────┘
                     │
                384 numbers
                     │
                     ▼
              ┌─────────────┐
              │  pgvector   │
              └─────────────┘
```

---

| python scripts/test_chunker.py                                                                                                |
| ----------------------------------------------------------------------------------------------------------------------------- |
| warning: The `fitz` API is deprecated and will be removed in future. Use `import pymupdf` instead.                            |
| Traceback (most recent call last):                                                                                            |
| File "/home/furba/arxiv_search/scripts/test_chunker.py", line 77, in <module>                                                 |
| main()                                                                                                                        |
| File "/home/furba/arxiv_search/scripts/test_chunker.py", line 39, in main                                                     |
| chunks = chunker.chunk_paper(                                                                                                 |
| File "/home/furba/arxiv_search/src/text_chunker.py", line 75, in chunk_paper                                                  |
| section_chunks = self._chunk_text(                                                                                            |
| File "/home/furba/arxiv_search/src/text_chunker.py", line 99, in _chunk_text                                                  |
| sentences = sent_tokenize(                                                                                                    |
| File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/nltk/tokenize/__init__.py", line 119, in sent_tokenize        |
| tokenizer = _get_punkt_tokenizer(language)                                                                                    |
| File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/nltk/tokenize/__init__.py", line 105, in _get_punkt_tokenizer |
| return PunktTokenizer(language)                                                                                               |
| File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/nltk/tokenize/punkt.py", line 1795, in __init__               |
| self.load_lang(lang)                                                                                                          |
| File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/nltk/tokenize/punkt.py", line 1800, in load_lang              |
| lang_dir = find(f"tokenizers/punkt_tab/{lang}/")                                                                              |
| File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/nltk/data.py", line 877, in find                              |
| raise LookupError(resource_not_found)                                                                                         |
| LookupError:                                                                                                                  |
| Resource 'punkt_tab' not found.                                                                                               |
| Please use the NLTK Downloader to obtain the resource:                                                                        |
|                                                                                                                               |
| >>> import nltk                                                                                                               |
| >>> nltk.download('punkt_tab')                                                                                                |
|                                                                                                                               |
| For more information see: https://www.nltk.org/data.html                                                                      |
|                                                                                                                               |
| Attempted to load 'tokenizers/punkt_tab/english/'                                                                             |
|                                                                                                                               |
| Searched in:                                                                                                                  |
| - '/home/furba/nltk_data'                                                                                                     |
| - '/home/furba/arxiv_search/env/nltk_data'                                                                                    |
| - '/home/furba/arxiv_search/env/share/nltk_data'                                                                              |
| - '/home/furba/arxiv_search/env/lib/nltk_data'                                                                                |
| - '/usr/share/nltk_data'                                                                                                      |
| - '/usr/local/share/nltk_data'                                                                                                |
| - '/usr/lib/nltk_data'                                                                                                        |
| - '/usr/local/lib/nltk_data'                                                                                                  |
|                                                                                                                               |
| (env) furba@fu:~/arxiv_search$                                                                                                |

---

```text
Resource 'punkt_tab' not found.
```

Newer versions of NLTK use **`punkt_tab`** for `sent_tokenize()`. Our code only downloaded the older `punkt` resource.

## Fix it
```bash
python -c "import nltk; nltk.download('punkt_tab')"
```

```text
[nltk_data] Downloading package punkt_tab ...
[nltk_data]   Unzipping tokenizers/punkt_tab.zip.
```

```bash
python scripts/test_chunker.py
```

```text
src/text_chunker.py
```

Find:
```python
try:
    nltk.data.find(
        "tokenizers/punkt"
    )
except LookupError:
    nltk.download(
        "punkt",
        quiet=True,
    )
```

Replace it :

```python
try:
    nltk.data.find(
        "tokenizers/punkt_tab"
    )
except LookupError:
    nltk.download(
        "punkt_tab",
        quiet=True,
    )
```

### Why this happened

The chain is:

```text
TextChunker
    │
    ▼
sent_tokenize()
    │
    ▼
NLTK
    │
    ▼
punkt_tab
    │
    X
    │
    └── wasn't installed
```

Installing `nltk` installs the **Python library**, but some NLTK models/data are downloaded separately.
```text
pip install nltk
        ↓
NLTK program installed

nltk.download("punkt_tab")
        ↓
sentence-tokenization data installed
```

```text
PDF
 ↓
PDFExtractor
 ↓
clean text
 ↓
TextChunker
 ↓
~768-token chunks
 ↓
EmbeddingGenerator
 ↓
384 numbers
 ↓
PostgreSQL + pgvector
```

# Create `src/embeddings.py`

```python
import logging
from typing import List

import numpy as np
import torch
from sentence_transformers import SentenceTransformer
from tqdm import tqdm

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

---

# Why `normalize_embeddings=True`?

Our PostgreSQL index is already:

```sql
embedding vector(384)
```

with:

```sql
vector_cosine_ops
```

Normalization makes the vectors have length 1:

```text
       vector
          │
          ▼
   normalize to length 1
          │
          ▼
    cosine similarity
```

---

# Why is the model a singleton?

```python
_instance = None
```

```python
def __new__(...)
```

This is preventing us from loading model multiple times.

Without it, we might accidentally do:

```text
EmbeddingGenerator()
       ↓
load 2 GB model

EmbeddingGenerator()
       ↓
load another model

EmbeddingGenerator()
       ↓
load another model
```

That wastes memory.

With the singleton:

```text
EmbeddingGenerator()
       ↓
load model
       ↓
same model reused
       ↑
       │
EmbeddingGenerator()
EmbeddingGenerator()
EmbeddingGenerator()
```

Create:
```text
scripts/test_embeddings.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.embeddings import EmbeddingGenerator


def main():

    generator = EmbeddingGenerator()

    texts = [
        "Transformers use self-attention mechanisms.",
        "PostgreSQL can store vector embeddings.",
        "Cats are domestic animals.",
    ]

    print("Generating embeddings...\n")

    embeddings = generator.generate_embeddings(
        texts,
        show_progress=True,
    )

    print("\nResults:")
    print(
        f"Shape: {embeddings.shape}"
    )

    print(
        f"Data type: {embeddings.dtype}"
    )

    print(
        f"Model dimension: "
        f"{generator.embedding_dimension}"
    )

    print("\nFirst vector:")
    print(
        embeddings[0]
    )

    query_embedding = (
        generator.generate_query_embedding(
            "How do transformer models work?"
        )
    )

    print(
        "\nQuery embedding shape:"
    )

    print(
        query_embedding.shape
    )


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_embeddings.py
```

```text
Generating embeddings...

Results:
Shape: (3, 384)
Data type: float32
Model dimension: 384

First vector:
[ 0.02 ... ]

Query embedding shape:
(384,)
```

create:
```text
src/embedding_pipeline.py
```

```python
import logging
import re
from typing import Dict, List

import numpy as np
import psycopg2
from psycopg2.extras import execute_batch

from config.settings import (
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_USER,
    DB_PASSWORD,
    EMBEDDING_BATCH_SIZE,
)

from src.embeddings import EmbeddingGenerator


logger = logging.getLogger(__name__)


class EmbeddingPipeline:
    """Generate and store embeddings for paper chunks."""

    def __init__(
        self,
        db_config: dict,
        embedding_generator: EmbeddingGenerator,
        batch_size: int = EMBEDDING_BATCH_SIZE,
    ):
        self.db_config = db_config
        self.embedding_generator = (
            embedding_generator
        )
        self.batch_size = batch_size

    def _get_connection(self):
        """Create a PostgreSQL connection."""

        return psycopg2.connect(
            host=self.db_config["host"],
            port=self.db_config["port"],
            dbname=self.db_config["dbname"],
            user=self.db_config["user"],
            password=self.db_config["password"],
        )

    def process_paper(
        self,
        paper_id: int,
        chunks: List[Dict],
    ) -> Dict[str, object]:
        """Generate and store embeddings for one paper."""

        if not chunks:
            return {
                "paper_id": paper_id,
                "chunks": 0,
                "embedded": 0,
                "status": "empty",
            }

        texts = [
            chunk["text"]
            for chunk in chunks
        ]

        logger.info(
            "Generating embeddings for paper %d (%d chunks)",
            paper_id,
            len(chunks),
        )

        embeddings = (
            self.embedding_generator.generate_embeddings(
                texts,
                show_progress=True,
            )
        )

        self._store_chunks_with_embeddings(
            paper_id,
            chunks,
            embeddings,
        )

        return {
            "paper_id": paper_id,
            "chunks": len(chunks),
            "embedded": len(embeddings),
            "status": "completed",
        }

    def _store_chunks_with_embeddings(
        self,
        paper_id: int,
        chunks: List[Dict],
        embeddings: np.ndarray,
    ):
        """Store chunks and embeddings in PostgreSQL."""

        if len(chunks) != len(embeddings):
            raise ValueError(
                "Number of chunks does not match "
                "number of embeddings"
            )

        conn = self._get_connection()

        try:
            with conn.cursor() as cursor:

                # Remove old chunks for this paper.
                # This makes re-processing safe.
                cursor.execute(
                    """
                    DELETE FROM paper_chunks
                    WHERE paper_id = %s
                    """,
                    (paper_id,),
                )

                rows = []

                for index, (
                    chunk,
                    embedding,
                ) in enumerate(
                    zip(
                        chunks,
                        embeddings,
                    )
                ):

                    text = chunk["text"]

                    rows.append(
                        (
                            paper_id,
                            index,
                            text,
                            chunk.get(
                                "token_count"
                            ),
                            embedding.tolist(),
                            chunk.get(
                                "section_name"
                            ),
                            chunk.get(
                                "page_number"
                            ),
                            chunk.get(
                                "char_start"
                            ),
                            chunk.get(
                                "char_end"
                            ),
                            self._detect_math(
                                text
                            ),
                            self._detect_code(
                                text
                            ),
                            self._detect_references(
                                text
                            ),
                        )
                    )

                execute_batch(
                    cursor,
                    """
                    INSERT INTO paper_chunks (
                        paper_id,
                        chunk_index,
                        chunk_text,
                        chunk_tokens,
                        embedding,
                        section_name,
                        page_number,
                        char_start,
                        char_end,
                        has_math,
                        has_code,
                        has_references
                    )
                    VALUES (
                        %s, %s, %s, %s, %s,
                        %s, %s, %s, %s, %s,
                        %s, %s
                    )
                    """,
                    rows,
                    page_size=self.batch_size,
                )

                cursor.execute(
                    """
                    UPDATE papers
                    SET
                        embedding_generated = TRUE,
                        pdf_processed = TRUE,
                        processing_error = NULL,
                        updated_at = CURRENT_TIMESTAMP
                    WHERE id = %s
                    """,
                    (paper_id,),
                )

            conn.commit()

        except Exception:
            conn.rollback()
            raise

        finally:
            conn.close()

    def _detect_math(
        self,
        text: str,
    ) -> bool:
        """Detect likely mathematical content."""

        patterns = [
            r"\\frac",
            r"\\sum",
            r"\\int",
            r"\\alpha",
            r"\\beta",
            r"\\theta",
            r"∑",
            r"∫",
            r"≤",
            r"≥",
            r"≈",
            r"\bEquation\s+\d+",
        ]

        return any(
            re.search(
                pattern,
                text,
                flags=re.IGNORECASE,
            )
            for pattern in patterns
        )

    def _detect_code(
        self,
        text: str,
    ) -> bool:
        """Detect likely source code."""

        patterns = [
            r"\bdef\s+\w+\(",
            r"\bclass\s+\w+",
            r"\bimport\s+\w+",
            r"\bfrom\s+\w+\s+import\b",
            r"\bSELECT\s+.+\s+FROM\b",
            r"\bfor\s+\w+\s+in\s+",
            r"```",
        ]

        return any(
            re.search(
                pattern,
                text,
                flags=re.IGNORECASE,
            )
            for pattern in patterns
        )

    def _detect_references(
        self,
        text: str,
    ) -> bool:
        """Detect likely academic references."""

        patterns = [
            r"\[\d+\]",
            r"\(\w+\s+et al\.,?\s+\d{4}\)",
            r"\bdoi:\s*10\.",
        ]

        return any(
            re.search(
                pattern,
                text,
                flags=re.IGNORECASE,
            )
            for pattern in patterns
        )

    def process_pending_papers(
        self,
        limit: int = 10,
    ) -> Dict[str, object]:
        """Process papers that still need embeddings."""

        conn = self._get_connection()

        processed = 0
        failed = 0

        try:
            with conn.cursor() as cursor:

                cursor.execute(
                    """
                    SELECT id
                    FROM papers
                    WHERE pdf_downloaded = TRUE
                      AND embedding_generated = FALSE
                    ORDER BY id
                    LIMIT %s
                    """,
                    (limit,),
                )

                paper_ids = [
                    row[0]
                    for row in cursor.fetchall()
                ]

        finally:
            conn.close()

        for paper_id in paper_ids:

            try:
                # Import here to avoid creating
                # unnecessary dependencies at module load.
                from src.pdf_extractor import (
                    PDFExtractor
                )
                from src.text_chunker import (
                    TextChunker
                )

                conn = self._get_connection()

                try:
                    with conn.cursor() as cursor:

                        cursor.execute(
                            """
                            SELECT
                                arxiv_id,
                                published_date
                            FROM papers
                            WHERE id = %s
                            """,
                            (paper_id,),
                        )

                        paper = cursor.fetchone()

                finally:
                    conn.close()

                if not paper:
                    continue

                arxiv_id = paper[0]
                published_date = paper[1]

                pdf_path = (
                    self._find_pdf(
                        arxiv_id,
                        published_date.year,
                    )
                )

                if not pdf_path:
                    raise FileNotFoundError(
                        f"PDF not found for "
                        f"{arxiv_id}"
                    )

                extractor = PDFExtractor()

                extracted = (
                    extractor.extract_paper_text(
                        str(pdf_path)
                    )
                )

                chunker = TextChunker()

                chunks = chunker.chunk_paper(
                    text=extracted["text"],
                    sections=extracted["sections"],
                )

                result = self.process_paper(
                    paper_id,
                    chunks,
                )

                processed += 1

                logger.info(
                    "Processed paper %s: %s",
                    arxiv_id,
                    result,
                )

            except Exception as error:

                failed += 1

                logger.exception(
                    "Failed to process paper %d: %s",
                    paper_id,
                    error,
                )

                self._record_error(
                    paper_id,
                    str(error),
                )

        return {
            "requested": limit,
            "found": len(paper_ids),
            "processed": processed,
            "failed": failed,
        }

    def _find_pdf(
        self,
        arxiv_id: str,
        year: int,
    ):
        """Find a downloaded PDF."""

        from config.settings import PDF_STORAGE_PATH

        safe_id = arxiv_id.replace(
            "/",
            "_",
        )

        path = (
            PDF_STORAGE_PATH
            / str(year)
            / f"{safe_id}.pdf"
        )

        if path.exists():
            return path

        return None

    def _record_error(
        self,
        paper_id: int,
        error: str,
    ):
        """Record processing error in papers table."""

        conn = self._get_connection()

        try:
            with conn.cursor() as cursor:

                cursor.execute(
                    """
                    UPDATE papers
                    SET
                        processing_error = %s,
                        updated_at = CURRENT_TIMESTAMP
                    WHERE id = %s
                    """,
                    (
                        error,
                        paper_id,
                    ),
                )

            conn.commit()

        finally:
            conn.close()


def create_default_pipeline():
    """Create an embedding pipeline using project settings."""

    db_config = {
        "host": DB_HOST,
        "port": DB_PORT,
        "dbname": DB_NAME,
        "user": DB_USER,
        "password": DB_PASSWORD,
    }

    generator = EmbeddingGenerator()

    return EmbeddingPipeline(
        db_config=db_config,
        embedding_generator=generator,
    )
```

---
Suppose our chunker produces:

```text
paper_id = 15

chunk 0 → Introduction...
chunk 1 → Transformer architecture...
chunk 2 → Experiments...
```

EmbeddingGenerator produces:

```text
chunk 0 → 384 numbers
chunk 1 → 384 numbers
chunk 2 → 384 numbers
```

Then we store:

```text
paper_chunks
┌────┬──────────┬─────────────┬──────────────────┐
│ id │ paper_id │ chunk_index │ embedding        │
├────┼──────────┼─────────────┼──────────────────┤
│ 1  │ 15       │ 0           │ vector(384)      │
│ 2  │ 15       │ 1           │ vector(384)      │
│ 3  │ 15       │ 2           │ vector(384)      │
└────┴──────────┴─────────────┴──────────────────┘
```

```text
Human-readable text
        ↓
"Attention is all you need..."
        ↓
Embedding model
        ↓
[0.023, -0.081, 0.145, ...]
        ↓
384-dimensional vector
        ↓
pgvector
```

---

# Why `execute_batch()`?

Instead of doing:

```text
INSERT
INSERT
INSERT
INSERT
INSERT
...
```

For 500 chunks:

```text
500 individual database operations
```

is less efficient than batching them.

```text
500 chunks
    ↓
generate embeddings in batches
    ↓
prepare rows
    ↓
batch INSERT
    ↓
PostgreSQL
```

```text
papers
   │
   │ 1
   │
   ├───────────────┐
   │               │
   │               │
   ▼               ▼
metadata       paper_chunks
                   │
                   ├── chunk_text
                   ├── section_name
                   ├── page_number
                   └── embedding vector(384)
```

Create:
```text
scripts/test_embedding_pipeline.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.embedding_pipeline import create_default_pipeline


def main():
    pipeline = create_default_pipeline()

    result = pipeline.process_pending_papers(
        limit=1
    )

    print("\nEmbedding pipeline results:")
    print("=" * 50)

    for key, value in result.items():
        print(f"{key}: {value}")


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_embedding_pipeline.py
```

The system should find a paper that has:

```text
pdf_downloaded = TRUE
embedding_generated = FALSE
```

Then:

```text
                    PostgreSQL
                        │
                        ▼
                 pending paper
                        │
                        ▼
                  PDF on disk
                        │
                        ▼
                 PDFExtractor
                        │
                        ▼
                    Text
                        │
                        ▼
                  TextChunker
                        │
                        ▼
                ~768-token chunks
                        │
                        ▼
             all-MiniLM-L6-v2
                        │
                        ▼
                  384D vectors
                        │
                        ▼
                  paper_chunks
```

If successful, you should see something along the lines of:

```text
Embedding pipeline results:
==================================================
requested: 1
found: 1
processed: 1
failed: 0
```

```bash
psql -d ragdb
```

```sql
SELECT
    id,
    paper_id,
    chunk_index,
    chunk_tokens,
    section_name,
    has_math,
    has_code,
    has_references
FROM paper_chunks
ORDER BY id
LIMIT 10;
```

And check embedding dimension:

```sql
SELECT
    chunk_index,
    vector_dims(embedding) AS dimensions
FROM paper_chunks
LIMIT 10;
```

```text
dimensions
----------
384
```

That will confirm that the entire path from **PDF → text → chunks → embedding → pgvector** is working.

---
## What we have now

```text
ArXiv paper
     │
     ▼
PDF
     │
     ▼
Extract text
     │
     ▼
Split into chunks
     │
     ▼
Embedding model
all-MiniLM-L6-v2
     │
     ▼
384-dimensional vector
     │
     ▼
PostgreSQL + pgvector
```

a chunk might be:

> "Transformers use self-attention mechanisms to model relationships between tokens."

embedding model turns that into:

```text
[0.021, -0.183, 0.044, ... 0.092]
             ↑
         384 numbers
```

Now the user asks:

> "How do transformer models understand relationships between words?"

```text
question vector
       │
       ▼
[0.018, -0.175, 0.051, ...]
       │
       │ cosine similarity
       ▼
paper chunk vectors
```

The chunks mathematically closest to question become search results.

---
# What does `SearchResult` do?

```python
@dataclass
class SearchResult:
    paper_id: int
    arxiv_id: str
    title: str
    abstract: str
    authors: List[str]
    score: float
    matched_chunks: List[Dict]
    published_date: str
    categories: List[str]
```

is simply a **container for one search result**.

Instead of returning a messy tuple such as:

```python
(123, "2501.12345", "Attention Is All You Need...", ...)
```

we can return:

```python
SearchResult(
    paper_id=123,
    arxiv_id="2501.12345",
    title="Some Transformer Paper",
    abstract="...",
    authors=["John Smith"],
    score=0.87,
    matched_chunks=[...],
    published_date="2025-01-20",
    categories=["cs.LG", "cs.AI"]
)
```

---

```python
def search(
    self,
    query: str,
    mode: SearchMode = SearchMode.HYBRID,
    limit: int = 10,
    filters: Optional[Dict] = None
) -> List[SearchResult]:
```

So eventually we should be able to do:

```python
results = engine.search(
    "How do transformers model long range dependencies?",
    mode=SearchMode.HYBRID,
    limit=10
)
```
or
```python
results = engine.search(
    "transformers",
    mode=SearchMode.VECTOR
)
```
or
```python
results = engine.search(
    "attention",
    mode=SearchMode.KEYWORD
)
```

And with filters:

```python
results = engine.search(
    "transformers",
    mode=SearchMode.HYBRID,
    filters={
        "categories": ["cs.LG"],
        "date_from": "2022-01-01",
        "date_to": "2026-12-31"
    }
)
```

---

# What `_build_filter_clause()` is doing

This part is important because it prevents us from writing SQL manually for every possible search.

Suppose:

```python
filters = {
    "categories": ["cs.LG"],
    "date_from": "2022-01-01"
}
```

We want SQL roughly like:

```sql
WHERE
    'cs.LG' = ANY(p.categories)
    AND p.published_date >= '2022-01-01'
```

But **we should not directly insert user values into SQL**.

Instead, psycopg2 parameters should be used:

```sql
WHERE
    %s = ANY(p.categories)
    AND p.published_date >= %s
```

with:

```python
[
    "cs.LG",
    "2022-01-01"
]
```

This is called **parameterized SQL** and is important for security.

---

# Vector search

This is where pgvector becomes important.

Your `paper_chunks.embedding` column is:

```sql
embedding vector(384)
```

And you created this index:

```sql
CREATE INDEX idx_chunks_embedding
ON paper_chunks
USING hnsw (embedding vector_cosine_ops);
```

So we can use PostgreSQL's cosine distance operator:

```sql
embedding <=> query_embedding
```

Important:

`<=>` is **cosine distance**, not cosine similarity.

Conceptually:

```text
cosine similarity

1.0  = extremely similar
0.0  = unrelated
-1.0 = opposite direction
```

Whereas distance behaves differently:

```text
0.0 = identical
larger = less similar
```

Therefore we can convert:

```sql
1 - (embedding <=> query_embedding)
```

into a similarity-like score.

For example:

```text
distance = 0.12

similarity = 1 - 0.12
           = 0.88
```

That's the value we can expose as:

```python
result.score
```

---

# Why search `paper_chunks`, not just `papers`

```text
papers
  │
  ├── metadata
  │
  └── paper_chunks
          │
          ├── chunk 0
          ├── chunk 1
          ├── chunk 2
          ├── chunk 3
          └── ...
```

The embedding belongs to the **chunk**:

```text
paper_chunks.embedding
```

not the whole paper.

Therefore vector search starts from:

```sql
paper_chunks
```

and joins back to:

```sql
papers
```

Something conceptually like:

```sql
SELECT
    p.id,
    p.arxiv_id,
    p.title,
    c.chunk_text,
    c.embedding <=> %s AS distance
FROM paper_chunks c
JOIN papers p
    ON p.id = c.paper_id
ORDER BY distance
LIMIT 10;
```

This lets us discover **which part of which paper** matches the question.

---

# Why `matched_chunks` exists

```text
Paper
│
├── Chunk 1: Introduction
├── Chunk 2: Transformer architecture
├── Chunk 3: Training procedure
├── Chunk 4: Experiments
└── Chunk 5: Conclusion
```

> "How does self-attention work?"

might match:

```text
Chunk 2
```

very strongly.

So our result can contain:

```python
matched_chunks=[
    {
        "chunk_index": 2,
        "text": "...self-attention...",
        "score": 0.91,
        "section": "Architecture",
        "page": 4
    }
]
```

That's extremely useful later when you build the **RAG** part.

Your LLM won't need the entire paper. You can give it the best matching chunks.

---
# Hybrid search

```text
                     Query
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
        Vector search      Keyword search
              │                 │
              ▼                 ▼
        vector score       keyword score
              │                 │
              └────────┬────────┘
                       ▼
                  normalization
                       │
                       ▼
                 weighted score
                       │
                       ▼
                    ranking
```

For example:

```text
Vector score:   0.91
Keyword score:  0.70

vector weight:  0.70
keyword weight: 0.30
```

Then:

```text
final_score =
    0.70 × vector_score +
    0.30 × keyword_score
```

So:

```text
0.70 × 0.91 + 0.30 × 0.70
= 0.637 + 0.210
= 0.847
```

The exact weighting is something we can tune later.

---

# `find_similar_papers()`

This is slightly different.

Instead of:

```text
user question
      ↓
embedding
      ↓
similar papers
```

we do:

```text
existing paper
      ↓
take its chunks/embedding
      ↓
vector search
      ↓
other similar papers
```

For example:

```text
Paper A:
"Efficient Transformer Architectures"

             ↓

       find_similar_papers()

             ↓

Paper B: Efficient Attention Mechanisms
Paper C: Sparse Transformers
Paper D: Long Context Transformers
```

This can eventually power features such as:

- "Similar papers"
    
- Related research
    
- Paper recommendations
    
- Research discovery
---

```bash
nano src/search.py
```

```python
import logging
from dataclasses import dataclass
from enum import Enum
from typing import List, Dict, Optional, Tuple

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
    """One paper returned by the search engine."""

    paper_id: int
    arxiv_id: str
    title: str
    abstract: str
    authors: List[str]
    score: float
    matched_chunks: List[Dict]
    published_date: str
    categories: List[str]


class PaperSearchEngine:
    """Search ArXiv papers using vector, keyword, or hybrid search."""

    def __init__(
        self,
        db_config: dict,
        embedding_generator: EmbeddingGenerator,
    ):
        self.db_config = db_config
        self.embedding_generator = embedding_generator

    def _get_connection(self):
        """Create a PostgreSQL database connection."""

        return psycopg2.connect(**self.db_config)

    def search(
        self,
        query: str,
        mode: SearchMode = SearchMode.VECTOR,
        limit: int = 10,
        filters: Optional[Dict] = None,
    ) -> List[SearchResult]:
        """Main search interface."""

        if not query or not query.strip():
            return []

        if limit <= 0:
            return []

        if mode == SearchMode.VECTOR:
            return self._vector_search(query, limit, filters)

        elif mode == SearchMode.KEYWORD:
            return self._keyword_search(query, limit, filters)

        elif mode == SearchMode.HYBRID:
            return self._hybrid_search(query, limit, filters)

        else:
            raise ValueError(f"Unknown search mode: {mode}")

    def _vector_search(
        self,
        query: str,
        limit: int,
        filters: Optional[Dict],
    ) -> List[SearchResult]:
        """Search papers using vector similarity."""

        # Convert user's question into a 384-dimensional vector.
        query_embedding = self.embedding_generator.generate_query_embedding(query)

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

                1 - (c.embedding <=> %s::vector) AS score,

                c.id AS chunk_id,
                c.chunk_index,
                c.chunk_text,
                c.section_name,
                c.page_number

            FROM paper_chunks c

            JOIN papers p
                ON p.id = c.paper_id

            WHERE c.embedding IS NOT NULL
            {filter_sql}

            ORDER BY c.embedding <=> %s::vector

            LIMIT %s
        """

        params = [
            query_embedding.tolist(),
            *filter_params,
            query_embedding.tolist(),
            limit,
        ]

        with self._get_connection() as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                cursor.execute(sql, params)
                rows = cursor.fetchall()

        return self._rows_to_results(rows)

    def _keyword_search(
        self,
        query: str,
        limit: int,
        filters: Optional[Dict],
    ) -> List[SearchResult]:
        """Keyword search using PostgreSQL trigram similarity."""

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

            WHERE
                c.chunk_text % %s
                {filter_sql}

            ORDER BY similarity(c.chunk_text, %s) DESC

            LIMIT %s
        """

        params = [
            query,
            query,
            *filter_params,
            query,
            limit,
        ]

        with self._get_connection() as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                cursor.execute(sql, params)
                rows = cursor.fetchall()

        return self._rows_to_results(rows)

    def _hybrid_search(
        self,
        query: str,
        limit: int,
        filters: Optional[Dict],
    ) -> List[SearchResult]:
        """Combine vector and keyword search."""

        # Get more candidates from each search method.
        candidate_limit = max(limit * 3, 20)

        vector_results = self._vector_search(
            query,
            candidate_limit,
            filters,
        )

        keyword_results = self._keyword_search(
            query,
            candidate_limit,
            filters,
        )

        return self._combine_results(
            vector_results,
            keyword_results,
            vector_weight=0.7,
            keyword_weight=0.3,
        )[:limit]

    def _build_filter_clause(
        self,
        filters: Optional[Dict],
    ) -> Tuple[str, List]:
        """Build SQL WHERE conditions safely."""

        if not filters:
            return "", []

        conditions = []
        params = []

        if filters.get("categories"):
            conditions.append("%s = ANY(p.categories)")
            params.append(filters["categories"][0])

        if filters.get("date_from"):
            conditions.append("p.published_date >= %s")
            params.append(filters["date_from"])

        if filters.get("date_to"):
            conditions.append("p.published_date <= %s")
            params.append(filters["date_to"])

        if filters.get("authors"):
            conditions.append("%s = ANY(p.authors)")
            params.append(filters["authors"][0])

        if not conditions:
            return "", []

        return "AND " + " AND ".join(conditions), params

    def _combine_results(
        self,
        vector_results: List[SearchResult],
        keyword_results: List[SearchResult],
        vector_weight: float,
        keyword_weight: float,
    ) -> List[SearchResult]:
        """Combine vector and keyword results."""

        combined = {}

        for result in vector_results:
            combined[result.paper_id] = {
                "result": result,
                "vector_score": result.score,
                "keyword_score": 0.0,
            }

        for result in keyword_results:
            if result.paper_id not in combined:
                combined[result.paper_id] = {
                    "result": result,
                    "vector_score": 0.0,
                    "keyword_score": result.score,
                }
            else:
                combined[result.paper_id]["keyword_score"] = result.score

        results = []

        for item in combined.values():

            result = item["result"]

            final_score = (
                item["vector_score"] * vector_weight
                + item["keyword_score"] * keyword_weight
            )

            result.score = final_score

            results.append(result)

        results.sort(
            key=lambda result: result.score,
            reverse=True,
        )

        return results

    def _rows_to_results(self, rows) -> List[SearchResult]:
        """Convert database rows into SearchResult objects."""

        results = []

        for row in rows:

            matched_chunk = {
                "chunk_id": row["chunk_id"],
                "chunk_index": row["chunk_index"],
                "text": row["chunk_text"],
                "section_name": row["section_name"],
                "page_number": row["page_number"],
                "score": float(row["score"]),
            }

            results.append(
                SearchResult(
                    paper_id=row["paper_id"],
                    arxiv_id=row["arxiv_id"],
                    title=row["title"],
                    abstract=row["abstract"] or "",
                    authors=row["authors"] or [],
                    score=float(row["score"]),
                    matched_chunks=[matched_chunk],
                    published_date=str(row["published_date"]),
                    categories=row["categories"] or [],
                )
            )

        return results

    def find_similar_papers(
        self,
        paper_id: int,
        limit: int = 10,
    ) -> List[SearchResult]:
        """Find papers similar to an existing paper."""

        sql = """
            SELECT
                c.embedding
            FROM paper_chunks c
            WHERE c.paper_id = %s
              AND c.embedding IS NOT NULL
            ORDER BY c.chunk_index
            LIMIT 1
        """

        with self._get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute(sql, (paper_id,))
                row = cursor.fetchone()

        if not row:
            return []

        paper_embedding = row[0]

        sql = """
            SELECT
                p.id AS paper_id,
                p.arxiv_id,
                p.title,
                p.abstract,
                p.authors,
                p.published_date,
                p.categories,

                1 - (c.embedding <=> %s::vector) AS score,

                c.id AS chunk_id,
                c.chunk_index,
                c.chunk_text,
                c.section_name,
                c.page_number

            FROM paper_chunks c

            JOIN papers p
                ON p.id = c.paper_id

            WHERE
                c.embedding IS NOT NULL
                AND p.id != %s

            ORDER BY c.embedding <=> %s::vector

            LIMIT %s
        """

        params = [
            paper_embedding,
            paper_id,
            paper_embedding,
            limit,
        ]

        with self._get_connection() as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:
                cursor.execute(sql, params)
                rows = cursor.fetchall()

        return self._rows_to_results(rows)
```

---

```bash
nano scripts/test_search.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from config.settings import (
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_USER,
    DB_PASSWORD,
)

from src.embeddings import EmbeddingGenerator
from src.search import PaperSearchEngine, SearchMode


def main():

    db_config = {
        "host": DB_HOST,
        "port": DB_PORT,
        "dbname": DB_NAME,
        "user": DB_USER,
        "password": DB_PASSWORD,
    }

    embedding_generator = EmbeddingGenerator()

    search_engine = PaperSearchEngine(
        db_config=db_config,
        embedding_generator=embedding_generator,
    )

    query = "How do transformer models use attention?"

    print("\nSearching...")
    print("=" * 70)
    print(f"Query: {query}")

    results = search_engine.search(
        query=query,
        mode=SearchMode.VECTOR,
        limit=5,
    )

    print("\nResults:")
    print("=" * 70)

    for i, result in enumerate(results, start=1):

        print(f"\n{i}. {result.title}")
        print(f"   ArXiv ID: {result.arxiv_id}")
        print(f"   Score:    {result.score:.4f}")
        print(f"   Authors:  {', '.join(result.authors[:3])}")

        if result.matched_chunks:
            chunk = result.matched_chunks[0]

            print(f"   Section:  {chunk['section_name']}")
            print(f"   Page:     {chunk['page_number']}")

            text = chunk["text"].replace("\n", " ")

            print(f"   Match:    {text[:300]}...")


if __name__ == "__main__":
    main()
```

```bash
python scripts/test_search.py
```

You should see something similar to:

```text
Searching...
======================================================================
Query: How do transformer models use attention?

Results:
======================================================================

1. Attention Is All You Need
   ArXiv ID: 1706.03762
   Score:    0.8123
   Authors:  Ashish Vaswani, Noam Shazeer, Niki Parmar
   Section:  Introduction
   Page:     1
   Match:    The dominant sequence transduction models...
```

The exact papers and scores will obviously depend on which papers you've downloaded.

---
```bash
touch scripts/test_search.py
python scripts/test_search.py
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|█████████████████████████████████████████| 103/103 [00:00<00:00, 3887.28it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================
Query: How do transformer models use attention?

Results:
======================================================================
python scripts/test_search.py
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|████████████████████████████████████████| 103/103 [00:00<00:00, 10243.84it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================
Query: What does a LoRA-generating hypernetwork produce as output on a user's device?

Results:
======================================================================
(env) furba@fu:~/arxiv_search$
```

```bash
psql -d ragdb
```

Then:

```sql
SELECT COUNT(*) FROM papers;
```

and:

```sql
SELECT COUNT(*) FROM paper_chunks;
```

and:

```sql
SELECT COUNT(*) FROM paper_chunks
WHERE embedding IS NOT NULL;
```

We want something like:

```text
 papers
-------
   10

 paper_chunks
-------------
   150

 embedding chunks
-----------------
   150
```

```sql
SELECT
    id,
    arxiv_id,
    title,
    pdf_downloaded,
    pdf_processed,
    embedding_generated
FROM papers
ORDER BY id;
```

And:

```sql
SELECT
    id,
    paper_id,
    chunk_index,
    section_name,
    vector_dims(embedding) AS dimensions
FROM paper_chunks
LIMIT 10;
```

You ideally want:

```text
 dimensions
------------
 384
 384
 384
 384
```

---

# Run these exact commands

You can do them all from the terminal without entering the PostgreSQL shell:

```bash
psql -d ragdb -c "SELECT COUNT(*) AS papers FROM papers;"
```

```bash
psql -d ragdb -c "SELECT COUNT(*) AS chunks FROM paper_chunks;"
```

```bash
psql -d ragdb -c "SELECT COUNT(*) AS embedded_chunks FROM paper_chunks WHERE embedding IS NOT NULL;"
```

Then:

```bash
psql -d ragdb -c "SELECT id, arxiv_id, title, pdf_downloaded, pdf_processed, embedding_generated FROM papers ORDER BY id;"
```

And:

```bash
psql -d ragdb -c "SELECT id, paper_id, chunk_index, section_name, vector_dims(embedding) AS dimensions FROM paper_chunks LIMIT 10;"
```

---

```bash
psql -d ragdb -c "SELECT id, arxiv_id, title, pdf_downloaded, pdf_processed, embedding_generated FROM papers ORDER BY id;"
psql -d ragdb -c "SELECT id, paper_id, chunk_index, section_name, vector_dims(embedding) AS dimensions FROM paper_chunks LIMIT 10;"
```

```         
furba@fu:~$ psql -d ragdb -c "SELECT COUNT(*) AS papers FROM papers;"
 papers
--------
      3
(1 row)

furba@fu:~$ psql -d ragdb -c "SELECT COUNT(*) AS chunks FROM paper_chunks;"
 chunks
--------
      0
(1 row)

furba@fu:~$ psql -d ragdb -c "SELECT COUNT(*) AS embedded_chunks FROM paper_chunks WHERE embedding IS NOT NULL;"
 embedded_chunks
-----------------
               0
(1 row)

furba@fu:~$ psql -d ragdb -c "SELECT id, arxiv_id, title, pdf_downloaded, pdf_processed, embedding_generated FROM papers ORDER BY id;"
furba@fu:~$
furba@fu:~$ psql -d ragdb -c "SELECT id, paper_id, chunk_index, section_name, vector_dims(embedding) AS dimensions FROM paper_chunks LIMIT 10;"
 id | paper_id | chunk_index | section_name | dimensions
----+----------+-------------+--------------+------------
(0 rows)

furba@fu:~$
```

| id | arxiv_id | title | pdf_downloaded | pdf_processed | embedding_generated |
|----|----------|-------|----------------|---------------|---------------------|
| 1 | 2609.24985v1 | Critical-State RL: Diagnosing Trainable States for Multi-Turn Tool Use | t | f | f |
| 2 | 2609.24983v1 | onPanda: Efficient Annotation of On-Policy Alignment Data for LLMs and Agents via Token-Level Correction | t | f | f |
| 3 | 2609.24979v1 | LoRA-generating hypernetworks for efficient on-device LLM generative personalization | t | f | f |

---

```text
papers
  ├── Paper 1  PDF downloaded ✓
  ├── Paper 2  PDF downloaded ✓
  └── Paper 3  PDF downloaded ✓

paper_chunks
  └── EMPTY ❌
```

```text
pdf_downloaded = t
pdf_processed = f
embedding_generated = f
```

That means the workflow stopped after downloading the PDFs:

```text
ArXiv
  ↓
Paper metadata       ✓
  ↓
PDF download         ✓
  ↓
PDF extraction       ← not done
  ↓
Chunking              ← not done
  ↓
Embedding             ← not done
  ↓
paper_chunks          ← EMPTY
```

---

```bash
touch scripts/test_embedding_pipeline.py
nano scripts/test_embedding_pipeline.py
```

```python
import sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.embedding_pipeline import create_default_pipeline


def main():
    pipeline = create_default_pipeline()

    result = pipeline.process_pending_papers(limit=3)

    print("\nEmbedding pipeline results:")
    print("=" * 50)

    for key, value in result.items():
        print(f"{key}: {value}")


if __name__ == "__main__":
    main()
```

---

```bash
python scripts/test_embedding_pipeline.py
```

This should find your three papers:

```text
1 | Critical-State RL...
2 | onPanda...
3 | LoRA-generating hypernetworks...
```

because they currently have:

```text
pdf_downloaded = true
embedding_generated = false
```

The pipeline should then:

```text
PDF
 ↓
PDFExtractor
 ↓
TextChunker
 ↓
EmbeddingGenerator
 ↓
384-dimensional vectors
 ↓
paper_chunks
```

---

```bash
nano scripts/test_embedding_pipeline.py
python scripts/test_embedding_pipeline.py
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|████████████████████████████████████████| 103/103 [00:00<00:00, 10892.65it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()
warning: The `fitz` API is deprecated and will be removed in future. Use `import pymupdf` instead.
Batches: 100%|███████████████████████████████████████████████████████| 1/1 [00:00<00:00,  2.59it/s]
Batches: 100%|███████████████████████████████████████████████████████| 1/1 [00:00<00:00,  7.02it/s]
Batches: 100%|███████████████████████████████████████████████████████| 1/1 [00:00<00:00,  4.92it/s]

Embedding pipeline results:
==================================================
requested: 3
found: 3
processed: 3
failed: 0
(env) furba@fu:~/arxiv_search$
```

Perfect. **The embedding pipeline is now working end-to-end.**

Your output confirms:

```text
requested: 3
found:     3
processed: 3
failed:    0
```

So the three papers were successfully processed.

The pipeline is now:

```text
                    ArXiv
                      │
                      ▼
                 Paper metadata
                      │
                      ▼
                    PDF ✓
                      │
                      ▼
                PDF extraction ✓
                      │
                      ▼
                  Chunking ✓
                      │
                      ▼
             all-MiniLM-L6-v2 ✓
                      │
                      ▼
             384-dimensional vectors
                      │
                      ▼
             PostgreSQL / pgvector ✓
```

## 1. Verify the database

Before testing search, let's make sure the vectors are really there.

Run:

```bash
psql -d ragdb -c "SELECT COUNT(*) AS chunks FROM paper_chunks;"
```

Then:

```bash
psql -d ragdb -c "SELECT COUNT(*) AS embedded_chunks FROM paper_chunks WHERE embedding IS NOT NULL;"
```

And finally:

```bash
psql -d ragdb -c "SELECT id, paper_id, chunk_index, section_name, vector_dims(embedding) AS dimensions FROM paper_chunks LIMIT 10;"
```

You should now get something similar to:

```text
 id | paper_id | chunk_index | section_name | dimensions
----+----------+-------------+--------------+-----------
  1 |    1     |      0      | Introduction |    384
  2 |    1     |      1      | Introduction |    384
  3 |    1     |      2      | Methods      |    384
 ...
```

The exact number of chunks doesn't matter right now.

What matters is:

```text
chunks > 0
embedded_chunks > 0
dimensions = 384
```

---

## 2. Then test similarity search again

Once those checks look correct, run:

```bash
python scripts/test_search.py
```

Use your actual question again:

```python
query = "What does a LoRA-generating hypernetwork produce as output on a user's device?"
```

This time we should get actual results.

For example:

```text
Results:
======================================================================

1. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.xxxx
   Authors: ...
   Section: ...
   Page: ...
   Match: ...
```

### If you get results

Excellent — then we've successfully implemented the first real piece of your **semantic search engine**.

### If you still get no results

Don't change the code yet. Paste the output of:

```bash
psql -d ragdb -c "SELECT COUNT(*) AS chunks FROM paper_chunks;"
```

and:

```bash
python scripts/test_search.py
```

and we'll inspect the SQL.

---

### About those warnings

These are harmless for now:

```text
HF Hub ... unauthenticated requests
```

This just means Hugging Face isn't using an authentication token.

And:

```text
get_sentence_embedding_dimension
```

is a deprecation warning.

And:

```text
The fitz API is deprecated
```

is also a warning.

**None of these prevented your pipeline from working.**

Let's verify the database first, then we'll run the similarity search.

```bash
python scripts/test_search.py
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|████████████████████████████████████████| 103/103 [00:00<00:00, 11328.82it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================
Query: What does a LoRA-generating hypernetwork produce as output on a user's device?

Results:
======================================================================

1. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.5470
   Authors:  Sean Augenstein, Li Ding, Jihwan Lee
   Section:  Introduction
   Page:     None
   Match:    Introduction  ∗Corresponding author: saugenst@google.com. †Google. ‡Formerly at Google, work done while at Google. Preprint. LoRA-generating hypernetworks for efficient  on-device LLM generative personalization  Sean Augenstein∗† Li Ding† Jihwan Lee‡ Keith Rush† Andrey Zhmoginov‡...

2. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.5142
   Authors:  Sean Augenstein, Li Ding, Jihwan Lee
   Section:  References
   Page:     None
   Match:    Interestingly, it performs much better at cross-entropy then the distillation-trained hypernetwork. E.2 Mismatching User Contexts  As a general check that the hypernetwork is actually making use of users’ contexts, we ran an experiment where at test time we mismatched the data so that every user had...

3. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.5117
   Authors:  Sean Augenstein, Li Ding, Jihwan Lee
   Section:  References
   Page:     None
   Match:    Bolded numbers indicate the best value in a column for a given experiment configuration (dataset and LoRA rank). In general, the hypernetwork-generated LoRAs exhibit the best ROUGE-1 score performance (or nearly so) in virtually all scenarios. We also consider token cross-entropy on evaluation examp...

4. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.5079
   Authors:  Sean Augenstein, Li Ding, Jihwan Lee
   Section:  Related Work
   Page:     None
   Match:    We leverage these public datasets in our experiments. LoRAs Reuse; User Embeddings Some previous work relates to sub-components of our hypernet- work architecture. LoraHub (Huang et al., 2024) customizes a LoRA via a combination of pre-existing LoRAs. EigenLoRAx (Kaushik et al., 2025) goes further a...

5. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   ArXiv ID: 2609.24979v1
   Score:    0.4566
   Authors:  Sean Augenstein, Li Ding, Jihwan Lee
   Section:  Results
   Page:     None
   Match:    Results  For a more detailed breakdown of results, see Tables 4, 5, and 6 in Appendix D. We also studied a few ablations, namely, changes in how the hypernetwork is changed, and swapping of context examples to validate that the hypernetwork is actually using context information. See Appendix E.  Int...
(env) furba@fu:~/arxiv_search$
```

Excellent. **Your semantic similarity search is now working.**

The important part is this:

```text
1. LoRA-generating hypernetworks for efficient on-device LLM generative personalization
   Score: 0.5470
```

Your query:

> What does a LoRA-generating hypernetwork produce as output on a user's device?

was converted into an embedding, PostgreSQL compared it against the stored chunk embeddings, and it found the correct paper.

---

# But we have one problem to fix

Notice your results:

```text
1. LoRA paper — Introduction — 0.5470
2. LoRA paper — References   — 0.5142
3. LoRA paper — References   — 0.5117
4. LoRA paper — Related Work — 0.5079
5. LoRA paper — Results      — 0.4566
```

We got **five chunks from the same paper**.

That's technically valid vector search, but it's not what we ultimately want.

The user usually wants:

```text
1. Paper A
2. Paper B
3. Paper C
4. Paper D
5. Paper E
```

with the **best matching chunks from each paper**.

Right now our SQL searches chunks:

```text
paper_chunks
     │
     ├── chunk 1 ── Paper A
     ├── chunk 2 ── Paper A
     ├── chunk 3 ── Paper A
     ├── chunk 4 ── Paper B
     └── ...
```

and simply does:

```sql
ORDER BY similarity
LIMIT 5
```

So PostgreSQL is allowed to return five chunks from the same paper.

---

# What we should build next

We should change the search architecture to:

```text
                 User query
                     │
                     ▼
             Query embedding
                     │
                     ▼
                pgvector
                     │
                     ▼
              Matching chunks
                     │
                     ▼
          Group by paper_id
                     │
                     ▼
       Best chunk for each paper
                     │
                     ▼
             Top N papers
```

For example:

```text
Query
 │
 ├── Paper 3, chunk 17 → 0.82
 ├── Paper 3, chunk 21 → 0.79
 ├── Paper 1, chunk 5  → 0.76
 ├── Paper 3, chunk 31 → 0.73
 └── Paper 2, chunk 8  → 0.71
```

Instead of returning those five chunks directly, we would produce:

```text
Paper 3 → 0.82
Paper 1 → 0.76
Paper 2 → 0.71
```

and keep the relevant chunks inside:

```python
matched_chunks
```

That's exactly why we created:

```python
matched_chunks: List[Dict]
```

in `SearchResult`.

---

# One more thing: your score

Your highest score is:

```text
0.5470
```

Don't worry about that number being "only" 0.547.

The score is based on:

```sql
1 - (embedding <=> query_embedding)
```

and the absolute value isn't a universal "percentage of relevance."

So:

```text
0.547 ≠ 54.7% relevant
```

It's a similarity measure useful primarily for **ranking**.

---

# Your system has now reached an important milestone

You have successfully built:

```text
                    ArXiv
                      │
                      ▼
                PDF Downloader
                      │
                      ▼
                PDF Extractor
                      │
                      ▼
                 Text Chunker
                      │
                      ▼
             all-MiniLM-L6-v2
                      │
                 384 dimensions
                      │
                      ▼
              PostgreSQL
                 pgvector
                      │
                      ▼
              HNSW index
                      │
                      ▼
              Vector Search
                      │
                      ▼
             Relevant chunks
```

So the **core semantic-search mechanism works**.

## Next change

I recommend we now fix **paper-level result grouping** before implementing keyword/hybrid search.

That gives us a cleaner foundation:

```text
Vector search
      ↓
paper-level ranking
      ↓
metadata filtering
      ↓
keyword search
      ↓
hybrid search
      ↓
RAG
```

The next code change should therefore be to `_vector_search()` and `_rows_to_results()` so that **one paper produces one `SearchResult`, containing its best matching chunks**.


Yes. Now that **semantic search is working**, we can build the interactive CLI on top of it.

One important adjustment: the book's imports assume everything is in one directory:

```python
from arxiv_client import ArxivClient
```

but your project has:

```text
arxiv_search/
├── src/
│   ├── arxiv_client.py
│   ├── search.py
│   ├── embeddings.py
│   └── ...
└── scripts/
```

So we'll use your actual project structure.

## 1. Install the CLI packages

From:

```bash
cd ~/arxiv_search
```

run:

```bash
pip install click tabulate
```

---

# 2. Create `scripts/cli.py`

Run:

```bash
touch scripts/cli.py
nano scripts/cli.py
```

Put this in it:

```python
#!/usr/bin/env python3

"""
ArXiv Paper Search System - Interactive CLI
"""

import sys
from pathlib import Path
import json
from datetime import datetime, timedelta

import click
import psycopg2
from tabulate import tabulate


# ---------------------------------------------------------
# Make project root importable
# ---------------------------------------------------------

PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))


# ---------------------------------------------------------
# Project imports
# ---------------------------------------------------------

from config.settings import (
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_USER,
    DB_PASSWORD,
)

from src.arxiv_client import ArxivClient
from src.embeddings import EmbeddingGenerator
from src.search import PaperSearchEngine, SearchMode


# ---------------------------------------------------------
# Database configuration
# ---------------------------------------------------------

DB_CONFIG = {
    "host": DB_HOST,
    "port": DB_PORT,
    "dbname": DB_NAME,
    "user": DB_USER,
    "password": DB_PASSWORD,
}


# ---------------------------------------------------------
# CLI setup
# ---------------------------------------------------------

@click.group()
@click.pass_context
def cli(ctx):
    """
    ArXiv Paper Search System - Personal Research Tool
    """

    ctx.ensure_object(dict)

    ctx.obj["db_config"] = DB_CONFIG

    # Load embedding model once.
    ctx.obj["embedding_generator"] = EmbeddingGenerator()

    # Create search engine.
    ctx.obj["search_engine"] = PaperSearchEngine(
        db_config=DB_CONFIG,
        embedding_generator=ctx.obj["embedding_generator"],
    )


# ---------------------------------------------------------
# SEARCH
# ---------------------------------------------------------

@cli.command()
@click.option(
    "--query",
    "-q",
    required=True,
    help="Search query",
)
@click.option(
    "--mode",
    "-m",
    type=click.Choice(
        ["vector", "hybrid", "keyword"],
        case_sensitive=False,
    ),
    default="hybrid",
    show_default=True,
    help="Search mode",
)
@click.option(
    "--limit",
    "-l",
    default=10,
    show_default=True,
    help="Number of results",
)
@click.option(
    "--authors",
    "-a",
    multiple=True,
    help="Filter by author",
)
@click.option(
    "--categories",
    "-c",
    multiple=True,
    help="Filter by ArXiv category",
)
@click.option(
    "--days",
    "-d",
    type=int,
    help="Papers from last N days",
)
@click.option(
    "--export",
    "-e",
    help="Export results to JSON file",
)
@click.pass_context
def search(
    ctx,
    query,
    mode,
    limit,
    authors,
    categories,
    days,
    export,
):
    """
    Search for papers.
    """

    click.echo()
    click.echo("Searching...")
    click.echo("=" * 70)

    # -----------------------------------------------------
    # Build filters
    # -----------------------------------------------------

    filters = {}

    if authors:
        filters["authors"] = list(authors)

    if categories:
        filters["categories"] = list(categories)

    if days is not None:
        date_from = (
            datetime.now() - timedelta(days=days)
        ).date()

        filters["date_from"] = str(date_from)

    # -----------------------------------------------------
    # Convert string to SearchMode
    # -----------------------------------------------------

    search_mode = SearchMode(mode.lower())

    # -----------------------------------------------------
    # Execute search
    # -----------------------------------------------------

    engine = ctx.obj["search_engine"]

    results = engine.search(
        query=query,
        mode=search_mode,
        limit=limit,
        filters=filters or None,
    )

    # -----------------------------------------------------
    # Display results
    # -----------------------------------------------------

    if not results:
        click.echo()
        click.echo("No results found.")
        return

    display_results(results)

    # -----------------------------------------------------
    # Export
    # -----------------------------------------------------

    if export:
        export_results(results, export)

        click.echo()
        click.echo(
            f"Results exported to: {export}"
        )

    # -----------------------------------------------------
    # Interactive exploration
    # -----------------------------------------------------

    explore_results(ctx, results)


# ---------------------------------------------------------
# DISPLAY RESULTS
# ---------------------------------------------------------

def display_results(results):
    """
    Display search results in a formatted table.
    """

    rows = []

    for index, result in enumerate(results, start=1):

        title = result.title

        if len(title) > 65:
            title = title[:62] + "..."

        authors = ", ".join(result.authors[:3])

        if len(result.authors) > 3:
            authors += " et al."

        rows.append(
            [
                index,
                title,
                result.arxiv_id,
                f"{result.score:.4f}",
                authors,
                result.published_date,
            ]
        )

    headers = [
        "#",
        "Title",
        "ArXiv ID",
        "Score",
        "Authors",
        "Published",
    ]

    click.echo()

    click.echo(
        tabulate(
            rows,
            headers=headers,
            tablefmt="rounded_outline",
        )
    )


# ---------------------------------------------------------
# INTERACTIVE EXPLORATION
# ---------------------------------------------------------

def explore_results(ctx, results):
    """
    Let the user explore search results.
    """

    while True:

        choice = click.prompt(
            "\nEnter result number to explore, "
            "'q' to quit",
            default="q",
        )

        if choice.lower() == "q":
            break

        try:
            index = int(choice) - 1

            if index < 0 or index >= len(results):
                click.echo("Invalid result number.")
                continue

        except ValueError:
            click.echo(
                "Please enter a number or 'q'."
            )
            continue

        result = results[index]

        while True:

            click.echo()
            click.echo("=" * 70)
            click.echo(f"Selected: {result.title}")
            click.echo("=" * 70)

            click.echo()
            click.echo("1. Show paper details")
            click.echo("2. Find similar papers")
            click.echo("3. Back")

            action = click.prompt(
                "Choose an option",
                type=click.Choice(
                    ["1", "2", "3"]
                ),
            )

            if action == "1":
                show_paper_details(result)

            elif action == "2":
                find_similar(ctx, result)

            elif action == "3":
                break


# ---------------------------------------------------------
# PAPER DETAILS
# ---------------------------------------------------------

def show_paper_details(result):
    """
    Show detailed information about a paper.
    """

    click.echo()
    click.echo("=" * 80)
    click.echo(result.title)
    click.echo("=" * 80)

    click.echo(f"\nArXiv ID: {result.arxiv_id}")

    click.echo(
        f"\nAuthors:\n"
        f"  {', '.join(result.authors)}"
    )

    click.echo(
        f"\nPublished: {result.published_date}"
    )

    click.echo(
        f"\nCategories:\n"
        f"  {', '.join(result.categories)}"
    )

    click.echo(
        f"\nSimilarity score: "
        f"{result.score:.4f}"
    )

    click.echo("\nAbstract:")
    click.echo(result.abstract)

    if result.matched_chunks:

        click.echo("\nMatched chunks:")
        click.echo("-" * 80)

        for index, chunk in enumerate(
            result.matched_chunks,
            start=1,
        ):

            click.echo(
                f"\nChunk {index}"
            )

            click.echo(
                f"Section: "
                f"{chunk.get('section_name')}"
            )

            click.echo(
                f"Page: "
                f"{chunk.get('page_number')}"
            )

            click.echo(
                f"Chunk score: "
                f"{chunk.get('score', 0):.4f}"
            )

            click.echo()

            click.echo(
                chunk.get("text", "")
            )


# ---------------------------------------------------------
# SIMILAR PAPERS
# ---------------------------------------------------------

def find_similar(ctx, result):
    """
    Find papers similar to the selected paper.
    """

    click.echo()
    click.echo(
        f"Finding papers similar to:"
    )

    click.echo(result.title)

    engine = ctx.obj["search_engine"]

    similar = engine.find_similar_papers(
        paper_id=result.paper_id,
        limit=5,
    )

    if not similar:
        click.echo(
            "\nNo similar papers found."
        )
        return

    click.echo()

    display_results(similar)


# ---------------------------------------------------------
# EXPORT
# ---------------------------------------------------------

def export_results(results, filename):
    """
    Export search results to JSON.
    """

    data = []

    for result in results:

        data.append(
            {
                "paper_id": result.paper_id,
                "arxiv_id": result.arxiv_id,
                "title": result.title,
                "abstract": result.abstract,
                "authors": result.authors,
                "score": result.score,
                "published_date": result.published_date,
                "categories": result.categories,
                "matched_chunks": result.matched_chunks,
            }
        )

    with open(
        filename,
        "w",
        encoding="utf-8",
    ) as file:

        json.dump(
            data,
            file,
            indent=2,
            ensure_ascii=False,
        )


# ---------------------------------------------------------
# FETCH
# ---------------------------------------------------------

@cli.command()
@click.option(
    "--categories",
    "-c",
    multiple=True,
    required=True,
    help="ArXiv categories to fetch",
)
@click.option(
    "--days",
    "-d",
    default=7,
    show_default=True,
    help="Fetch papers from last N days",
)
@click.option(
    "--max-papers",
    "-m",
    default=100,
    show_default=True,
    help="Maximum papers to fetch",
)
@click.pass_context
def fetch(
    ctx,
    categories,
    days,
    max_papers,
):
    """
    Fetch recent papers from ArXiv.
    """

    click.echo()
    click.echo("Fetching papers...")
    click.echo("=" * 70)

    client = ArxivClient()

    papers = client.search_recent_papers(
        categories=list(categories),
        days_back=days,
    )

    count = 0

    for paper in papers:

        if count >= max_papers:
            break

        click.echo(
            f"[{count + 1}] {paper.title}"
        )

        count += 1

    click.echo()
    click.echo(
        f"Found {count} papers."
    )

    click.echo(
        "\nNote: this command currently only "
        "fetches/displays ArXiv results."
    )


# ---------------------------------------------------------
# STATS
# ---------------------------------------------------------

@cli.command()
@click.pass_context
def stats(ctx):
    """
    Show database statistics.
    """

    conn = psycopg2.connect(
        **ctx.obj["db_config"]
    )

    try:

        with conn.cursor() as cursor:

            cursor.execute(
                "SELECT COUNT(*) FROM papers"
            )
            papers = cursor.fetchone()[0]

            cursor.execute(
                "SELECT COUNT(*) FROM paper_chunks"
            )
            chunks = cursor.fetchone()[0]

            cursor.execute(
                """
                SELECT COUNT(*)
                FROM paper_chunks
                WHERE embedding IS NOT NULL
                """
            )
            embeddings = cursor.fetchone()[0]

            cursor.execute(
                """
                SELECT COUNT(*)
                FROM papers
                WHERE pdf_downloaded = TRUE
                """
            )
            pdfs = cursor.fetchone()[0]

            cursor.execute(
                """
                SELECT COUNT(*)
                FROM papers
                WHERE embedding_generated = TRUE
                """
            )
            processed = cursor.fetchone()[0]

    finally:
        conn.close()

    click.echo()
    click.echo("Database Statistics")
    click.echo("=" * 50)

    click.echo(
        f"Papers:             {papers}"
    )

    click.echo(
        f"PDFs downloaded:    {pdfs}"
    )

    click.echo(
        f"Papers processed:   {processed}"
    )

    click.echo(
        f"Paper chunks:       {chunks}"
    )

    click.echo(
        f"Embeddings:         {embeddings}"
    )


# ---------------------------------------------------------
# MANAGE
# ---------------------------------------------------------

@cli.command()
@click.option(
    "--init",
    is_flag=True,
    help="Initialize database schema",
)
@click.option(
    "--rebuild-index",
    is_flag=True,
    help="Rebuild vector indexes",
)
@click.option(
    "--cleanup",
    is_flag=True,
    help="Clean up old data",
)
@click.pass_context
def manage(
    ctx,
    init,
    rebuild_index,
    cleanup,
):
    """
    Database management operations.
    """

    if not any(
        [init, rebuild_index, cleanup]
    ):
        click.echo(
            "Choose an operation:"
        )

        click.echo(
            "  --init"
        )

        click.echo(
            "  --rebuild-index"
        )

        click.echo(
            "  --cleanup"
        )

        return

    if init:

        click.echo(
            "Database initialization should "
            "be performed with:"
        )

        click.echo(
            "python scripts/setup_db.py"
        )

    if rebuild_index:

        conn = psycopg2.connect(
            **ctx.obj["db_config"]
        )

        try:

            with conn.cursor() as cursor:

                cursor.execute(
                    """
                    REINDEX INDEX
                    idx_chunks_embedding
                    """
                )

            conn.commit()

        finally:
            conn.close()

        click.echo(
            "Vector index rebuilt."
        )

    if cleanup:

        click.echo(
            "Cleanup operation selected."
        )

        click.echo(
            "No automatic cleanup performed yet."
        )


# ---------------------------------------------------------
# Main
# ---------------------------------------------------------

if __name__ == "__main__":
    cli()
```

Save:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 3. First test: show the CLI help

Don't search yet.

Run:

```bash
python scripts/cli.py --help
```

You should get something similar to:

```text
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

 ArXiv Paper Search System - Personal Research Tool

Commands:
  fetch    Fetch recent papers from ArXiv
  manage   Database management operations
  search   Search for papers
  stats    Show database statistics
```

---

# 4. Check your database through the CLI

Run:

```bash
python scripts/cli.py stats
```

Given what we just processed, you should see approximately:

```text
Database Statistics
==================================================
Papers:             3
PDFs downloaded:    3
Papers processed:   3
Paper chunks:       ...
Embeddings:         ...
```

The last two numbers should be **greater than zero**.

---

# 5. Now perform a semantic search

Run:

```bash
python scripts/cli.py search \
    --query "What does a LoRA-generating hypernetwork produce as output on a user's device?" \
    --mode vector \
    --limit 5
```

This gives us a much nicer interface than:

```bash
python scripts/test_search.py
```

You should get a table such as:

```text
╭────┬───────────────────────────────┬──────────────┬────────┬──────────────┬────────────╮
│ #  │ Title                         │ ArXiv ID     │ Score  │ Authors      │ Published  │
├────┼───────────────────────────────┼──────────────┼────────┼──────────────┼────────────┤
│ 1  │ LoRA-generating hypernetworks │ 2609.24979v1 │ 0.5470 │ Sean ...     │ 2026-09-.. │
╰────┴───────────────────────────────┴──────────────┴────────┴──────────────┴────────────╯
```

Then you'll get:

```text
Enter result number to explore, 'q' to quit:
```

Enter:

```text
1
```

Then:

```text
1. Show paper details
2. Find similar papers
3. Back
```

Choose:

```text
1
```

Now you can see the paper's:

- title
    
- ArXiv ID
    
- authors
    
- publication date
    
- categories
    
- similarity score
    
- abstract
    
- matching chunk
    

---

## 6. We have now added the UI layer

Your architecture is becoming:

```text
                    CLI
                     │
                     ▼
             PaperSearchEngine
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       VECTOR     KEYWORD    HYBRID
          │          │          │
          └──────────┼──────────┘
                     ▼
                 PostgreSQL
                     │
              ┌──────┴──────┐
              ▼             ▼
           pgvector       pg_trgm
```

And the CLI gives you:

```text
search
  │
  ├── vector
  ├── keyword
  └── hybrid

stats

fetch

manage
  ├── init
  ├── rebuild-index
  └── cleanup
```

### One caveat

The `fetch` command above intentionally **only searches and displays ArXiv papers**. Your existing `PaperProcessor` is the component that actually stores papers and downloads PDFs. We should connect those pieces next rather than duplicating the ingestion logic in the CLI.

For now, run these **two commands in order**:

```bash
python scripts/cli.py --help
```

then:

```bash
python scripts/cli.py stats
```

Paste the output. Then we'll test the interactive search itself.

---


```
bash
# Run the interactive search
python scripts/search_cli.py

# In the CLI:
# Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
# ======================================================================
#
# 1. Show paper details
# 2. Find similar papers
# 3. Back
# Choose an option (1, 2, 3): 2
#
# Finding papers similar to:
# LoRA-generating hypernetworks for efficient on-device LLM generative personalization
#
#
# ╭─────┬───────────────────────────────────────────────────────────────────┬──────────────┬─────────┬────────────────────────────────────────────────┬─────────────╮
# │   # │ Title                                                             │ ArXiv ID     │   Score │ Authors                                        │ Published   │
# ├─────┼───────────────────────────────────────────────────────────────────┼──────────────┼─────────┼────────────────────────────────────────────────┼─────────────┤
# │   1 │ Critical-State RL: Diagnosing Trainable States for Multi-Turn ... │ 2609.24985v1 │  0.9881 │ Zixiang Chen, Wenting Zhao, Zhepeng Cen et al. │ 2026-09-21  │
# │   2 │ Critical-State RL: Diagnosing Trainable States for Multi-Turn ... │ 2609.24985v1 │  0.2889 │ Zixiang Chen, Wenting Zhao, Zhepeng Cen et al. │ 2026-09-21  │
# │   3 │ Critical-State RL: Diagnosing Trainable States for Multi-Turn ... │ 2609.24985v1 │  0.2286 │ Zixiang Chen, Wenting Zhao, Zhepeng Cen et al. │ 2026-09-21  │
# │   4 │ Critical-State RL: Diagnosing Trainable States for Multi-Turn ... │ 2609.24985v1 │  0.2065 │ Zixiang Chen, Wenting Zhao, Zhepeng Cen et al. │ 2026-09-21  │
# │   5 │ Critical-State RL: Diagnosing Trainable States for Multi-Turn ... │ 2609.24985v1 │  0.2052 │ Zixiang Chen, Wenting Zhao, Zhepeng Cen et al. │ 2026-09-21  │
# ╰─────┴───────────────────────────────────────────────────────────────────┴──────────────┴─────────┴────────────────────────────────────────────────┴─────────────╯
#
# ======================================================================
# Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
# ======================================================================
#
# 1. Show paper details
# 2. Find similar papers
# 3. Back
# Choose an option (1, 2, 3): 3
#
# Enter result number to explore, 'q' to quit [q]: q
```

Yes. The main problem is that your table has **too many wide columns**—especially `Title` and `Authors`. The terminal is trying to display the entire paper metadata in one horizontal table.

For a CLI, I would make it much cleaner by:

- limiting the title width
    
- limiting authors to 2–3 names
    
- removing the `Published` column from the main table
    
- using a compact score
    
- showing full details only after selecting a paper
    
- using terminal width dynamically
    

For example:

```text
┌───┬──────────────────────────────────────┬──────────────┬────────┐
│ # │ Title                                │ ArXiv ID     │ Score  │
├───┼──────────────────────────────────────┼──────────────┼────────┤
│ 1 │ Critical-State RL: Diagnosing ...    │ 2609.24985v1 │ 0.9881 │
│ 2 │ onPanda: Efficient Annotation ...    │ 2609.24983v1 │ 0.4127 │
│ 3 │ LoRA-generating hypernetworks ...    │ 2609.24979v1 │ 0.3012 │
└───┴──────────────────────────────────────┴──────────────┴────────┘
```

Then when you select one:

```text
╭──────────────────────────────────────────────────────────────╮
│ LoRA-generating hypernetworks for efficient on-device LLM    │
│ generative personalization                                   │
╰──────────────────────────────────────────────────────────────╯

ArXiv:     2609.24979v1
Score:     0.5470
Published: 2026-09-21

Authors
  Sean Augenstein
  Li Ding
  Jihwan Lee

Categories
  cs.CL, cs.LG

Abstract
──────────────────────────────────────────────────────────────
...
```

### The important fix

Open:

```bash
nano src/search.py
```

The bigger issue is actually your **similar-paper search**. You're getting:

```text
Critical-State RL    0.9881
Critical-State RL    0.2889
Critical-State RL    0.2286
Critical-State RL    0.2065
Critical-State RL    0.2052
```

Those aren't really five similar papers. They are **five chunks from the same paper**.

So I'd fix **both the visual table and the search result grouping** now.

---

## 1. First fix the CLI table

Open:

```bash
nano scripts/cli.py
```

Find your `display_results()` function and replace it with this:

```python
def display_results(results):
    """Display search results in a compact terminal-friendly table."""

    if not results:
        click.echo("\nNo results found.")
        return

    rows = []

    for i, result in enumerate(results, start=1):
        title = result.title.replace("\n", " ").strip()

        # Keep the table narrow.
        if len(title) > 45:
            title = title[:42] + "..."

        authors = ", ".join(result.authors[:2])

        if len(result.authors) > 2:
            authors += " et al."

        if len(authors) > 28:
            authors = authors[:25] + "..."

        rows.append([
            i,
            title,
            result.arxiv_id,
            f"{result.score:.4f}",
        ])

    headers = [
        "#",
        "Title",
        "ArXiv ID",
        "Score",
    ]

    click.echo()
    click.echo(tabulate(
        rows,
        headers=headers,
        tablefmt="rounded_outline",
        colalign=("right", "left", "left", "right"),
    ))
    click.echo()
```

Now your table becomes much narrower because we're no longer putting authors and publication date into every row.

---

# 2. Make the similar-paper search paper-level

This is actually more important.

Your current `find_similar_papers()` is effectively doing:

```text
Paper
  │
  ├── Chunk 1 ── similarity
  ├── Chunk 2 ── similarity
  ├── Chunk 3 ── similarity
  ├── Chunk 4 ── similarity
  └── Chunk 5 ── similarity
```

So PostgreSQL returns:

```text
Paper A / Chunk 1 → 0.9881
Paper A / Chunk 2 → 0.2889
Paper A / Chunk 3 → 0.2286
...
```

What we actually want is:

```text
Paper A
   │
   ├── Chunk 1 → 0.98
   ├── Chunk 2 → 0.28
   └── Chunk 3 → 0.22
          ↓
     Best/combined score
          ↓
      Paper A → 0.98
```

And, importantly, **exclude the paper you're currently viewing**.

So the result should eventually look like:

```text
╭───┬──────────────────────────────────────┬──────────────┬────────┬────────────╮
│ # │ Title                                │ ArXiv ID     │ Score  │ Published  │
├───┼──────────────────────────────────────┼──────────────┼────────┼────────────┤
│ 1 │ Critical-State RL: Diagnosing ...    │ 2609.24985v1 │ 0.9881 │ 2026-09-21 │
│ 2 │ onPanda: Efficient Annotation ...    │ 2609.24983v1 │ 0.4123 │ 2026-09-21 │
╰───┴──────────────────────────────────────┴──────────────┴────────┴────────────╯
```

That's a much more meaningful "similar papers" result.

---

## 3. We can make the UI look much better

Since you're using `click`, we can take advantage of its terminal formatting.

For example, instead of:

```text
Finding papers similar to:
LoRA-generating hypernetworks...
```

we can display:

```text
╭──────────────────────────────────────────────────────────────╮
│                     SIMILAR PAPERS                           │
╰──────────────────────────────────────────────────────────────╯

Based on:
LoRA-generating hypernetworks for efficient on-device LLM
generative personalization
```

And the menu:

```text
╭──────────────────────────╮
│ What would you like to do?│
├──────────────────────────┤
│ 1  Show paper details    │
│ 2  Find similar papers   │
│ 3  Back                  │
╰──────────────────────────╯
```

We can also make the **search screen** look like:

```text
╭──────────────────────────────────────────────────────────────╮
│                         PAPER SEARCH                         │
╰──────────────────────────────────────────────────────────────╯

Query
  LoRA hypernetwork on-device personalization

Mode
  Semantic / Vector

Results: 5
──────────────────────────────────────────────────────────────

╭───┬──────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                │ ArXiv ID     │ Score  │
├───┼──────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks ...    │ 2609.24979v1 │ 0.5470 │
│ 2 │ ...                                  │ ...          │ 0.5123 │
╰───┴──────────────────────────────────────┴──────────────┴────────╯
```

### One thing I'd avoid

Don't try to put **everything** into the result table.

A good CLI design is:

```text
SEARCH RESULTS
     │
     ├── title
     ├── arxiv ID
     └── score
             │
             ▼
       User selects paper
             │
             ▼
       FULL PAPER DETAILS
             │
       ├── authors
       ├── categories
       ├── dates
       ├── abstract
       └── matching chunks
```

That gives the terminal much more breathing room.

**I'd fix the duplicate-paper problem at the same time as the table.** Otherwise the UI will look nicer, but "Find similar papers" will still give misleading results.

Absolutely. Let's test the table change first.

### 1. Make sure you're in the project

```bash
cd ~/arxiv_search
source env/bin/activate
```

### 2. Run the CLI

```bash
python scripts/cli.py --help
```

If that works, run:

```bash
python scripts/cli.py
```

Then try a search such as:

```text
LoRA hypernetwork on-device personalization
```

Choose **vector** search and let it display the results.

### 3. What we're looking for

The result table should now be much narrower, roughly:

```text
╭───┬───────────────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                         │ ArXiv ID     │ Score  │
├───┼───────────────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks for efficient... │ 2609.24979v1 │ 0.5470 │
│ 2 │ ...                                           │ ...          │ 0.51   │
╰───┴───────────────────────────────────────────────┴──────────────┴────────╯
```

If it still wraps or goes beyond your terminal width, **paste the output here**. We can then adjust the widths specifically for your terminal.

After that, we'll fix the **duplicate similar-paper results** so one paper doesn't appear five times.


---

```bash
python scripts/cli.py
```

```
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

  ArXiv Paper Search System - Personal Research Tool

Options:
  --help  Show this message and exit.

Commands:
  fetch   Fetch recent papers from ArXiv.
  manage  Database management operations.
  search  Search for papers.
  stats   Show database statistics.
```

```bash
python scripts/cli.py --help
```

```
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

  ArXiv Paper Search System - Personal Research Tool

Options:
  --help  Show this message and exit.

Commands:
  fetch   Fetch recent papers from ArXiv.
  manage  Database management operations.
  search  Search for papers.
  stats   Show database statistics.
```

```bash
python scripts/cli.py --help
```

```
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

  ArXiv Paper Search System - Personal Research Tool

Options:
  --help  Show this message and exit.

Commands:
  fetch   Fetch recent papers from ArXiv.
  manage  Database management operations.
  search  Search for papers.
  stats   Show database statistics.
```

```bash
python scripts/cli.py
```

```
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

  ArXiv Paper Search System - Personal Research Tool

Options:
  --help  Show this message and exit.

Commands:
  fetch   Fetch recent papers from ArXiv.
  manage  Database management operations.
  search  Search for papers.
  stats   Show database statistics.
```

---

Perfect. **The CLI itself is working correctly.** The output you're seeing is expected because `cli.py` is a Click command group—you need to choose one of its commands.

Let's test the **search UI** now.

### 1. Run search help

```bash
python scripts/cli.py search --help
```

You should see options similar to:

```text
Usage: cli.py search [OPTIONS]

  Search for papers.

Options:
  -q, --query TEXT
  -m, --mode [vector|hybrid|keyword]
  -l, --limit INTEGER
  -a, --authors TEXT
  -c, --categories TEXT
  -d, --days INTEGER
  -e, --export TEXT
  --help
```

### 2. Run a real search

Use:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m vector -l 5
```

This should load your embedding model and search the 3 papers currently in your database.

You should get the **new narrower table**:

```text
╭───┬───────────────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                         │ ArXiv ID     │ Score  │
├───┼───────────────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks for efficient.. │ 2609.24979v1 │ 0.5... │
│ 2 │ ...                                           │ ...          │ ...    │
╰───┴───────────────────────────────────────────────┴──────────────┴────────╯
```

### 3. Then test the interactive part

After the table appears, enter:

```text
1
```

You should get:

```text
1. Show paper details
2. Find similar papers
3. Back
```

Choose:

```text
1
```

That will test whether the **new compact result table + paper details interface** works together.

---

### One important thing

Don't worry about this:

```bash
python scripts/cli.py
```

showing the command list. That's normal for the way we currently built the CLI.

Think of it like:

```text
cli.py
  │
  ├── search
  ├── fetch
  ├── stats
  └── manage
```

So:

```bash
python scripts/cli.py search
python scripts/cli.py fetch
python scripts/cli.py stats
python scripts/cli.py manage
```

are the actual commands.

**Run the search command above and paste the resulting table here.** Then we'll see exactly how it looks in your terminal before changing anything else.

---

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m vector -l 5
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|████████████████████████████████████████| 103/103 [00:00<00:00, 10751.68it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================

╭─────┬───────────────────────────────────────────────┬──────────────┬─────────╮
│   # │ Title                                         │ ArXiv ID     │   Score │
├─────┼───────────────────────────────────────────────┼──────────────┼─────────┤
│   1 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.7641 │
│   2 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.6338 │
│   3 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.5814 │
│   4 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │   0.559 │
│   5 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.5373 │
╰─────┴───────────────────────────────────────────────┴──────────────┴─────────╯

╭─────┬───────────────────────────────────────────────┬──────────────┬─────────╮
│   # │ Title                                         │ ArXiv ID     │   Score │
├─────┼───────────────────────────────────────────────┼──────────────┼─────────┤
│   1 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.7641 │
│   2 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.6338 │
│   3 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.5814 │
│   4 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.559  │
│   5 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.5373 │
╰─────┴───────────────────────────────────────────────┴──────────────┴─────────╯

Enter result number to explore, 'q' to quit [q]: 1

======================================================================
Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
======================================================================

1. Show paper details
2. Find similar papers
3. Back
Choose an option (1, 2, 3): 1

================================================================================
LoRA-generating hypernetworks for efficient on-device LLM generative personalization
================================================================================

ArXiv ID: 2609.24979v1

Authors:
  Sean Augenstein, Li Ding, Jihwan Lee, Keith Rush, Andrey Zhmoginov

Published: 2026-09-21

Categories:
  cs.LG

Similarity score: 0.7641

Abstract:
On-device large language models (`LLMs'), e.g. running on mobile phones, are ripe for improvement via personalization. The limited compute resources of mobile devices impose limits on model scale and thus model quality, making any realizablequality gains highly impactful. At the same time, their personal nature (i.e., the close coupling to a particular user) means that a given on-device LLM tends to be used in similar, predictable patterns over the course of time. This paper presents a novel method for personalizing on-device LLMs. It trains a hypernetwork to map a user's context tokens to a low-rank adaptation (`LoRA') well-suited to that user. Once the trained common artifacts are deployed to users' devices, eachuser uses the hypernetwork to synthesize (entirely on device) a personalized LoRA. This approach blends the benefits while avoiding the drawbacks of two existing approaches to LLM customization: in-context learning (`ICL') and parameter-efficient fine-tuning (`PEFT'). Like ICL (and unlike PEFT), the on-device phase of our approach is computationally feasible, requiring only forward passes through neural networks. Like PEFT (and unlike ICL), our approach modifies the `target' base LLM via weights (the LoRA), avoiding negative consequences (e.g. increased latency) associated with extending the inputsequence. Our approach is particularly well-suited to the mobile device regime. Apart from the on-device compute and latency benefits mentioned, it also requires minimal additional storage, as internally its architecture partly leverages thesame LLM weights as belong to the target LLM to be personalized. We demonstrate the benefits of LoRA-generating hypernetworks on several representative personalization datasets, comparing against baselines like ICL and PEFT. Of note, our personalization experiments focus on more challenging and less studied long-form text generation tasks.

Matched chunks:
--------------------------------------------------------------------------------

Chunk 1
Section: Introduction
Page: None
Chunk score: 0.7641

Introduction

∗Corresponding author: saugenst@google.com. †Google. ‡Formerly at Google, work done while at Google. Preprint. LoRA-generating hypernetworks for efficient

on-device LLM generative personalization

Sean Augenstein∗†
Li Ding†
Jihwan Lee‡
Keith Rush†
Andrey Zhmoginov‡

======================================================================
Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
======================================================================

1. Show paper details
2. Find similar papers
3. Back
Choose an option (1, 2, 3): 3

Enter result number to explore, 'q' to quit [q]: q
(env) furba@fu:~/arxiv_search$
```

---

Yes — this test tells us exactly what is happening.

## 1. The table width is fixed now

This part is much better:

```text
╭─────┬───────────────────────────────────────────────┬──────────────┬─────────╮
│   # │ Title                                         │ ArXiv ID     │   Score │
```

It fits nicely in your terminal. **So the visual width problem is fixed.**

However, we have **two UI problems left**.

---

## 2. We're displaying the results table twice

You got:

```text
╭─────┬───────────────...
│ 1   │ LoRA...
...
╰─────┴───────────────╯

╭─────┬───────────────...
│ 1   │ LoRA...
...
╰─────┴───────────────╯
```

That's because `search()` is doing something like:

```python
display_results(results)

if results:
    explore_results(results, ...)
```

and `explore_results()` is displaying the results again before asking:

```text
Enter result number to explore
```

We should only display it **once**.

---

# 3. More importantly: your search is returning the same paper 5 times

This is the bigger architectural issue.

You searched:

```text
LoRA hypernetwork on-device personalization
```

and got:

```text
1  LoRA-generating... 0.7641
2  LoRA-generating... 0.6338
3  LoRA-generating... 0.5814
4  LoRA-generating... 0.5590
5  LoRA-generating... 0.5373
```

Those are **five different chunks**, not five different papers.

Remember our database structure:

```text
papers
   │
   ├── Paper 1
   │     ├── Chunk 1
   │     ├── Chunk 2
   │     ├── Chunk 3
   │     └── ...
   │
   ├── Paper 2
   │     ├── Chunk 1
   │     └── ...
   │
   └── Paper 3
         ├── Chunk 1
         └── ...
```

Our SQL currently searches `paper_chunks`, so PostgreSQL naturally says:

> "These five chunks are the five closest things to your query."

But our UI says:

> "Here are five papers."

That's the mismatch.

---

# 4. What we should change

The architecture should be:

```text
                    User query
                        │
                        ▼
                Embedding model
                        │
                        ▼
                 pgvector search
                        │
                        ▼
                  Find chunks
                        │
                        ▼
              Group by paper_id
                        │
                        ▼
        ┌───────────────┴───────────────┐
        │                               │
     Paper A                         Paper B
     best chunk                     best chunk
       0.76                            0.51
        │                               │
        └───────────────┬───────────────┘
                        ▼
                  Rank papers
                        │
                        ▼
                    CLI table
```

So if Paper A has 20 highly similar chunks, it should still appear **once**.

---

# 5. And your "Find similar papers" has the same problem

Earlier you got:

```text
Critical-State RL    0.9881
Critical-State RL    0.2889
Critical-State RL    0.2286
Critical-State RL    0.2065
Critical-State RL    0.2052
```

Same underlying issue.

We should fix the search engine rather than trying to hide duplicates in the UI.

---

## 6. The final CLI experience should look like this

For normal search:

```text
Searching...
======================================================================

Query: LoRA hypernetwork on-device personalization
Mode:  vector

╭───┬──────────────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                        │ ArXiv ID     │ Score  │
├───┼──────────────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks for efficient… │ 2609.24979v1 │ 0.7641 │
│ 2 │ Critical-State RL: Diagnosing Trainable ...  │ 2609.24985v1 │ 0.4218 │
│ 3 │ onPanda: Efficient Annotation of On-Policy... │ 2609.24983v1 │ 0.3982 │
╰───┴──────────────────────────────────────────────┴──────────────┴────────╯

Enter result number to explore, 'q' to quit: 
```

Then:

```text
Enter result number to explore, 'q' to quit: 1

Selected: LoRA-generating hypernetworks...

1. Show paper details
2. Find similar papers
3. Back
```

That's the behavior we want.

---

## Let's fix this next

I recommend we **don't change the visual design further yet**.

First fix the underlying search result grouping in:

```bash
nano src/search.py
```

Specifically:

1. **Vector search** → one result per paper.
    
2. **Keyword search** → one result per paper.
    
3. **Hybrid search** → one result per paper.
    
4. **Similar papers** → one result per paper.
    
5. Keep the best matching chunks attached to each paper.
    

Then we'll remove the duplicate table display in `cli.py`.

That will give us a clean foundation for the UI instead of putting cosmetic fixes on top of a search-result problem.

---

Yes. Let's fix it properly.

Your pgvector setup is fine: cosine similarity is calculated as `1 - (embedding <=> query)`, which is exactly what our current search is doing. ([GitHub](https://github.com/pgvector/pgvector?utm_source=chatgpt.com "GitHub - pgvector/pgvector: Open-source vector similarity search for Postgres · GitHub"))

We need to change **`src/search.py`**, then make one small change to **`scripts/cli.py`**.

## Step 1 — Replace `src/search.py`

Open it:

```bash
nano src/search.py
```

Rather than changing individual lines, replace the whole file with this version:

```python
import logging
from dataclasses import dataclass
from enum import Enum
from typing import List, Dict, Optional

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
    arxiv_id: str
    title: str
    abstract: str
    authors: List[str]
    score: float
    matched_chunks: List[Dict]
    published_date: str
    categories: List[str]


class PaperSearchEngine:

    def __init__(self, db_config: dict, embedding_generator: EmbeddingGenerator):
        self.db_config = db_config
        self.embedding_generator = embedding_generator

    def _get_connection(self):
        return psycopg2.connect(**self.db_config)

    # ---------------------------------------------------------
    # PUBLIC SEARCH
    # ---------------------------------------------------------

    def search(
        self,
        query: str,
        mode: SearchMode = SearchMode.VECTOR,
        limit: int = 10,
        filters: Optional[Dict] = None,
    ) -> List[SearchResult]:

        if not query or not query.strip():
            return []

        if limit <= 0:
            return []

        filters = filters or {}

        if mode == SearchMode.VECTOR:
            return self._vector_search(query, limit, filters)

        elif mode == SearchMode.KEYWORD:
            return self._keyword_search(query, limit, filters)

        elif mode == SearchMode.HYBRID:
            return self._hybrid_search(query, limit, filters)

        raise ValueError(f"Unknown search mode: {mode}")

    # ---------------------------------------------------------
    # VECTOR SEARCH
    # ---------------------------------------------------------

    def _vector_search(self, query, limit, filters):

        query_embedding = self.embedding_generator.generate_query_embedding(query)

        # Search more chunks than we ultimately need.
        #
        # Why?
        # Because several chunks may belong to the same paper.
        # We need enough candidates to get several different papers.
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

                1 - (c.embedding <=> %s::vector) AS score,

                c.id AS chunk_id,
                c.chunk_index,
                c.chunk_text,
                c.section_name,
                c.page_number

            FROM paper_chunks c

            JOIN papers p
                ON p.id = c.paper_id

            WHERE c.embedding IS NOT NULL
            {filter_sql}

            ORDER BY c.embedding <=> %s::vector

            LIMIT %s
        """

        params = [
            query_embedding.tolist(),
            *filter_params,
            query_embedding.tolist(),
            candidate_limit,
        ]

        with self._get_connection() as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cursor:

                cursor.execute(sql, params)

                rows = cursor.fetchall()

        return self._group_results_by_paper(rows, limit)

    # ---------------------------------------------------------
    # KEYWORD SEARCH
    # ---------------------------------------------------------

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

            WHERE c.chunk_text % %s
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

    # ---------------------------------------------------------
    # HYBRID SEARCH
    # ---------------------------------------------------------

    def _hybrid_search(self, query, limit, filters):

        candidate_limit = max(limit * 3, 20)

        vector_results = self._vector_search(
            query,
            candidate_limit,
            filters,
        )

        keyword_results = self._keyword_search(
            query,
            candidate_limit,
            filters,
        )

        return self._combine_results(
            vector_results,
            keyword_results,
            limit,
        )

    # ---------------------------------------------------------
    # GROUP CHUNKS INTO PAPERS
    # ---------------------------------------------------------

    def _group_results_by_paper(self, rows, limit):

        papers = {}

        for row in rows:

            paper_id = row["paper_id"]

            chunk = {
                "chunk_id": row["chunk_id"],
                "chunk_index": row["chunk_index"],
                "text": row["chunk_text"],
                "section_name": row["section_name"],
                "page_number": row["page_number"],
                "score": float(row["score"]),
            }

            if paper_id not in papers:

                papers[paper_id] = {
                    "paper_id": paper_id,
                    "arxiv_id": row["arxiv_id"],
                    "title": row["title"],
                    "abstract": row["abstract"],
                    "authors": row["authors"] or [],
                    "published_date": str(row["published_date"])
                    if row["published_date"]
                    else "",
                    "categories": row["categories"] or [],
                    "score": float(row["score"]),
                    "matched_chunks": [chunk],
                }

            else:

                paper = papers[paper_id]

                paper["matched_chunks"].append(chunk)

                # Keep the best chunk score as the paper score.
                if float(row["score"]) > paper["score"]:
                    paper["score"] = float(row["score"])

        # Sort papers by their best matching chunk.
        results = sorted(
            papers.values(),
            key=lambda x: x["score"],
            reverse=True,
        )

        # Keep only a few chunks per paper.
        for paper in results:
            paper["matched_chunks"] = sorted(
                paper["matched_chunks"],
                key=lambda x: x["score"],
                reverse=True,
            )[:3]

        return [
            SearchResult(**paper)
            for paper in results[:limit]
        ]

    # ---------------------------------------------------------
    # COMBINE VECTOR + KEYWORD
    # ---------------------------------------------------------

    def _combine_results(
        self,
        vector_results,
        keyword_results,
        limit,
    ):

        combined = {}

        # Vector weight
        vector_weight = 0.7

        # Keyword weight
        keyword_weight = 0.3

        for result in vector_results:

            combined[result.paper_id] = {
                "result": result,
                "vector_score": result.score,
                "keyword_score": 0.0,
            }

        for result in keyword_results:

            if result.paper_id not in combined:

                combined[result.paper_id] = {
                    "result": result,
                    "vector_score": 0.0,
                    "keyword_score": result.score,
                }

            else:

                combined[result.paper_id]["keyword_score"] = result.score

        final_results = []

        for data in combined.values():

            result = data["result"]

            final_score = (
                vector_weight * data["vector_score"]
                + keyword_weight * data["keyword_score"]
            )

            result.score = final_score

            final_results.append(result)

        final_results.sort(
            key=lambda result: result.score,
            reverse=True,
        )

        return final_results[:limit]

    # ---------------------------------------------------------
    # FILTERS
    # ---------------------------------------------------------

    def _build_filter_clause(self, filters):

        conditions = []
        params = []

        if filters.get("categories"):

            categories = filters["categories"]

            conditions.append(
                "p.categories && %s::text[]"
            )

            params.append(categories)

        if filters.get("date_from"):

            conditions.append(
                "p.published_date >= %s"
            )

            params.append(filters["date_from"])

        if filters.get("date_to"):

            conditions.append(
                "p.published_date <= %s"
            )

            params.append(filters["date_to"])

        if filters.get("authors"):

            authors = filters["authors"]

            conditions.append(
                "p.authors && %s::text[]"
            )

            params.append(authors)

        if not conditions:
            return "", []

        return " AND " + " AND ".join(conditions), params

    # ---------------------------------------------------------
    # SIMILAR PAPERS
    # ---------------------------------------------------------

    def find_similar_papers(
        self,
        paper_id,
        limit=10,
    ):

        with self._get_connection() as conn:

            with conn.cursor(cursor_factory=RealDictCursor) as cursor:

                # Get the strongest chunk from the paper.
                cursor.execute(
                    """
                    SELECT embedding
                    FROM paper_chunks
                    WHERE paper_id = %s
                      AND embedding IS NOT NULL
                    ORDER BY id
                    LIMIT 1
                    """,
                    (paper_id,),
                )

                row = cursor.fetchone()

                if not row:
                    return []

                embedding = row["embedding"]

                candidate_limit = max(limit * 10, 50)

                cursor.execute(
                    """
                    SELECT
                        p.id AS paper_id,
                        p.arxiv_id,
                        p.title,
                        p.abstract,
                        p.authors,
                        p.published_date,
                        p.categories,

                        1 - (c.embedding <=> %s::vector) AS score,

                        c.id AS chunk_id,
                        c.chunk_index,
                        c.chunk_text,
                        c.section_name,
                        c.page_number

                    FROM paper_chunks c

                    JOIN papers p
                        ON p.id = c.paper_id

                    WHERE c.embedding IS NOT NULL
                      AND p.id != %s

                    ORDER BY c.embedding <=> %s::vector

                    LIMIT %s
                    """,
                    (
                        embedding,
                        paper_id,
                        embedding,
                        candidate_limit,
                    ),
                )

                rows = cursor.fetchall()

        return self._group_results_by_paper(rows, limit)
```

### What changed?

The important new function is:

```python
_group_results_by_paper()
```

Previously:

```text
chunk → result
chunk → result
chunk → result
chunk → result
chunk → result
```

Now:

```text
chunk ─┐
chunk ─┤
chunk ─┤
chunk ─┘
        ↓
      paper
```

We keep the **best chunk score as the paper's score**, and retain up to three matching chunks for later display.

This is appropriate for your current setup because pgvector searches individual vectors/chunks, while our application wants paper-level results. pgvector itself supports nearest-neighbor searches and cosine distance; the grouping is application-level logic. ([GitHub](https://github.com/pgvector/pgvector?utm_source=chatgpt.com "GitHub - pgvector/pgvector: Open-source vector similarity search for Postgres · GitHub"))

---

# Step 2 — Fix the duplicate table

Now open:

```bash
nano scripts/cli.py
```

Find `explore_results()`.

You probably have something similar to:

```python
def explore_results(results, search_engine):
    display_results(results)

    while True:
        ...
```

Change it so that **the first thing inside `explore_results()` is NOT `display_results(results)`**.

Use:

```python
def explore_results(results, search_engine):
    """Interactively explore already-displayed search results."""

    while True:
        choice = click.prompt(
            "\nEnter result number to explore, 'q' to quit",
            default="q",
        )

        if choice.lower() == "q":
            break

        try:
            index = int(choice) - 1

            if index < 0 or index >= len(results):
                click.echo("Invalid result number.")
                continue

        except ValueError:
            click.echo("Please enter a number or 'q'.")
            continue

        selected = results[index]

        click.echo()
        click.echo("=" * 70)
        click.echo(f"Selected: {selected.title}")
        click.echo("=" * 70)

        while True:

            click.echo()
            click.echo("1. Show paper details")
            click.echo("2. Find similar papers")
            click.echo("3. Back")

            option = click.prompt(
                "Choose an option",
                type=click.Choice(["1", "2", "3"]),
            )

            if option == "1":

                show_paper_details(selected)

            elif option == "2":

                find_similar(
                    selected,
                    search_engine,
                )

            elif option == "3":

                break
```

The key difference is:

```python
# DON'T DO THIS HERE
display_results(results)
```

because `search()` already displays it.

---

# Step 3 — Test it

First test syntax:

```bash
python -m py_compile src/search.py scripts/cli.py
```

If there is **no output**, that's good.

Then run:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m vector -l 5
```

## What we expect now

Instead of:

```text
1 LoRA-generating... 0.7641
2 LoRA-generating... 0.6338
3 LoRA-generating... 0.5814
4 LoRA-generating... 0.5590
5 LoRA-generating... 0.5373
```

you should get something more like:

```text
╭───┬──────────────────────────────────────────────┬──────────────┬────────╮
│ # │ Title                                        │ ArXiv ID     │ Score  │
├───┼──────────────────────────────────────────────┼──────────────┼────────┤
│ 1 │ LoRA-generating hypernetworks for efficient… │ 2609.24979v1 │ 0.7641 │
│ 2 │ Critical-State RL: Diagnosing Trainable ...  │ 2609.24985v1 │ 0.4... │
│ 3 │ onPanda: Efficient Annotation of On-Policy... │ 2609.24983v1 │ 0.3... │
╰───┴──────────────────────────────────────────────┴──────────────┴────────╯
```

And **only one table** should appear.

Then choose:

```text
1
```

and test:

```text
2. Find similar papers
```

The similar-paper results should also contain **different papers rather than five chunks of the same paper**.

### One small note

I intentionally kept the implementation simple for now. Later, we can improve the similarity calculation by representing a paper with multiple chunk embeddings rather than just selecting one chunk. For your current 3-paper prototype, the important architectural correction is that **search results are paper-level, while chunks remain evidence attached to the paper**.

---

```bash
nano src/search.py
python -m py_compile src/search.py scripts/cli.py
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m vector -l 5
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|█████████████████████████████████████████████████████████████| 103/103 [00:00<00:00, 10211.15it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================

╭─────┬───────────────────────────────────────────────┬──────────────┬─────────╮
│   # │ Title                                         │ ArXiv ID     │   Score │
├─────┼───────────────────────────────────────────────┼──────────────┼─────────┤
│   1 │ LoRA-generating hypernetworks for efficien... │ 2609.24979v1 │  0.7641 │
│   2 │ onPanda: Efficient Annotation of On-Policy... │ 2609.24983v1 │  0.2664 │
│   3 │ Critical-State RL: Diagnosing Trainable St... │ 2609.24985v1 │  0.1858 │
╰─────┴───────────────────────────────────────────────┴──────────────┴─────────╯


Enter result number to explore, 'q' to quit [q]: 1

======================================================================
Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
======================================================================

1. Show paper details
2. Find similar papers
3. Back
Choose an option (1, 2, 3): 2

Finding papers similar to:
LoRA-generating hypernetworks for efficient on-device LLM generative personalization


╭─────┬───────────────────────────────────────────────┬──────────────┬─────────╮
│   # │ Title                                         │ ArXiv ID     │   Score │
├─────┼───────────────────────────────────────────────┼──────────────┼─────────┤
│   1 │ Critical-State RL: Diagnosing Trainable St... │ 2609.24985v1 │  0.9881 │
│   2 │ onPanda: Efficient Annotation of On-Policy... │ 2609.24983v1 │  0.1866 │
╰─────┴───────────────────────────────────────────────┴──────────────┴─────────╯


======================================================================
Selected: LoRA-generating hypernetworks for efficient on-device LLM generative personalization
======================================================================

1. Show paper details
2. Find similar papers
3. Back
Choose an option (1, 2, 3): 3

Enter result number to explore, 'q' to quit [q]: q
(env) furba@fu:~/arxiv_search$
```

---

Excellent. **The main search architecture is now working correctly.**

Your output shows that we fixed both problems.

### Search results

You now get one row per paper:

```text
1 │ LoRA-generating hypernetworks... │ 2609.24979v1 │ 0.7641
2 │ onPanda...                       │ 2609.24983v1 │ 0.2664
3 │ Critical-State RL...             │ 2609.24985v1 │ 0.1858
```

Instead of five chunks from the same paper.

### Similar papers

You also get:

```text
1 │ Critical-State RL... │ 2609.24985v1 │ 0.9881
2 │ onPanda...           │ 2609.24983v1 │ 0.1866
```

And importantly, the selected LoRA paper itself is excluded.

So our architecture is now:

```text
                    User query
                       │
                       ▼
                Embedding model
                       │
                       ▼
                  pgvector
                       │
                       ▼
                 Chunk matches
                       │
                       ▼
                Group by paper
                       │
                       ▼
              Best chunk = score
                       │
                       ▼
                 Paper results
                       │
                       ▼
                    CLI UI
```

That's a much better foundation.

---

## One thing I notice

The score:

```text
Critical-State RL → 0.9881
```

for "similar papers" is **very high** compared with:

```text
onPanda → 0.1866
```

That doesn't necessarily mean the system is wrong. Remember, our current `find_similar_papers()` uses **one chunk from the selected paper** as the query vector.

So we're currently doing:

```text
LoRA paper
   │
   └── one chunk
          │
          ▼
       pgvector
          │
          ├── Critical-State RL → 0.9881
          └── onPanda → 0.1866
```

That's a simplification. Eventually we can make paper similarity much more robust by representing the **whole paper using multiple chunk embeddings**.

But I would **not change that yet**. We should keep moving through the architecture one piece at a time.

---

# Next UI improvement

The functionality is now good, but visually we can improve the experience.

Right now you still have:

```text
======================================================================
Selected: ...
======================================================================
```

and the paper details are very wide.

We can make the CLI feel more like a real research application:

```text
╭──────────────────────────────────────────────────────────────╮
│ PAPER                                                        │
│ LoRA-generating hypernetworks for efficient on-device LLM   │
│ generative personalization                                  │
╰──────────────────────────────────────────────────────────────╯

ArXiv      2609.24979v1
Published  2026-09-21
Category   cs.LG
Score      0.7641

Authors
  Sean Augenstein
  Li Ding
  Jihwan Lee
  Keith Rush
  Andrey Zhmoginov

Abstract
──────────────────────────────────────────────────────────────
On-device large language models...
```

And eventually:

```text
╭──────────────────── SEARCH ────────────────────╮
│                                               │
│  Query: LoRA hypernetwork on-device           │
│  Mode:  Vector                                │
│  Results: 3                                   │
│                                               │
╰───────────────────────────────────────────────╯
```

We can do that next.

**But before changing the UI further, I'd make one small functional improvement: test all three search modes — `vector`, `keyword`, and `hybrid`.** This will verify that our new paper-level grouping didn't break the other two search paths.

Run:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m keyword -l 5
```

then:

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m hybrid -l 5
```

Paste both outputs. Then we'll know the entire search layer is behaving correctly before polishing the UI.

---

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m keyword -l 5
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|█████████████████████████████████████████████████████████████| 103/103 [00:00<00:00, 11388.25it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================
Traceback (most recent call last):
  File "/home/furba/arxiv_search/scripts/cli.py", line 744, in <module>
    cli()
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1631, in __call__
    return self.main(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1552, in main
    rv = self.invoke(ctx)
           ^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 2032, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1415, in invoke
    return ctx.invoke(self.callback, **ctx.params)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 910, in invoke
    return callback(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/decorators.py", line 34, in new_func
    return f(get_current_context(), *args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/scripts/cli.py", line 182, in search
    results = engine.search(
              ^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/src/search.py", line 66, in search
    return self._keyword_search(query, limit, filters)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/src/search.py", line 189, in _keyword_search
    cursor.execute(sql, params)
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/psycopg2/extras.py", line 236, in execute
    return super().execute(query, vars)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
IndexError: list index out of range
```

```bash
python scripts/cli.py search -q "LoRA hypernetwork on-device personalization" -m hybrid -l 5
```

```
Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
Loading weights: 100%|█████████████████████████████████████████████████████████████| 103/103 [00:00<00:00, 11272.95it/s]
/home/furba/arxiv_search/src/embeddings.py:57: FutureWarning: The `get_sentence_embedding_dimension` method has been renamed to `get_embedding_dimension`.
  self.model.get_sentence_embedding_dimension()

Searching...
======================================================================
Traceback (most recent call last):
  File "/home/furba/arxiv_search/scripts/cli.py", line 744, in <module>
    cli()
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1631, in __call__
    return self.main(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1552, in main
    rv = self.invoke(ctx)
           ^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 2032, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/core.py", line 1415, in invoke
    return ctx.invoke(self.callback, **ctx.params)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/decorators.py", line 910, in invoke
    return callback(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/click/decorators.py", line 34, in new_func
    return f(get_current_context(), *args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/scripts/cli.py", line 182, in search
    results = engine.search(
              ^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/src/search.py", line 69, in search
    return self._hybrid_search(query, limit, filters)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/src/search.py", line 209, in _hybrid_search
    keyword_results = self._keyword_search(
                      ^^^^^^^^^^^^^^^^^^^^^
  File "/home/furba/arxiv_search/src/search.py", line 189, in _keyword_search
    cursor.execute(sql, params)
  File "/home/furba/arxiv_search/env/lib/python3.12/site-packages/psycopg2/extras.py", line 236, in execute
    return super().execute(query, vars)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
IndexError: list index out of range
(env) furba@fu:~/arxiv_search$
```

