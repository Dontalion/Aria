# Output Management & Multi-Format Export — Design

## Purpose

ARIA produces research results in multiple output formats so users can consume
findings in whatever tool or workflow they prefer. Markdown is the default, with
JSON, PDF, and DOCX available on demand. Multiple formats can be generated from a
single run, all from the same internal research data.

## Supported Formats (Exactly Four)

| Format            | Use Case                                          |
|-------------------|---------------------------------------------------|
| Markdown (default)| Readable, version-friendly, quick iteration       |
| JSON              | Structured data, programmatic consumption          |
| PDF               | Sharing, archival, printing                        |
| DOCX              | Editing in Word, collaboration                     |

ARIA supports these four and only these four (Req 9.1).

## Choosing Formats

- The config can request one or more formats; ARIA generates one output file per
  requested format from the same internal data (Req 9.2).
- When **no format is specified**, ARIA defaults to **Markdown** (Req 9.3).
- When a requested format is **not one of the four supported formats**, ARIA
  **rejects the configuration at load time**, returns an error that names the
  unsupported format, and writes **no output files** (Req 9.4).

## File Organization

Every output file follows a predictable path pattern (Req 9.5):

```
{output_dir}/{section_name}/{subsection_name}.{format_extension}
```

Example: `output/literature-review/key-findings.md`

If the target directory does not exist, ARIA creates it — **including any missing
parent directories** — before writing the file (Req 9.10). Users never need to
create folders manually.

## Metadata in Every Output

Each generated file embeds the same metadata (Req 9.6):

- **Source file paths** — which inputs contributed
- **Sub-section name** — which sub-section this output belongs to
- **Generation date** — when the output was produced
- **Search queries used** — what queries produced the underlying data

The metadata is placed according to each format:

| Format   | Metadata placement       |
|----------|--------------------------|
| Markdown | YAML frontmatter         |
| JSON     | Dedicated metadata fields|
| PDF      | Header section           |
| DOCX     | Header section           |

## Safety: No Silent Overwrites

ARIA protects existing work (Req 9.7–9.9):

- If a file at the target path already contains **more than 100 characters** and
  **no force flag** is provided, ARIA **preserves the existing file unchanged** and
  does not write the new content (Req 9.7).
- When a file is preserved this way, ARIA **emits a warning** that names the file
  path that was preserved and not overwritten (Req 9.8).
- If the existing file has more than 100 characters and a **force flag is
  provided**, ARIA **overwrites** it with the new content (Req 9.9).

Small or empty files (100 characters or fewer) are written normally.

## Simultaneous Generation

Users can request multiple formats in one command. All requested formats are
generated from the same internal data:

```
aria run literature-review --format md,pdf,json
```

## What Users Don't Need to Know

- How each formatter renders the internal data
- How directories are created recursively
- How the 100-character overwrite check is implemented

## Dependencies

- Req 07 (Configuration) — selects formats and the output directory
- Req 05 (Orchestrator) — supplies the internal research data to format
