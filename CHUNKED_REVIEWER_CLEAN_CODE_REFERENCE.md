# ChunkedReviewer Clean Code Reference

## Purpose
This document is the deep reference for cleaning `backend/app/review/services/chunked_reviewer.py` while keeping the current chunked-review behavior intact.

The main goal is to understand:
- what the file currently contains
- what responsibilities are mixed together
- where this file sits in the review pipeline
- which parts are true chunked-review core logic and which parts are good extraction candidates

## File Facts
- File: `backend/app/review/services/chunked_reviewer.py`
- Size: 1402 lines
- Main top-level contents:
  - `ChunkReviewResult` dataclass
  - `MergedIssue` dataclass
  - `ChunkedReviewResult` dataclass
  - `ChunkedReviewer` class
- Primary role:
  - run the large-PR review pipeline after `AIReviewer` decides chunking is needed

## Where This File Fits in the System
High-level flow:

1. `AIReviewer.analyze_mr()` decides the PR is too large.
2. `AIReviewer._chunked_review()` prepares shared inputs:
   - JIRA context
   - framework rules
   - round templates
   - system prompt
   - repository ID
3. `AIReviewer._chunked_review()` creates `ChunkedReviewer(...)`.
4. `ChunkedReviewer.review_large_pr()` runs the chunked pipeline:
   - R0 context lookup/extraction
   - chunk splitting
   - parallel chunk review
   - merge
   - global R3 verification
   - final result assembly
5. `ChunkedReviewer` returns `ChunkedReviewResult`.
6. `AIReviewer` unwraps that into the normal analysis shape expected by `review_tasks.py`.

So this file is the large-PR execution engine inside the broader AI review system.

## Core Dependencies

### Direct dependencies
- `ChunkManager`, `ReviewChunk`, `FileChange`, `SkippedFile`
- `settings`
- `AIReviewer` via `TYPE_CHECKING`

### Runtime dependencies reached indirectly
- `app.r0_constructore.services.r0_context_scheduler`
  - `get_cached_context`
  - `store_context_to_cache`
- multiple `AIReviewer` methods, including private ones

## What the File Contains Today

## 1. Result/Data Structures

### `ChunkReviewResult`
Represents the result of one chunk’s R1+R2 review.

Fields:
- `chunk_index`
- `session_id`
- `issues`
- `r1_raw`
- `r2_raw`
- `success`
- `error`
- `retry_count`
- `processing_time`

Current observation:
- `r2_raw` is used later for merged non-issue data and for collecting `description` and `flow_diagram`
- `r1_raw` is currently stored but not meaningfully used

### `MergedIssue`
Represents a normalized issue that moves from chunk review into global R3 verification.

Fields:
- `file`
- `line`
- `category`
- `severity`
- `issue`
- `suggestion`
- `source_chunk`

This is the transition structure between:
- chunk-level results
- verification-level input

### `ChunkedReviewResult`
Represents the final output of the whole chunked-review pipeline.

Fields:
- `merged_analysis`
- `chunk_count`
- `successful_chunks`
- `failed_chunks`
- `total_time_seconds`
- `parallel_workers`
- `session_id`
- `skipped_files`
- `verification_stats`
- `error`

This is the main return type of `review_large_pr()`.

## 2. Main Pipeline Orchestration

### `ChunkedReviewer.__init__`
Stores all shared inputs for the pipeline:
- `ai_reviewer`
- MR details
- JIRA context
- framework rules context
- templates for R1, R2, verification, extractor
- system prompt
- parallelism config
- repository ID

Also:
- mutates `round1_template` to strip diagram requirements when MR is not a feature MR
- reads retry settings from `settings`

This means constructor logic is not only assignment; it already contains prompt mutation behavior.

### `review_large_pr()`
This is the single biggest method and the heart of the file.

It currently does all of this:

1. Generates master session ID and logs pipeline header.
2. Resolves R0 context:
   - checks cache
   - runs extractor if cache is missing
   - stores extractor output back to cache
   - formats global context for prompts
   - resets CLAUDE stage to `review`
3. Runs chunk creation through `ChunkManager`.
4. Enforces `MAX_AGENT_CALLS` by truncating chunk count.
5. Runs parallel chunk reviews.
6. Separates success/failure chunk results.
7. Merges issues.
8. Merges non-issue arrays.
9. Runs global R3 verification in 2 rounds.
10. Injects merged non-issue data into final analysis.
11. Calls `ai_reviewer._deduplicate_issues(...)`.
12. Builds chunk-review stats metadata.
13. Collects `description` and `flow_diagram` from chunk results.
14. Builds `ChunkedReviewResult`.
15. Logs final timing/statistics summary.

This one method is doing:
- context resolution
- policy enforcement
- execution control
- merge control
- formatting of final metadata
- reporting/logging

That is the main clean-code pressure point in the file.

## 3. Parallel Chunk Execution

### `_review_chunks_parallel()`
Responsibilities:
- creates semaphore with `self.max_parallel`
- launches one coroutine per chunk
- gathers results
- converts exceptions into failed `ChunkReviewResult`

This is good orchestration logic and feels like true chunk-review core behavior.

### `_review_single_chunk_with_retry()`
Responsibilities:
- retries chunk review
- records timing and retry count
- short-circuits on usage/rate-limit errors

This is also core chunk execution behavior, but it mixes:
- retry policy
- error classification
- result shaping

### `_review_single_chunk()`
Responsibilities:
- build chunk diff
- build chunk R1 prompt
- call `ai_reviewer.run_two_round_review_async(...)`
- normalize issues from analysis
- return `ChunkReviewResult`

Important design detail:
- chunk R1+R2 execution is delegated back to `AIReviewer`
- so this file is not independent; it is tightly coupled to the reviewer facade

## 4. Chunk Prompt Building

### `_prepare_chunk_diff()`
Extracts only the relevant file-diff sections for the current chunk from the combined unified diff.

### `_build_file_changes_text()`
Recreates the same file-changes markdown format used by `AIReviewer.create_review_prompt()`.

### `_build_r1_prompt()`
Formats the round 1 prompt template using:
- MR title/source/target
- chunk file count
- file changes text
- JIRA context
- framework rules context

Fallback path:
- `self.ai_reviewer._get_fallback_prompt()`

Important observation:
- this file duplicates prompt-formatting behavior that already lives conceptually near `AIReviewer.create_review_prompt()`
- it also reaches into a private fallback method on `AIReviewer`

## 5. Issue Normalization and Merge Logic

### `_extract_issues_from_analysis()`
Pulls issue arrays from:
- `critical_issues`
- `bugs`
- `security_concerns`
- `performance_notes`
- `code_quality`
- `integration_issues`
- `testing_gaps`

### `_normalize_issue()`
Normalizes different issue shapes into a common structure.

### `_merge_chunk_results()`
Converts normalized issue dicts into `MergedIssue` objects.

Important observation:
- despite the module docstring mentioning pure-Python deduplication, this method does not deduplicate
- it simply concatenates all issues from all successful chunks
- real deduplication happens later through `ai_reviewer._deduplicate_issues(...)`

This is a small design/documentation mismatch.

### `_merge_non_issue_data()`
Merges:
- `positive_aspects`
- `recommendations`
- `testing_gaps`
- `integration_issues`

It uses ad hoc dedup keys based on text snippets.

This is important business logic, but it is a pure merge/helper concern rather than orchestration.

## 6. Global R3 Verification

### `_run_global_verification()`
Runs the global verification stage after chunk issues are merged.

It currently does all of this:
- handles empty input
- sets CLAUDE stage to `verification`
- creates verification session UUID
- builds R3 round 1 prompt
- runs R3 round 1
- parses round 1 result
- builds R3 round 2 self-critique prompt
- resumes the same session for round 2
- parses round 2 result
- combines verification stats

This is a second large orchestration method inside the same class.

### `_build_verification_prompt()`
Builds round 1 R3 prompt using:
- merged issues converted to analysis-like structure
- `ai_reviewer._create_issues_for_verification(...)`
- optional R0 global context
- verification agent template or fallback

Important observation:
- it depends on `AIReviewer` private helper `_create_issues_for_verification`
- it also depends on `AIReviewer` private verification fallback prompt

### `_build_verification_round2_prompt()`
Builds self-critique verification prompt using:
- round 1 analysis
- round 1 stats
- same verification template

Again depends on:
- `ai_reviewer._create_issues_for_verification(...)`
- `ai_reviewer._get_fallback_verification_agent_prompt()`

### `_run_verification_round()`
Runs verification using `AIReviewer` helpers:
- preferred: `ai_reviewer._run_verification_session(...)`
- fallback: `ai_reviewer._run_claude_with_session(...)`

Important observation:
- this file relies on private execution methods of `AIReviewer`
- the `hasattr(...)` check is a clue that boundaries are weak here

### `_parse_verification_result()`
Parses two possible R3 output shapes:
- format 1: `verified_issues` / `rejected_issues`
- format 2: direct categorized issue arrays

It also preserves:
- `integration_issues`
- `testing_gaps`
- `positive_aspects`
- `recommendations`
- `summary`
- `overall_score`

This is domain translation logic, not pipeline control.

## 7. Analysis/Result Conversion Helpers

### `_convert_merged_to_analysis_for_verification()`
Converts `MergedIssue` objects into analysis-shaped dicts by category.

### `_build_analysis_from_verified()`
Builds final analysis after successful verification.

### `_convert_to_analysis()`
Builds fallback analysis when verification fails and merged issues are returned directly.

### `_create_empty_analysis()`
Creates empty standard analysis shape.

### `_create_empty_result()`
Creates empty `ChunkedReviewResult`.

These are pure result-shaping helpers and strong extraction candidates.

## Current Strengths Worth Preserving

1. The overall pipeline shape is clear and meaningful:
   - R0 → split → parallel R1/R2 → merge → global R3 → final output
2. Parallel chunk review uses bounded concurrency via semaphore.
3. Chunk review still reuses `AIReviewer` for actual R1/R2 execution instead of re-implementing it.
4. Global verification after chunk merge is a strong design choice for reducing false positives.
5. The result objects (`ChunkReviewResult`, `MergedIssue`, `ChunkedReviewResult`) make the pipeline easier to follow than raw dicts alone.
6. Failure paths are explicitly handled:
   - empty chunks
   - all chunks failed
   - verification failed
   - rate-limit short-circuit

## Main Clean-Code Problems in This File

### 1. Too many responsibilities in one class
`ChunkedReviewer` currently owns:
- pipeline orchestration
- context resolution
- chunk prompt building
- chunk issue normalization
- chunk result merging
- verification prompt building
- verification execution
- verification result parsing
- final analysis/result conversion

That is too much for a single service class.

### 2. Very tight coupling to `AIReviewer`
This file depends on many `AIReviewer` methods, including private ones:

- `run_extractor_agent`
- `set_claude_md_for_stage`
- `format_global_context_for_prompt`
- `run_two_round_review_async`
- `_deduplicate_issues`
- `_create_issues_for_verification`
- `_get_fallback_prompt`
- `_get_fallback_verification_agent_prompt`
- `_run_verification_session`
- `_run_claude_with_session`

This is the biggest structural issue in the file.

It means `ChunkedReviewer` is not really a clean collaborator. It is partially an extension of `AIReviewer` that reaches into internal methods.

### 3. `review_large_pr()` is overloaded
This method is the equivalent of a pipeline script, coordinator, context resolver, and summary reporter combined into one block.

### 4. Prompt building is duplicated here
`_build_file_changes_text()` and `_build_r1_prompt()` recreate logic that already exists conceptually around `AIReviewer.create_review_prompt()`.

### 5. Verification logic is too broad
Prompt building, LLM execution, parsing, stats calculation, and final analysis shaping are all together under the same class.

### 6. Result shaping and orchestration are mixed
Helpers like:
- `_build_analysis_from_verified()`
- `_convert_to_analysis()`
- `_create_empty_analysis()`
- `_create_empty_result()`

are pure transformation helpers and do not need to live inside the orchestration class.

### 7. Some dead/stale details exist
Examples from the file:
- unused imports: `os`, `json`, `functools`, `ChunkingResult`
- assigned but unused variables: `context_source`, `extractor_output`
- `r1_raw` is stored but not actually used in the current flow
- `_run_global_verification(..., combined_diff, chunks)` accepts parameters that are not meaningfully used for behavior
- module comments mention merge deduplication, but `_merge_chunk_results()` just combines all issues

These are good signs of cleanup opportunities without changing logic.

## What This File Should Eventually Be
After cleanup, `ChunkedReviewer` should ideally remain the chunked-review orchestrator only:

- coordinate the large-PR pipeline
- delegate prompt building
- delegate context resolution
- delegate verification handling
- delegate analysis/result conversion

It should keep the chunked-review order and policies, but stop owning all of the supporting mechanics.

## Most Important Takeaway
`chunked_reviewer.py` contains strong chunked-review behavior, but it mixes orchestration with many helper concerns and depends too deeply on `AIReviewer` internals. The clean-code mission for this file is mainly about extracting helpers and replacing private cross-class calls with shared collaborators, not about changing the pipeline itself.
