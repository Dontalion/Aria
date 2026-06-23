# Req-7: Output Management & Multi-Format Export — Tasks

## Dependencies

- **Req 3 (Orchestrator)** — supplies the internal research data to be formatted
- **Req 5 (Configuration)** — selects requested formats and the output directory

## Implementation Tasks

- [ ] 1. Create `OutputMetadata` Pydantic model in `src/aria/output/metadata.py`
  - Fields: sources, section, generated_at, queries_used, model_id, aria_version (Req 7.6)
- [ ] 2. Create `OutputFormatter` ABC in `src/aria/output/formatter.py`
  - Abstract `format(data, metadata)` and `file_extension()` methods (Req 7.1)
- [ ] 3. Implement `MarkdownFormatter` (default) in `src/aria/output/markdown.py`
  - Emit metadata as YAML frontmatter, then rendered body (Req 7.1, 7.3, 7.6)
- [ ] 4. Implement `JSONFormatter` in `src/aria/output/json_out.py`
  - Pretty-printed JSON with a dedicated `metadata` object (Req 7.1, 7.6)
- [ ] 5. Implement `PDFFormatter` using weasyprint in `src/aria/output/pdf.py`
  - Markdown → HTML → PDF, metadata rendered as a header section (Req 7.1, 7.6)
- [ ] 6. Implement `DOCXFormatter` using python-docx in `src/aria/output/docx.py`
  - Heading/paragraph styles, metadata header section (Req 7.1, 7.6)
- [ ] 7. Implement `create_formatter(fmt)` factory with the four-format registry
  - Reject unsupported formats at load time, naming the format, write no files (Req 7.4)
  - Caller defaults to Markdown when no format is specified (Req 7.3)
- [ ] 8. Implement `write_output(path, content, force)` in `src/aria/output/writer.py`
  - Build path `{output_dir}/{section}/{subsection}.{ext}` (Req 7.5)
  - Auto-create directories including parents before writing (Req 7.10)
  - Preserve existing files >100 chars with a warning unless force flag set (Req 7.7, 7.8)
  - Overwrite >100-char files when force flag provided (Req 7.9)
- [ ] 9. Add parallel multi-format generation from the same internal data (Req 7.2)
- [ ] 10. Write unit tests for all four formatters (mock weasyprint/python-docx)
  - Assert metadata is present and parseable in every format (Req 7.6)
- [ ] 11. Write integration test: data → format → write across md/json/pdf/docx
  - Cover overwrite protection, force overwrite, and auto-mkdir behavior

## Acceptance Criteria Coverage

- 9.1 — exactly four formats supported (Markdown default)
- 9.2 — multiple formats generated from the same internal data
- 9.3 — no format specified defaults to Markdown
- 9.4 — unsupported format rejected at load, named, no files written
- 9.5 — path pattern `{output_dir}/{section_name}/{subsection_name}.{ext}`
- 9.6 — metadata embedded per-format (frontmatter/fields/header)
- 9.7–9.9 — overwrite protection for >100-char files unless forced, with warning
- 9.10 — directories created (including parents) before writing
