# Req-7: Output Management & Multi-Format Export — Technical Spec

## Stack

| Library       | Role                                            |
|---------------|-------------------------------------------------|
| `pydantic`    | `OutputMetadata` model + format validation      |
| `weasyprint`  | HTML → PDF rendering for the PDF formatter       |
| `python-docx` | DOCX document generation                         |
| built-in      | Markdown + JSON rendering (no extra dependency) |

## Module Layout

```
src/aria/output/
├── __init__.py     # exports OutputFormatter, create_formatter, write_output
├── formatter.py    # OutputFormatter ABC + create_formatter factory
├── markdown.py     # MarkdownFormatter (default)
├── json_out.py     # JSONFormatter
├── pdf.py · docx.py · metadata.py · writer.py
```

## Abstract Base Class (Req 7.1)

```python
# src/aria/output/formatter.py
from abc import ABC, abstractmethod
from aria.output.metadata import OutputMetadata

class OutputFormatter(ABC):
    @abstractmethod
    def format(self, data: dict, metadata: OutputMetadata) -> bytes | str:
        """Render internal research data into the target format."""

    @abstractmethod
    def file_extension(self) -> str:
        """Return the file extension without the leading dot (e.g. 'md')."""
```

## Concrete Formatters (Req 7.1, 7.6)

| Class             | Module      | Library     | Returns | Metadata placement | Renders |
|-------------------|-------------|-------------|---------|--------------------|---------|
| MarkdownFormatter | markdown.py | built-in    | str     | YAML frontmatter   | `---` block then body |
| JSONFormatter     | json_out.py | built-in    | str     | dedicated fields   | `json.dumps(indent=2)` with top-level `metadata` |
| PDFFormatter      | pdf.py      | weasyprint  | bytes   | header section     | Markdown → HTML → `weasyprint.HTML(...).write_pdf()` |
| DOCXFormatter     | docx.py     | python-docx | bytes   | header section     | `docx.Document` header + body styles |

## Factory (Req 7.1, 7.3, 7.4)

```python
# src/aria/output/formatter.py
def create_formatter(fmt: str) -> OutputFormatter:
    registry = {
        "md": MarkdownFormatter,
        "json": JSONFormatter,
        "pdf": PDFFormatter,
        "docx": DOCXFormatter,
    }
    if fmt not in registry:                       # Req 7.4 — reject at load time
        raise ValueError(
            f"Unsupported output format: {fmt!r}. Allowed: {sorted(registry)}"
        )
    return registry[fmt]()
```

- When the config requests no format, the caller defaults to `["md"]` (Req 7.3).
- An unsupported format is rejected before any file is written, naming it (Req 7.4).

## Metadata Model (Req 7.6)

```python
# src/aria/output/metadata.py
from datetime import datetime
from pydantic import BaseModel

class OutputMetadata(BaseModel):
    sources: list[str]          # contributing source file paths
    section: str            # which sub-section this output belongs to
    generated_at: datetime      # generation date
    queries_used: list[str]     # search queries that produced the data
    model_id: str
    aria_version: str
```

## File Writer (Req 7.5, 7.7–7.10)

```python
# src/aria/output/writer.py
from pathlib import Path

def write_output(path: Path, content: bytes | str, force: bool = False) -> bool:
    path.parent.mkdir(parents=True, exist_ok=True)         # Req 7.10 (incl. parents)
    if path.exists() and path.stat().st_size > 100 and not force:
        logger.warning("Preserved existing file, not overwritten: %s", path)  # Req 7.8
        return False                                       # Req 7.7
    mode = "wb" if isinstance(content, bytes) else "w"
    with open(path, mode) as fh:
        fh.write(content)
    return True                                            # Req 7.9 when force=True
```

- The caller builds the path `{output_dir}/{section}/{subsection}.{ext}` (Req 7.5).
  Files of 100 chars or fewer are written normally; larger files require `force=True`.

## Dependencies

`weasyprint`, `python-docx`, `pydantic` — declared in `pyproject.toml`.
