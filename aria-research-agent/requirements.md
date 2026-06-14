# Requirements Document

## Introduction

ARIA (Autonomous Research & Idea Agent) is a standalone Python multi-agent research system that performs automated research workflows — including but not limited to keyword enrichment, idea generation, source completion, multi-topic exploration, and other research scenarios. The system replaces the prior bash-based orchestrator (`run-enrichment.sh` + kiro-cli) with a proper application built on a multi-agent framework. It is provider-agnostic (LLM and search), resumable, parallel, and designed for headless background execution. The architecture is topic-agnostic and workflow-agnostic — the first use case is AI engineering project idea generation, but the system supports any research domain and any research workflow pattern.

## Glossary

- **Research_Agent**: The top-level Python application that orchestrates research workflows through multi-agent coordination
- **Research_Workflow**: A configurable sequence of pipeline stages (e.g., keyword enrichment → idea generation, or source completion → synthesis → report). Not limited to any single pattern.
- **Keyword_Pipeline**: One pipeline type — reads source keywords, performs web searches, and generates new enriched keywords organized by sub-section
- **Idea_Pipeline**: One pipeline type — reads keyword files, performs web searches, and generates structured project ideas for each keyword
- **LLM_Provider**: An abstraction over language model backends (OpenRouter, Ollama, direct APIs, CLI tools) that exposes a unified interface for text generation
- **Search_Provider**: An abstraction over web search backends (DuckDuckGo, Tavily, Brave Search) that exposes a unified interface for retrieving search results
- **State_Store**: The database-backed persistence layer that tracks pipeline progress, task dependencies, retry queues, and enables crash recovery and resumption
- **Task**: A single unit of work (e.g., process one keyword, generate ideas for one sub-section). Tasks have dependencies, status, and retry history.
- **Task_Queue**: The set of pending, in-progress, failed, and completed tasks managed by the State_Store
- **Config**: A YAML configuration file specifying provider settings, model selection, parallelism limits, file paths, retry policies, and workflow definitions
- **Sub_Section**: A logical grouping of keywords within a keyword file, processed as a unit
- **Idea_Record**: A structured output containing configurable fields (default: title, description, resume-worthiness) for a single research output item
- **Orchestrator**: The multi-agent coordination layer that manages agent execution order, parallelism, task dependencies, and state transitions
- **Agent_Node**: A single unit of work within the orchestration graph (e.g., search agent, keyword generator, idea generator)
- **Output_Formatter**: The layer responsible for rendering results into the configured output format (Markdown, JSON, PDF, DOCX, etc.)
- **Cache_Layer**: In-memory and persistent cache for LLM responses and search results to avoid redundant API calls
- **Vector_Store**: Optional vector database for semantic similarity search across generated content (deduplication, finding related ideas)
- **Graph_Database**: Optional graph database for mapping relationships between keywords, ideas, topics, and sources
- **Reviewer_Agent**: An optional agent that validates and critiques outputs produced by the Executor before they are finalized
- **Executor_Agent**: The agent that performs primary generation work (keywords, ideas, and other research outputs)
- **Skill**: A Markdown-defined unit of agent behavior with a name, description, input/output schema, and prompt template
- **Wiki**: A per-research-project knowledge base that accumulates findings, decisions, sources, and cross-references over time

## Requirements

### Requirement 1: LLM Provider Abstraction

**User Story:** As a developer, I want to swap LLM providers without changing pipeline logic, so that I can use whichever model is available or cost-effective.

#### Acceptance Criteria

1. THE LLM_Provider SHALL expose a unified text-generation interface that accepts a prompt string as input and returns a completion as a text string.
2. WHEN OpenRouter is configured, THE LLM_Provider SHALL route requests through the OpenRouter API using OpenAI-compatible endpoints.
3. WHEN Ollama is configured, THE LLM_Provider SHALL route requests to a local Ollama instance at the configured base URL.
4. WHEN a CLI tool is configured, THE LLM_Provider SHALL invoke the specified CLI binary (one of: kiro-cli, codex, claude-code, gemini-cli) as a subprocess.
5. WHEN a direct API provider is configured, THE LLM_Provider SHALL use the native SDK for the specified provider (one of: Anthropic, OpenAI).
6. IF the configured LLM_Provider returns a connection error or fails to return a response within the configured request timeout (default 60 seconds, valid range 1 to 600 seconds), THEN THE Research_Agent SHALL log the error and retry the request, applying exponential backoff between successive attempts, up to the configured retry limit (default 3 retries).
7. WHEN all configured retry attempts for an unreachable LLM_Provider are exhausted, THE Research_Agent SHALL mark the task as failed and report an error indicating which provider failed and the number of attempts made, without falling back to a different provider.
8. THE Config SHALL specify the active LLM_Provider type, model name, API key reference, and optional parameters including temperature, max tokens, request timeout, and retry limit.
9. WHEN the LLM_Provider configuration is loaded, THE Research_Agent SHALL validate that the provider type is one of the supported values (openrouter, ollama, cli_tool, direct_api) and that optional parameters fall within valid bounds (temperature 0.0 to 2.0, max tokens >= 1, retry limit 0 to 10, request timeout 1 to 600 seconds).
10. IF the LLM_Provider configuration is invalid because the provider type is unsupported or an optional parameter falls outside its valid bounds, THEN THE Research_Agent SHALL reject the configuration at load time and report an error indicating which field is invalid, without starting any pipeline task.

### Requirement 2: Search Provider Abstraction

**User Story:** As a developer, I want to swap web search providers without changing pipeline logic, so that I can use free search in development and paid search in production.

#### Acceptance Criteria

1. THE Search_Provider SHALL expose a unified interface that accepts a query string and returns between 0 and the configured maximum number of results (default: 5), each result containing a non-empty title, a URL, and a snippet
2. WHEN DuckDuckGo is configured, THE Search_Provider SHALL perform searches using the DuckDuckGo API without requiring an API key
3. WHEN Tavily is configured, THE Search_Provider SHALL perform searches using the Tavily API with the configured API key
4. WHEN Brave Search is configured, THE Search_Provider SHALL perform searches using the Brave Search API with the configured API key
5. IF a search request fails due to a network error or an error response, THEN THE Search_Provider SHALL retry the request once after a 3-second delay before returning an empty result set containing 0 results
6. WHEN a retried search request succeeds, THE Search_Provider SHALL return the results from the successful response rather than an empty result set
7. THE Config SHALL specify the active Search_Provider as one of the supported values (DuckDuckGo, Tavily, Brave Search) and, for Tavily and Brave Search, the associated API key credential
8. WHEN the Search_Provider configuration is loaded, THE Research_Agent SHALL validate that the configured provider is one of the supported values (DuckDuckGo, Tavily, Brave Search) and that an API key credential is present for Tavily and Brave Search
9. IF the Search_Provider configuration specifies an unsupported provider or omits a required API key credential, THEN THE Research_Agent SHALL log an error identifying the invalid field and exit before performing any search

### Requirement 3: Keyword Enrichment Pipeline

**User Story:** As a researcher, I want to automatically expand a set of seed keywords into a larger enriched keyword set using web research, so that I cover angles I would not think of manually.

#### Acceptance Criteria

1. WHEN the Keyword_Pipeline is invoked for a Sub_Section, THE Keyword_Pipeline SHALL read all source keywords listed under the specified Sub_Section heading in the input file
2. IF the specified Sub_Section is absent from the input file or contains zero source keywords, THEN THE Keyword_Pipeline SHALL log an error identifying the missing or empty Sub_Section and terminate processing for that Sub_Section without creating or modifying any output file
3. WHEN processing a source keyword, THE Keyword_Pipeline SHALL generate exactly 3 distinct search terms derived from that keyword
4. WHEN the 3 search terms are generated, THE Keyword_Pipeline SHALL execute all 3 searches in parallel via the Search_Provider and wait for all 3 to return before proceeding
5. WHEN search results are returned, THE Keyword_Pipeline SHALL assign each result exactly one classification — accepted or rejected — evaluated against the configured research topic, and SHALL record a reason for each classification
6. IF more than 50% of the search results for a source keyword are classified as rejected, THEN THE Keyword_Pipeline SHALL generate exactly 1 replacement search term, execute one additional search and classification cycle for that term only, and SHALL perform at most 1 such retry per source keyword
7. IF a source keyword has zero accepted results after its retry is exhausted, THEN THE Keyword_Pipeline SHALL skip keyword generation for that source keyword, log the reason, and continue processing the remaining source keywords
8. WHEN at least 1 accepted result is available for a source keyword, THE Keyword_Pipeline SHALL generate exactly 10 new keywords for that source keyword, each distinct from the source keywords and from keywords already generated within the same Sub_Section
9. THE Keyword_Pipeline SHALL write generated keywords to a markdown file organized by Sub_Section, appending in batches of no more than 20 keywords per write operation, and SHALL write only complete batches without writing partial batches
10. THE Keyword_Pipeline SHALL treat source keyword files as read-only and produce output in a separate file

### Requirement 4: Idea Generation Pipeline

**User Story:** As a researcher, I want to automatically generate structured research outputs (ideas, summaries, analyses) from keywords using web research, so that I get actionable results backed by real-world context.

#### Acceptance Criteria

1. WHEN the Idea_Pipeline is invoked for a keyword file, THE Idea_Pipeline SHALL read all keywords from that file grouped by their Sub_Section.
2. IF the keyword file is missing, empty, or cannot be parsed into Sub_Sections, THEN THE Idea_Pipeline SHALL log an error identifying the cause and SHALL terminate without writing any output file.
3. WHEN processing a keyword, THE Idea_Pipeline SHALL generate exactly 3 search terms derived from that keyword.
4. WHEN the 3 search terms are generated, THE Idea_Pipeline SHALL execute all 3 searches concurrently via the Search_Provider.
5. WHEN search results are returned, THE Idea_Pipeline SHALL classify each result as either accepted or rejected based on its relevance to the configured research topic.
6. IF more than half of the search results for a keyword are classified as rejected, THEN THE Idea_Pipeline SHALL regenerate one search term and re-execute the searches, up to a maximum of one retry per keyword.
7. IF no accepted results remain for a keyword after the single retry is exhausted, THEN THE Idea_Pipeline SHALL skip Idea_Record generation for that keyword, log the reason, and continue processing the remaining keywords.
8. WHEN at least one accepted result is available for a keyword, THE Idea_Pipeline SHALL generate exactly 10 Idea_Records for that keyword.
9. THE Idea_Pipeline SHALL ensure each Idea_Record contains all fields defined in the configured output schema (default schema: title, description, and resume-worthiness, where resume-worthiness names at least one specific technology).
10. THE Idea_Pipeline SHALL write Idea_Records to the output file in batches of no more than 5 records per write operation.
11. THE Idea_Pipeline SHALL ensure no two Idea_Records in the same output file share the same title, using case-insensitive comparison, and SHALL drop any Idea_Record whose title already exists in that file.
12. THE Idea_Pipeline is ONE type of research workflow — the Research_Agent SHALL support defining other pipeline types (source completion, synthesis, comparison, etc.) via configuration.

### Requirement 5: Parallel Execution and Orchestration

**User Story:** As a user, I want multiple keywords and sub-sections to be processed simultaneously, so that the total enrichment time is reduced.

#### Acceptance Criteria

1. WHILE a Sub_Section is being processed, THE Orchestrator SHALL execute its keyword Tasks concurrently up to the configured maximum parallel task limit, queuing any additional ready keyword Tasks and starting them as running Tasks complete, such that the number of simultaneously running Tasks never exceeds the configured limit
2. THE Orchestrator SHALL execute Tasks from multiple independent Sub_Sections concurrently, where two Sub_Sections are independent when neither contains a Task that depends on a Task or output of the other, such that the total number of simultaneously running Tasks never exceeds the configured maximum parallel task limit
3. THE Config SHALL specify the maximum number of parallel Tasks as an integer greater than or equal to 1 (default: 3)
4. IF the configured maximum number of parallel Tasks is set to a value less than 1, THEN THE Research_Agent SHALL log an error identifying the invalid value and exit before processing any Task
5. WHEN the Keyword_Pipeline and the Idea_Pipeline target different, independent sections, THE Orchestrator SHALL run them concurrently, such that the total number of simultaneously running Tasks never exceeds the configured maximum parallel task limit
6. IF the Idea_Pipeline depends on output from the Keyword_Pipeline for the same section, THEN THE Orchestrator SHALL defer all Idea_Pipeline Tasks for that section until every Keyword_Pipeline Task for that section is completed with output written and verified
7. IF a Keyword_Pipeline Task that an Idea_Pipeline Task depends on has not completed successfully, THEN THE Orchestrator SHALL keep the dependent Idea_Pipeline Task deferred rather than starting it
8. THE Orchestrator SHALL attempt to determine, for each pair of Sub_Sections, whether they are independent (neither Sub_Section contains a Task depending on a Task or output of the other) before scheduling them
9. IF the Orchestrator cannot determine whether two Sub_Sections are independent, THEN THE Orchestrator SHALL present an explanation of the ambiguous dependency to the user and request the user's decision before scheduling those Sub_Sections, rather than guessing
10. WHILE the Orchestrator is awaiting the user's decision about an ambiguous dependency, THE Orchestrator SHALL NOT start either of the two Sub_Sections concurrently
11. WHEN the user resolves an ambiguous dependency, THE Orchestrator SHALL schedule the two Sub_Sections concurrently or sequentially according to the user's decision

### Requirement 6: State Persistence and Resumability

**User Story:** As a user, I want the system to resume from where it left off after a crash or interruption, so that I do not lose completed work or repeat expensive API calls.

#### Acceptance Criteria

1. THE State_Store SHALL use a database (SQLite minimum, with option to upgrade to PostgreSQL) to persist task status, dependencies, and retry history
2. THE State_Store SHALL track each Task with fields: id, type, status (pending/in_progress/completed/failed/retry_queued), dependencies, attempt_count, last_error, created_at, completed_at
3. WHEN the Research_Agent starts, THE State_Store SHALL load previously persisted state, skip Tasks whose status is completed, and reset any Task whose status is in_progress to pending so that it is re-executed
4. WHEN a Task's output is written to its target output file and confirmed present, THE State_Store SHALL set that Task's status to completed and record its completed_at timestamp
5. WHEN all Tasks in a Sub_Section are completed, THE State_Store SHALL mark that Sub_Section as completed
6. IF the Research_Agent is interrupted while a Task is in status in_progress, THEN THE State_Store SHALL retain that Task in a non-completed status so that, on the next run, the Task is re-executed rather than skipped
7. THE State_Store SHALL maintain a retry queue in which each failed Task is re-queued with status retry_queued and its attempt_count incremented by 1, up to the configured maximum attempt limit (default: 5)
8. WHERE a Task has unmet dependency Tasks, THE State_Store SHALL defer execution of that Task until all of its dependency Tasks have status completed
9. THE State_Store SHALL support querying pending Tasks, failed Tasks, Tasks filtered by section, and overall progress statistics (total Task count, completed count, failed count, pending count, retry-queued count, and percentage completed from 0 to 100)
10. WHEN a Task transitions between any of the statuses pending, in_progress, completed, failed, or retry_queued, THE State_Store SHALL commit the new status to the database before the Orchestrator initiates the next state transition, so that an interruption at any point preserves the most recent committed state
11. IF a Task's dependency set contains a cycle or references a Task that does not exist, THEN THE State_Store SHALL set that Task's status to failed and record an error in last_error indicating an unsatisfiable dependency, rather than deferring the Task indefinitely

### Requirement 7: Configuration Management

**User Story:** As a user, I want to configure providers, models, parallelism, and file paths in a single config file, so that I can adjust behavior without modifying code.

#### Acceptance Criteria

1. THE Config SHALL be loaded from a YAML file at a configurable path (default: `config.yaml` in the project root)
2. THE Config SHALL support specifying: LLM provider and model, search provider and credentials, maximum parallelism (integer from 1 to 10), input file paths, output directory structure, and retry limits (integer from 0 to 5)
3. IF a required configuration value is missing, THEN THE Research_Agent SHALL emit an error message identifying the missing field name and exit with a non-zero status code before initiating any research
4. THE Config SHALL resolve environment variable references for secrets and other values using the syntax `${ENV_VAR_NAME}`, substituting the value of the named environment variable
5. WHERE a configuration value is not explicitly set, THE Config SHALL apply that field's documented default value
6. WHEN no configuration file exists at the resolved path, THE Config SHALL initialize using documented default values without terminating
7. IF a provided configuration value falls outside its allowed set or defined numeric bounds, THEN THE Research_Agent SHALL emit an error message identifying the field name and the expected constraint, and exit with a non-zero status code before initiating any research
8. IF an environment variable referenced via `${ENV_VAR_NAME}` is not set, THEN THE Research_Agent SHALL emit an error message identifying the variable name and exit with a non-zero status code before initiating any research

### Requirement 8: CLI Interface

**User Story:** As a user, I want to control the research agent from the command line, so that I can run it headlessly in tmux or nohup sessions.

#### Acceptance Criteria

1. THE Research_Agent SHALL provide a CLI exposing the following subcommands: `run` (execute a specified pipeline on a specified section), `run-all` (execute all configured pipelines across all sections), `status` (display completion progress), and `reset` (clear persisted state for a specified section)
2. WHEN the `run` subcommand is invoked with a section identifier that exists in the loaded Config, THE Research_Agent SHALL execute only the pipeline for that section and SHALL leave the state of all other sections unchanged
3. WHEN the `status` subcommand is invoked, THE Research_Agent SHALL display, for each configured section, the number of completed Tasks and the total number of Tasks in the form completed/total
4. WHEN the `reset` subcommand is invoked with a section identifier that exists in the loaded Config, THE Research_Agent SHALL clear all persisted State_Store entries for that section so the section can be re-run, and SHALL leave the state of all other sections unchanged
5. IF the `run` or `reset` subcommand is invoked with a section identifier that does not exist in the loaded Config, THEN THE Research_Agent SHALL exit without modifying any persisted state and SHALL return an error indication identifying the unknown section identifier
6. THE Research_Agent SHALL execute every subcommand without requiring an attached TTY, writing log output as structured records that each include a timestamp, a log level, and an event description
7. THE Research_Agent SHALL persist its current progress to the database continuously as work proceeds, so that at any moment the database reflects the most recent completed work and the Research_Agent can resume from that point after any interruption
8. WHEN the Research_Agent receives a SIGINT or SIGTERM signal, THE Research_Agent SHALL stop accepting new work, finish persisting the in-progress state to the database, and exit, leaving any unfinished Task in a non-completed status so it is resumed on the next run
9. IF the Research_Agent is terminated abruptly without a graceful signal (for example, power loss, network loss, or process kill), THEN on the next run THE Research_Agent SHALL resume from the last state persisted in the database without repeating already-completed work

### Requirement 9: Output Management and Multi-Format Export

**User Story:** As a user, I want results in multiple formats (Markdown, JSON, PDF, DOCX), so that I can use them in different contexts.

#### Acceptance Criteria

1. THE Output_Formatter SHALL support exactly four output formats: Markdown (default, used for development and quick iteration), JSON (structured data), PDF (reports), and DOCX (documents)
2. WHEN the Config specifies one or more output formats, THE Research_Agent SHALL generate an output file for each specified format from the same internal research data
3. IF the Config specifies no output format, THEN THE Research_Agent SHALL default to generating Markdown output
4. IF the Config specifies an output format that is not one of the four supported formats (Markdown, JSON, PDF, DOCX), THEN THE Research_Agent SHALL reject the configuration at load time and return an error indicating the unsupported format, without generating any output files
5. THE Research_Agent SHALL write each output file using the path pattern `{output_dir}/{section_name}/{subsection_name}.{format_extension}`
6. THE Research_Agent SHALL include the following metadata in each output file: source file paths, sub-section name, generation date, and search queries used, embedded as YAML frontmatter for Markdown, as dedicated metadata fields for JSON, and as a header section for PDF and DOCX
7. WHEN an existing output file at the target path contains more than 100 characters and no force flag is provided, THE Research_Agent SHALL preserve the existing file unchanged and SHALL not write the new content
8. WHEN an existing output file is preserved because no force flag was provided, THE Research_Agent SHALL emit a warning indicating which file path was preserved and was not overwritten
9. WHEN an existing output file at the target path contains more than 100 characters and a force flag is provided, THE Research_Agent SHALL overwrite the existing file with the new content
10. WHEN the output directory specified in the target path does not exist, THE Research_Agent SHALL create the output directory, including any missing parent directories, before writing the output file

### Requirement 10: Error Handling, Retry, and Task Recovery

**User Story:** As a user, I want failed tasks to be retried and re-queued (not abandoned), because a failed task may have downstream dependencies that depend on its completion.

#### Acceptance Criteria

1. WHEN an LLM call fails, THE Research_Agent SHALL retry the call using exponential backoff that starts at a 1-second initial delay and multiplies the delay by the configured backoff multiplier (default: 2.0) before each subsequent retry, up to the configured immediate retry limit (default: 3 retries).
2. WHEN a search call fails, THE Research_Agent SHALL retry the search call exactly once after a 3-second delay, and IF the retried call also fails, THEN THE Research_Agent SHALL mark that search as failed.
3. IF a Task fails after its immediate retries are exhausted, THEN THE Research_Agent SHALL set that Task's status to `retry_queued` and add it to the retry queue without discarding it.
4. WHEN all other pending Tasks in the current batch have completed, THE Research_Agent SHALL re-attempt each Task whose status is `retry_queued` by granting it a fresh set of immediate retries with the backoff delay reset to the 1-second initial delay, while preserving that Task's cumulative attempt count across retry-queue cycles.
5. WHERE a Task has one or more downstream dependency Tasks, THE Research_Agent SHALL re-attempt that Task across retry-queue cycles until it succeeds or its cumulative attempt count reaches the configured maximum total attempts (default: 5).
6. THE Research_Agent SHALL increment and persist a cumulative attempt count for each Task that combines immediate retries and retry-queue cycles.
7. WHEN a run completes, THE Research_Agent SHALL produce a summary report listing the count of total Tasks processed, successful Tasks, failed Tasks, retry-queued Tasks, and Tasks blocked by dependency.
8. THE Config SHALL allow configuring retry behavior including: immediate retry limit (default: 3), backoff multiplier (default: 2.0), initial backoff delay (default: 1 second), maximum total attempts (default: 5), and whether to halt on failure or continue with the retry queue (default: halt on failure).
9. IF a Task's cumulative attempt count reaches the configured maximum total attempts (default: 5) without the Task succeeding, THEN THE Research_Agent SHALL log the failure at ERROR level, halt the dispatch of new Tasks to pause the pipeline, and retain all persisted state so the run can be resumed or the Task skipped through manual intervention.
10. WHEN a Task that has one or more downstream dependency Tasks is escalated after reaching the configured maximum total attempts, THE Research_Agent SHALL set each of its downstream dependency Tasks to status `blocked_by_dependency`.

### Requirement 11: Progress Reporting and Logging

**User Story:** As a user running the agent in the background, I want structured logs and progress tracking, so that I can monitor execution without attaching to the process.

#### Acceptance Criteria

1. THE Research_Agent SHALL log pipeline start, pipeline end, keyword processing start, keyword processing end, errors, and retries, each annotated with an ISO 8601 timestamp and the originating component name
2. THE Research_Agent SHALL support exactly four configurable log levels — debug, info, warning, and error — and SHALL default to info when no level is configured
3. THE Research_Agent SHALL write each log entry to both stdout and a log file, where the log file is capped at 10 MB and retains at most 3 rotated backups
4. WHEN a Sub_Section is completed, THE Research_Agent SHALL log a progress summary stating the count of completed items and the total number of items for that Sub_Section
5. THE Research_Agent SHALL maintain a machine-readable JSON progress file that external tools can poll, containing for each Sub_Section its name, completed item count, total item count, and status (one of pending, running, done, or error), plus a last_updated ISO 8601 timestamp
6. WHEN a Sub_Section item is completed, THE Research_Agent SHALL update the progress file with the new completed count and a refreshed last_updated timestamp
7. IF writing the progress file fails, THEN THE Research_Agent SHALL log a warning-level message and continue pipeline execution without terminating
8. IF the configured log level is not one of debug, info, warning, or error, THEN THE Research_Agent SHALL reject the configuration and emit an error message indicating the invalid value and the allowed values

### Requirement 12: Extensibility for New Research Topics

**User Story:** As a user, I want to use this system for any research topic (not just AI engineering), so that the tool is reusable across different projects.

#### Acceptance Criteria

1. THE Research_Agent SHALL accept the research topic as a required configuration parameter (a non-empty string of 1 to 200 characters) and optional context as a list of 0 to 50 subtopic strings (each 1 to 200 characters), and SHALL inject these configured values into all generated prompts so that changing the topic requires only a configuration change
2. THE Config SHALL support custom prompt templates for the keyword generation step and the idea generation step, each able to reference the configured research topic and subtopics
3. WHEN a custom prompt template is provided for a generation step, THE Research_Agent SHALL use that custom template for that step instead of the default template
4. WHEN no custom prompt template is provided for a generation step, THE Research_Agent SHALL use the default template for that step
5. IF a provided custom prompt template cannot be parsed or references a variable that is not supplied, THEN THE Research_Agent SHALL reject the configuration at load time with an error identifying the offending template and SHALL NOT begin processing tasks
6. THE Config SHALL support defining the Idea_Record structure as an ordered set of 1 to 20 field definitions, each specifying a field name (1 to 64 characters) and whether the field is required, and SHALL apply the default fields (title, description, resume-worthiness) when no structure is defined
7. IF the configured Idea_Record structure is invalid (a missing or empty field name, a duplicate field name, or fewer than 1 or more than 20 fields), THEN THE Research_Agent SHALL reject the configuration at load time with an error identifying the offending field and SHALL NOT begin processing tasks

### Requirement 13: Data Layer Architecture

**User Story:** As a developer, I want a clear data layer with appropriate storage technologies for different data types, so that the system is performant, queryable, and scalable.

#### Acceptance Criteria

1. THE Research_Agent SHALL use SQLite (upgradeable to PostgreSQL) as the primary relational database for: task state, task dependencies, retry history, run metadata, and progress tracking
2. THE Research_Agent SHALL use a Cache_Layer (in-memory plus optional Redis) to cache LLM responses keyed by the exact prompt text and model identifier, and to cache search results keyed by the exact query string, returning a cached entry only when its age is within the configured TTL
3. WHERE a Vector_Store (ChromaDB or similar) is enabled, THE Research_Agent SHALL use it for semantic deduplication of generated content, finding related ideas across sections, and clustering keywords, treating two items as semantically related only when their similarity score meets or exceeds a configurable threshold
4. WHERE a Graph_Database (Neo4j, or NetworkX for local) is enabled, THE Research_Agent SHALL use it for: mapping relationships between keywords, ideas, topics, and sources, visualizing research dependency trees, and finding connection paths between disparate topics
5. THE Config SHALL specify which data stores are active, where only SQLite and Cache_Layer are required and Vector_Store and Graph_Database are optional extensions
6. THE Cache_Layer SHALL apply a configurable TTL to each cached item, accepting values between 60 seconds and 30 days, defaulting to 24 hours for search results and 7 days for LLM responses
7. WHEN the Research_Agent runs and the primary relational database does not yet exist, THE Research_Agent SHALL create and initialize it in code (not via the LLM) before processing tasks
8. WHEN the Research_Agent runs and the existing database schema is out of date, THE Research_Agent SHALL take a backup of the existing database and then apply migrations to that same existing database, preserving prior data rather than deleting and recreating the database
9. WHEN the Research_Agent requests an LLM response or search result for which no matching cached entry exists, or the matching entry's age exceeds its configured TTL, THE Cache_Layer SHALL treat the request as a cache miss, allow the corresponding API call or search to proceed, and store the returned result with a new TTL
10. IF applying a migration fails, THEN THE Research_Agent SHALL stop before processing any tasks, leave the existing database unchanged so it can be restored from the backup, and report an error indicating the failure reason
11. IF the primary relational database is present but corrupted or in an unexpected state that is neither a clean first run nor a known out-of-date schema, THEN THE Research_Agent SHALL stop before processing any tasks and report a clear error stating that the database requires manual intervention, rather than attempting automatic repair or recreation
12. IF an enabled optional data store (Vector_Store or Graph_Database) is unavailable during a run, THEN THE Research_Agent SHALL continue processing tasks without that store's features and report an error indicating which data store is unavailable

### Requirement 14: Adversarial Review Agent

**User Story:** As a researcher, I want a separate review agent (preferably from a different model family) to validate and critique generated outputs, so that I catch hallucinations, weak evidence, and low-quality results before they are finalized.

#### Acceptance Criteria

1. THE Research_Agent SHALL provide an optional Reviewer_Agent, disabled by default, that validates outputs produced by the Executor_Agent before they are finalized
2. WHERE the Reviewer_Agent is enabled, WHEN the Executor_Agent produces a generated batch of keywords or ideas, THE Orchestrator SHALL route every item in that batch through the Reviewer_Agent before marking the corresponding task as completed
3. WHERE the Reviewer_Agent is enabled, THE Config SHALL allow specifying an LLM model or model family for review that differs from the one used by the Executor_Agent to avoid self-reinforcing biases
4. WHEN the Reviewer_Agent evaluates an output item, THE Reviewer_Agent SHALL assign exactly one classification of ACCEPT, REVISE, or REJECT to that item and SHALL produce non-empty feedback text identifying the specific issues to be corrected for any item classified as REVISE or REJECT
5. WHEN items are marked REVISE and the number of completed revision cycles for that item is below the configured maximum, THE Orchestrator SHALL return them to the Executor_Agent together with the Reviewer_Agent's feedback for regeneration and SHALL increment that item's revision cycle count
6. IF an item remains classified as REVISE after the configured maximum number of revision cycles is reached, THEN THE Orchestrator SHALL discard that item, exclude it from the finalized output, and log the reason for discarding
7. WHEN an item is marked REJECT, THE Orchestrator SHALL discard that item, exclude it from the finalized output, and log the rejection reason
8. WHERE the Reviewer_Agent classifies an item as ACCEPT, THE Orchestrator SHALL treat ACCEPT as a quality approval only, and SHALL finalize the item only if it also satisfies the structural rules defined elsewhere in this specification (no duplicate title within the same output file per Requirement 4, and conformance to the configured Idea_Record schema per Requirement 12); an item that is ACCEPT but violates a structural rule SHALL be discarded with the reason logged
9. THE Config SHALL specify whether review is enabled (default disabled), the model to use for review, and the maximum number of revision cycles (an integer from 0 to 5, default 2)
10. IF the Reviewer_Agent's model is unreachable or returns no response while reviewing an item, THEN THE Orchestrator SHALL retry the review up to the configured retry limit, and once all retries are exhausted SHALL log the review failure and retain the item flagged as unreviewed rather than discarding it
11. IF the Reviewer_Agent returns a response that does not contain a valid classification of ACCEPT, REVISE, or REJECT, THEN THE Orchestrator SHALL treat that item as REVISE for the current revision cycle and log the invalid-response reason
12. WHEN the review configuration is loaded and review is enabled, THE Research_Agent SHALL validate that a review model is specified and that the maximum number of revision cycles is an integer within the range 0 to 5, and IF validation fails THEN THE Research_Agent SHALL reject the configuration and log an error indicating the invalid review setting

### Requirement 15: Markdown-Defined Skills and Workflows

**User Story:** As a user, I want to define agent behaviors (skills) and research workflows as simple Markdown files, so that I can create, share, and modify them without touching code.

#### Acceptance Criteria

1. WHEN the Research_Agent starts, THE Research_Agent SHALL load every Markdown file with a `.md` extension from the configured skills directory (default: `skills/`) as a Skill definition.
2. THE Research_Agent SHALL require each Skill file to define a non-empty skill name, a non-empty description, an input schema, an output schema, and a prompt template, and SHALL treat tool requirements as optional.
3. WHEN the Research_Agent starts, THE Research_Agent SHALL load every Markdown file with a `.md` extension from the configured workflows directory (default: `workflows/`) as a Research_Workflow definition.
4. THE Research_Agent SHALL require each Research_Workflow file to define a non-empty workflow name, an ordered sequence of one or more steps, the skill referenced by each step, the input/output mappings between steps, and the dependency rules between steps.
5. WHEN the Research_Agent starts, THE Research_Agent SHALL validate every loaded Skill and Research_Workflow file and, for each validation failure, produce an error message that identifies the source file path, the specific missing or invalid field or reference, and the reason for the failure.
6. IF a Research_Workflow step references a skill name that is not present among the loaded Skill definitions, or a Research_Workflow defines a circular dependency between its steps, THEN THE Research_Agent SHALL treat that workflow as invalid and produce an error message identifying the source file path and the unresolved reference or circular dependency.
7. IF any Skill or Research_Workflow file fails validation at startup, THEN THE Research_Agent SHALL halt startup without processing any research request, rather than continue with the invalid files.
8. WHEN a Skill or Research_Workflow file is added, modified, or removed in the configured directory, THE Research_Agent SHALL detect the change and reload the affected definitions within 5 seconds without requiring a restart.

### Requirement 16: Self-Aware Framework — Conversational Skill and Workflow Creation

**User Story:** As a user, I want to create new skills and workflows by chatting with the system in natural language, so that I don't need to manually write Markdown files or understand the internal schema.

#### Acceptance Criteria

1. THE Research_Agent SHALL provide a conversational CLI chat mode in which the user can describe a desired Skill or Research_Workflow in natural language and exchange multiple turns within a single session.
2. WHEN the user describes a new Skill in chat mode, THE Research_Agent SHALL first check the internal registry for an existing Skill that already provides the described capability before generating anything
3. IF an existing Skill already provides the described capability, THEN THE Research_Agent SHALL inform the user that it already exists and offer the user the choice to use the existing Skill as-is, modify the existing Skill, or create a different variant alongside it, and SHALL act according to the user's choice
4. WHERE no existing Skill provides the described capability, WHEN the user describes a new Skill, THE Research_Agent SHALL generate a Skill Markdown file containing all required schema elements (name, description, input/output schema, and prompt template) and save it to the skills directory, then notify the user that the Skill was created and where it was saved
5. WHEN the user describes a new Research_Workflow in chat mode, THE Research_Agent SHALL first check the internal registry for an existing Research_Workflow that already provides the described capability, and IF one exists THEN THE Research_Agent SHALL inform the user and offer the choice to use it as-is, modify it, or create a different variant, otherwise THE Research_Agent SHALL generate a Research_Workflow Markdown file containing all required schema elements (name, description, and ordered steps), save it to the workflows directory, and notify the user that the Research_Workflow was created and where it was saved
6. WHILE in chat mode after a Skill or Research_Workflow has been created, THE Research_Agent SHALL allow the user to continue the conversation to improve it, remove parts of it, or delete it entirely, and SHALL apply each requested change to the saved file
7. IF a generated Skill or Research_Workflow Markdown file fails schema validation, THEN THE Research_Agent SHALL regenerate it up to 2 additional times, and if validation still fails after those attempts, THE Research_Agent SHALL report an error indicating the validation failure and SHALL save no file.
8. IF the generated Skill or Research_Workflow references a tool that is not installed, THEN THE Research_Agent SHALL reject the generated file with an error identifying the unavailable tool and SHALL save no file.
9. THE Research_Agent SHALL maintain an internal registry of its currently available skills, workflows, agents, and tools, and SHALL include that registry in the context used when checking for existing capabilities and generating new skills and workflows.
10. WHEN the user asks the system to describe itself in chat mode, THE Research_Agent SHALL respond with a summary of its available skills, available workflows, and referenceable tools drawn from the internal registry.

### Requirement 17: MCP/Tool/Workflow Discovery and Auto-Integration

**User Story:** As a user, I want the system to proactively search for and suggest useful MCP servers, tools, and community workflows that could enhance my research, so that I benefit from the ecosystem without manual discovery.

#### Acceptance Criteria

1. WHEN the user requests tool discovery or a scheduled discovery scan begins, THE Research_Agent SHALL search the configured external registries (MCP registry, GitHub, package indexes) for tools and workflows matching the current research topic, performing no more than 10 queries per hour per registry
2. WHEN the Research_Agent identifies a tool, MCP server, or workflow whose relevance score is at or above the configured relevance threshold (default 0.5 on a 0.0 to 1.0 scale) and that is not present in the approved or rejected history, THE Research_Agent SHALL present a recommendation including its name, source, description, the expected improvement to the research, and the steps required to integrate it
3. THE Research_Agent SHALL NOT install, configure, or integrate any external tool, MCP server, or workflow without explicit user confirmation
4. WHEN the user explicitly approves a recommended integration, THE Research_Agent SHALL run its installation, register its configuration into the tool registry, and verify the tool is accessible before reporting completion to the user
5. IF installation, configuration, or accessibility verification of an approved integration fails, THEN THE Research_Agent SHALL abort the integration, restore the tool registry to its state prior to the integration attempt, and notify the user with an error indication describing the failure
6. IF a registry cannot be reached due to network unavailability or a registry error, THEN THE Research_Agent SHALL skip that registry, continue searching the remaining available registries, and notify the user that the affected registry was unavailable
7. WHEN the user approves or rejects a recommendation, THE Config SHALL persist a record of that decision including tool name, source, status (approved or rejected), and timestamp, and THE Research_Agent SHALL exclude any tool whose latest recorded status is rejected from future recommendations

### Requirement 18: Dynamic Agent Creation and Lifecycle

**User Story:** As a user, I want the system to create specialized agents on-the-fly based on task needs, so that each research task gets an optimally configured agent without manual setup.

#### Acceptance Criteria

1. WHEN a research task requires specialized behavior that no existing agent provides, THE Research_Agent SHALL create a new specialized agent (e.g., a "FinTech keyword expert" for domain-specific enrichment) defined by a name, role, model, tool list, and prompt.
2. WHEN creating a dynamic agent, THE Research_Agent SHALL present the complete agent definition (name, role, model, tools, prompt) to the user and SHALL withhold activation of the agent until the user explicitly approves the definition.
3. IF the user rejects a presented agent definition, THEN THE Research_Agent SHALL NOT activate the agent, SHALL discard the proposed definition, and SHALL notify the user that agent creation was cancelled.
4. IF creating a temporary agent would cause the number of concurrent temporary agents to exceed 5, THEN THE Research_Agent SHALL reject the creation request, SHALL leave all existing temporary agents unchanged, and SHALL notify the user that the maximum of 5 concurrent temporary agents has been reached.
5. THE Config SHALL support two agent lifecycle modes: temporary, in which the agent is destroyed when its associated task or pipeline completes, and persistent, in which the agent definition is saved for reuse across sessions.
6. WHEN the user requests a lifecycle change for an agent, THE Research_Agent SHALL promote a temporary agent to persistent by saving its definition for reuse, demote a persistent agent by archiving its definition, or delete a persistent agent by removing its definition, according to the requested action.
7. WHEN an agent is used for a task, THE State_Store SHALL record the agent identifier, the associated task identifier, the agent lifecycle mode, and the creation timestamp so that the agent-to-task history can be audited.
8. IF a temporary agent remains active longer than its configured timeout (default 30 minutes), THEN THE Research_Agent SHALL destroy the temporary agent, SHALL release its associated resources, and SHALL notify the user that the agent was terminated due to timeout.

### Requirement 19: Per-Research Wiki with Selective Inclusion

**User Story:** As a researcher, I want each research project to have its own wiki that accumulates findings, decisions, and context over time, but I want control over what goes in.

#### Acceptance Criteria

1. THE Research_Agent SHALL maintain a per-research-project Wiki, stored as structured Markdown or in the database, that accumulates entries categorized as: key findings, decisions made, sources consulted, rejected approaches, and cross-references between topics, where each entry SHALL NOT exceed the configured maximum entry size (default 2,000 characters)
2. WHEN a pipeline produces a finding whose confidence score is greater than or equal to the configured notability threshold (range 0.0 to 1.0, default 0.8), THE Research_Agent SHALL propose adding that finding to the Wiki as a single entry
3. WHERE auto-wiki mode is disabled, WHEN the Research_Agent proposes a Wiki entry, THE Research_Agent SHALL add the entry to the Wiki only after the user explicitly approves it, and SHALL discard the proposed entry and leave the Wiki unchanged if the user rejects it
4. WHERE auto-wiki mode is enabled, WHEN the Research_Agent proposes a Wiki entry, THE Research_Agent SHALL add the entry to the Wiki without requiring user confirmation
5. WHEN the user queries the Wiki (e.g., "what do we know about X?"), THE Research_Agent SHALL return the matching Wiki entries ranked by relevance, up to the configured maximum query result count (default 5)
6. WHEN the Orchestrator starts a research run, THE Orchestrator SHALL inject the relevant Wiki entries for the run topic into agent prompts, limited to the configured maximum context entry count (default 10), to avoid re-discovering known information
7. THE Config SHALL specify: Wiki storage location, storage backend (Markdown or database), auto-wiki mode (enabled/disabled), notability threshold (0.0 to 1.0), maximum entry size in characters, maximum query result count, maximum context entry count, and maximum Wiki size (entry count) before archiving the oldest entries
8. IF writing a proposed entry to the Wiki fails, THEN THE Research_Agent SHALL return an error indicating that the addition failed and SHALL leave the existing Wiki contents unchanged
9. IF a Wiki query matches no entries, THEN THE Research_Agent SHALL return an empty result set with an indication that no matching entries were found
10. WHEN the number of active Wiki entries reaches the configured maximum Wiki size, THE Research_Agent SHALL archive the oldest entries so that the active Wiki size does not exceed the configured maximum
