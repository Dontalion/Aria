# Design: Keyword Enrichment Pipeline

## Purpose

Expand a small set of seed keywords into a larger, research-validated keyword
set by using web search and intelligent classification. Researchers get angles
they would not think of manually, grounded in real web results rather than
guesswork. This is one pipeline type within ARIA — the system architecture
supports additional pipeline types defined through configuration.

## Input

- A source keyword file containing seed keywords organized into named
  sub-sections
- The pipeline is invoked for a specific Sub_Section and reads every source
  keyword listed under that Sub_Section heading
- Source files are treated as **read-only** — they are never modified by this
  pipeline

## Process (per source keyword)

1. **Read the Sub_Section** — Collect all source keywords under the requested
   Sub_Section heading. If the Sub_Section is absent or contains zero keywords,
   the pipeline logs an error naming the missing or empty Sub_Section and stops
   processing that Sub_Section. No output file is created or modified in that
   case.

2. **Generate search terms** — For each seed keyword, produce exactly 3 distinct
   search terms designed to surface diverse, relevant web results.

3. **Parallel web search** — Execute all 3 searches concurrently and wait for all
   3 to return before moving on. This maximizes throughput and reduces
   wall-clock time.

4. **Classify results** — Each search result is given exactly one label,
   `accepted` or `rejected`, judged against the configured research topic. Every
   classification records a short reason so decisions stay traceable.

5. **Retry on low acceptance** — If more than 50% of the results for a seed
   keyword are rejected, the pipeline generates exactly 1 replacement search
   term and runs one additional search-and-classify cycle for that term only.
   At most 1 retry happens per source keyword, which prevents loops.

6. **Skip when nothing survives** — If a source keyword has zero accepted results
   after its single retry is exhausted, the pipeline skips keyword generation for
   that keyword, logs the reason, and continues with the remaining source
   keywords.

7. **Generate new keywords** — When at least 1 accepted result is available, the
   pipeline produces exactly 10 new keywords for that source keyword. Each new
   keyword is distinct from the source keywords and from keywords already
   generated within the same Sub_Section.

8. **Write output** — Generated keywords are appended to the output file in
   batches of no more than 20 keywords per write. Only complete batches are
   written — a partial batch is never flushed.

## Output

- A new keyword file, separate from the source, organized by the same
  Sub_Section structure as the input
- Keywords appended in complete batches of up to 20 to avoid large single writes
- Each batch is a valid, well-formed addition to the file's Sub_Section structure

## Constraints

- Source keyword files are never written to or modified
- Exactly 3 search terms are generated per source keyword
- At most 1 retry per source keyword (triggered only when more than 50% of
  results are rejected), and that retry adds exactly 1 replacement term
- A missing or empty Sub_Section terminates that Sub_Section with no output
  change
- A keyword with zero accepted results after retry is skipped and logged, never
  silently dropped without a record
- Exactly 10 new keywords are generated per qualifying source keyword, each
  distinct from source keywords and from already-generated keywords in the same
  Sub_Section
- Batch size of 20 keeps file operations predictable and recoverable; partial
  batches are not written
- All web searches within a single keyword run in parallel
