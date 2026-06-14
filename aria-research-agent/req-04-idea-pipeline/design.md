# Design: Idea Generation Pipeline

## Purpose

Generate structured research outputs (project ideas, summaries, analyses) from
keywords using web research, so researchers get actionable results backed by
real-world context rather than unverified suggestions. This is one pipeline type
within ARIA — the system supports defining other pipeline types (source
completion, synthesis, comparison, and more) through configuration.

## Input

- A keyword file organized by named sub-sections
- Each sub-section contains one or more research keywords
- The pipeline reads all keywords from the file grouped by their Sub_Section. If
  the file is missing, empty, or cannot be parsed into Sub_Sections, the pipeline
  logs an error naming the cause and terminates without writing any output file.

## Process (per keyword)

1. **Generate search terms** — From each keyword, produce exactly 3 search terms
   derived from that keyword.

2. **Parallel web search** — Execute all 3 searches concurrently to reduce
   wall-clock time.

3. **Classify results** — Evaluate each result and label it `accepted` or
   `rejected` based on its relevance to the configured research topic.

4. **Retry on low acceptance** — If more than half of the results for a keyword
   are rejected, regenerate one search term and re-run the searches. At most 1
   retry happens per keyword.

5. **Skip when nothing survives** — If no accepted results remain for a keyword
   after the single retry is exhausted, skip idea generation for that keyword,
   log the reason, and continue with the remaining keywords.

6. **Generate ideas** — When at least 1 accepted result is available, produce
   exactly 10 Idea_Records for that keyword. Each record contains all fields of
   the configured output schema.

7. **Deduplicate** — Drop any idea whose title already exists in the output file,
   compared case-insensitively, so titles stay unique within that file.

8. **Write output** — Append surviving ideas to the output file in batches of no
   more than 5 records per write.

## Output

- One output file per sub-section
- Ideas appended in batches of up to 5 (not all at once)
- Each idea contains the configured fields and a unique title within the file

## Idea Structure (default fields)

| Field | Type | Description |
|-------|------|-------------|
| title | text | Concise idea name (unique per output file) |
| description | text | What the idea involves |
| resume_worthiness | text | Why this is resume-worthy, naming at least one specific technology |

## Quality Gate vs Structural Gates

An idea is finalized only when it passes both the quality gate and the structural
gates:

- **Quality gate** — A Reviewer ACCEPT decision is a quality judgment only. It
  does not by itself qualify an idea for output.
- **Structural gates** — The idea must also clear deduplication (no duplicate
  title in the file, case-insensitive) and conform to the configured Idea_Record
  schema.

An idea that the Reviewer accepts but that fails deduplication or schema
conformance is not written.

## Configurability

- Idea_Record fields are configurable per pipeline instance
- The default schema above applies when no custom schema is provided
- Other pipeline types can define entirely different output structures

## Constraints

- Exactly 3 search terms are generated per keyword
- At most 1 retry per keyword, triggered only when more than half of results are
  rejected
- A missing, empty, or unparsable keyword file terminates the run with no output
- A keyword with zero accepted results after retry is skipped and logged
- Exactly 10 Idea_Records are generated per qualifying keyword
- Batch size fixed at 5 records per file append
- Title uniqueness is enforced per output file using case-insensitive comparison
- Reviewer ACCEPT is a quality gate only; ideas must also pass dedup and schema
  conformance to be finalized
