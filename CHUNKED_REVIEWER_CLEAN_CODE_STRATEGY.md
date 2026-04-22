# ChunkedReviewer Clean Code Strategy

## Goal
Refactor `backend/app/review/services/chunked_reviewer.py` into a cleaner orchestration file while preserving the existing chunked-review behavior and output shape.

The key rule for this file is:

> preserve the pipeline, extract the helpers

This file already contains valuable logic. The goal is not to redesign chunked review. The goal is to keep the old logic and move each concern into a better home.

## Non-Negotiable Invariants
These behaviors should stay unchanged during cleanup:

1. Pipeline order must remain:
   - R0 context
   - chunk splitting
   - parallel chunk R1/R2
   - merge
   - global R3 verification
   - final analysis
2. `ChunkedReviewer.review_large_pr()` must still be the public entry point.
3. Return type must remain `ChunkedReviewResult`.
4. `_chunked_review_stats` metadata shape should stay compatible with current consumers.
5. Agent-call limit truncation behavior should remain the same unless changed intentionally later.
6. Verification-failure fallback behavior must remain:
   - if R3 fails, use merged chunk issues
7. Empty-analysis shape must remain compatible with:
   - `review_tasks.py`
   - score calculation
   - comment formatting
8. Chunk-level R1/R2 execution should continue to reuse existing review logic instead of duplicating it.

## Recommended End State
After cleanup, `ChunkedReviewer` should mostly do this:

- receive prepared inputs
- coordinate the chunked-review pipeline
- call collaborators for:
  - context resolution
  - prompt building
  - issue merging
  - verification
  - result shaping

In other words:

- `ChunkedReviewer` should stay the conductor
- helper modules should play the instruments

## Best Extraction Map

| Current Concern | Current Methods / Code Area | Recommended Destination | Why |
| --- | --- | --- | --- |
| Chunk pipeline result models | `ChunkReviewResult`, `MergedIssue`, `ChunkedReviewResult` | `backend/app/review/services/chunk_review_models.py` | Data models should be reusable and independent of execution logic |
| R0 context resolution and caching | R0 portion inside `review_large_pr()` | `backend/app/review/services/chunk_context_service.py` or shared `review_context_extractor.py` | Cache lookup and extractor orchestration are separate from chunk execution |
| Chunk prompt building | `_prepare_chunk_diff`, `_build_file_changes_text`, `_build_r1_prompt` | `backend/app/review/services/chunk_prompt_builder.py` | Prompt generation is pure helper logic and currently duplicates reviewer prompt behavior |
| Issue normalization and merge | `_extract_issues_from_analysis`, `_normalize_issue`, `_merge_chunk_results`, `_merge_non_issue_data` | `backend/app/review/services/chunk_result_merger.py` | These are merge/transformation helpers, not pipeline control |
| Verification prompt building and execution | `_run_global_verification`, `_build_verification_round2_prompt`, `_build_verification_prompt`, `_run_verification_round` | `backend/app/review/services/chunk_verification_service.py` | R3 verification is large enough to be its own sub-service |
| Verification parsing and final analysis shaping | `_parse_verification_result`, `_build_analysis_from_verified`, `_convert_merged_to_analysis_for_verification`, `_convert_to_analysis`, `_create_empty_analysis`, `_create_empty_result` | `backend/app/review/services/chunk_analysis_mapper.py` or `chunk_result_mapper.py` | These are pure conversion/mapping concerns |
| Shared reviewer helpers currently borrowed from `AIReviewer` | `_create_issues_for_verification`, `_deduplicate_issues`, fallback prompt access, session helpers | Shared collaborators under `backend/app/review/services/` | `ChunkedReviewer` should stop reaching into `AIReviewer` private methods |

## What Should Stay in `ChunkedReviewer`
These methods are the best candidates to remain in the class, at least in the first cleanup rounds:

- `__init__`
- `review_large_pr`
- `_review_chunks_parallel`
- `_review_single_chunk_with_retry`
- `_review_single_chunk`

Why these should stay:
- they are the real chunked-review orchestration core
- they describe the review flow itself
- moving them too early would make the refactor harder to follow

Even these methods should become thinner over time by delegating helper work.

## Biggest Structural Problem To Fix First
The most important cleanup target is not file length by itself.

The real issue is this:

`ChunkedReviewer` relies on many `AIReviewer` internals, including private methods.

Current examples:
- `ai_reviewer._deduplicate_issues(...)`
- `ai_reviewer._create_issues_for_verification(...)`
- `ai_reviewer._get_fallback_prompt()`
- `ai_reviewer._get_fallback_verification_agent_prompt()`
- `ai_reviewer._run_verification_session(...)`
- `ai_reviewer._run_claude_with_session(...)`

That means the file boundary is weak. The clean-code strategy should reduce this coupling first by extracting shared services, not by copying more code into `ChunkedReviewer`.

## Safest Refactor Order

### Phase 1: Extract Pure Helpers First
This is the safest starting point because these moves should not change runtime behavior.

1. Move dataclasses to `chunk_review_models.py`
2. Extract chunk prompt-building helpers:
   - `_prepare_chunk_diff`
   - `_build_file_changes_text`
   - `_build_r1_prompt`
3. Extract merge/normalization helpers:
   - `_extract_issues_from_analysis`
   - `_normalize_issue`
   - `_merge_chunk_results`
   - `_merge_non_issue_data`
4. Extract empty-analysis/result builders

Why this phase first:
- mostly pure functions / pure mappings
- easiest to test
- immediately reduces class size and noise

### Phase 2: Extract Verification as a Sub-Service
Move all R3-specific behavior into `chunk_verification_service.py`:

- `_run_global_verification`
- `_build_verification_round2_prompt`
- `_build_verification_prompt`
- `_run_verification_round`
- `_parse_verification_result`
- `_build_analysis_from_verified`
- `_convert_merged_to_analysis_for_verification`
- `_convert_to_analysis`

Why this phase second:
- R3 is already a distinct sub-pipeline
- it has its own prompts, session handling, parsing, and stats
- separating it dramatically improves readability without changing chunk R1/R2 flow

### Phase 3: Extract R0 Context Resolution
Move the R0 block from `review_large_pr()` into a focused service:

Recommended destination:
- `chunk_context_service.py`

Responsibilities:
- get cached R0 context
- run extractor if cache missing
- store context back to cache
- format global context for downstream use
- set stage back to `review`

Why this phase third:
- R0 is conceptually separate from chunk splitting and chunk execution
- it currently inflates `review_large_pr()` heavily

### Phase 4: Replace Private `AIReviewer` Calls with Shared Collaborators
This is the highest-value design cleanup.

Create or reuse shared collaborators for:
- issue deduplication
- issue-to-verification formatting
- fallback prompt access
- Claude session execution

Then update both `AIReviewer` and `ChunkedReviewer` to use those shared services.

Why this phase after extractions:
- once helpers are split, it becomes much easier to identify what should be shared
- it reduces cross-class private coupling without changing top-level behavior

## Recommended New File Layout
Inside `backend/app/review/services/`, a clean outcome could look like:

- `chunked_reviewer.py`
  - orchestrator only
- `chunk_review_models.py`
  - `ChunkReviewResult`
  - `MergedIssue`
  - `ChunkedReviewResult`
- `chunk_prompt_builder.py`
  - diff extraction
  - chunk prompt building
- `chunk_result_merger.py`
  - normalization
  - issue merging
  - non-issue merging
- `chunk_verification_service.py`
  - full R3 verification sub-pipeline
- `chunk_analysis_mapper.py`
  - analysis/result conversion helpers
- `chunk_context_service.py`
  - R0 cache/extractor/context preparation

This is modular, but still close to the existing code. It is an extraction plan, not a rewrite.

## Recommended Responsibilities After Cleanup

### `chunked_reviewer.py`
Should keep:
- top-level chunked pipeline control
- parallel execution coordination
- retry coordination

### `chunk_prompt_builder.py`
Should own:
- diff slicing for a chunk
- building file-changes markdown
- building chunk R1 prompt from template + contexts

### `chunk_result_merger.py`
Should own:
- normalize issue shapes
- merge issue lists
- merge non-issue arrays

### `chunk_verification_service.py`
Should own:
- building verification prompts
- running verification rounds
- parsing verification responses
- returning final verified analysis + stats

### `chunk_context_service.py`
Should own:
- R0 cache lookup
- extractor invocation
- cache persistence
- global context preparation

### shared review helpers
Should own logic that both `AIReviewer` and `ChunkedReviewer` need:
- issue deduplication
- verification issue formatting
- fallback prompt access
- Claude session execution

## Best Way To Preserve Old Logic

### 1. Keep the existing public method signatures first
Do not start by changing how callers use the class.

Example:
- keep `review_large_pr(file_changes, combined_diff, chunk_manager)`
- keep `ChunkedReviewResult`

### 2. Extract code into helpers, then call the helpers
Instead of rewriting logic, move the exact method bodies into new modules and call them.

Example:
- old:
  - `self._merge_non_issue_data(successful_results)`
- new:
  - `self.result_merger.merge_non_issue_data(successful_results)`

### 3. Keep intermediate data shapes stable
Do not change:
- `MergedIssue`
- analysis dict keys
- verification stats keys
- `_chunked_review_stats`

### 4. Keep failure and fallback logic unchanged in the first pass
Especially preserve:
- empty chunk handling
- rate-limit short-circuit
- all-chunks-failed behavior
- verification-failed fallback

## Best First PR For This File
If this cleanup is done incrementally, the best first PR is:

1. Add `chunk_review_models.py`
2. Add `chunk_prompt_builder.py`
3. Add `chunk_result_merger.py`
4. Update `ChunkedReviewer` to call those helpers

Why this is the best first PR:
- lowest risk
- no pipeline redesign
- removes the easiest non-orchestration code first
- makes later R3/context extraction simpler

## What Not To Do During Cleanup

1. Do not redesign chunking thresholds or agent-call policy at the same time.
2. Do not change the R0 cache/extractor policy while extracting helpers.
3. Do not change verification output schema while moving verification code.
4. Do not replace the chunked-review flow with an entirely new abstraction in one PR.
5. Do not keep adding more private `AIReviewer` calls to avoid short-term duplication.

## Most Important Design Recommendation
The best long-term cleanup for this file is:

> make `ChunkedReviewer` depend on shared review services, not on `AIReviewer` private methods

That one change fixes the biggest structural problem:
- it keeps old behavior
- it removes fragile cross-class coupling
- it lets both files become clean orchestrators instead of partial helper libraries for each other

## Final Recommendation
Clean this file in layers:

1. move pure helper logic out first
2. extract R3 verification as its own service
3. extract R0 context resolution
4. replace private `AIReviewer` dependencies with shared collaborators

If that order is followed, `chunked_reviewer.py` can become a clean orchestration file without losing the existing chunked-review behavior that already works today.
