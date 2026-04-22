# Clean Code Agent Master Context

## Purpose
This file is the master handoff document for clean-code work on these three service files:

- `backend/app/review/services/ai_reviewer.py`
- `backend/app/review/services/chunked_reviewer.py`
- `backend/app/review/services/webhook_service.py`

This document answers:

- what the current situation is
- what we want to achieve
- which new files should be created
- which responsibility should live in which file

Use this file together with the existing per-file reference and strategy docs.

## How To Use This Handoff
When the agent is asked to clean one file, give it:

### For `ai_reviewer.py`
- `AI_REVIEWER_CLEAN_CODE_REFERENCE.md`
- `AI_REVIEWER_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`

### For `chunked_reviewer.py`
- `CHUNKED_REVIEWER_CLEAN_CODE_REFERENCE.md`
- `CHUNKED_REVIEWER_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`

### For `webhook_service.py`
- `WEBHOOK_SERVICE_CLEAN_CODE_REFERENCE.md`
- `WEBHOOK_SERVICE_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`

The per-file docs explain the file in detail. This master doc explains the big picture and the exact target structure.

## Current Situation
Right now, all three files are doing valuable work, but each file mixes too many responsibilities.

### Shared pattern across all three files
Each file currently mixes:

- orchestration
- helper logic
- transformation logic
- formatting or parsing logic
- provider-specific logic
- retry or execution logic
- database or application workflow logic

So the problem is not that the old code is useless. The problem is that too much code is living in the wrong file.

## What We Want Overall
For all three targets, we want the same final direction:

- keep the old behavior
- keep the public entry points stable
- keep the main original file as the orchestrator or facade
- move helper logic into smaller focused files
- move provider-specific logic into provider service files
- reduce cross-file private-method dependency

In short:

> keep the pipeline, clean the boundaries

## Target 1: `ai_reviewer.py`

### Current role
`AIReviewer` is the main review engine facade, but today it also owns:

- prompt loading
- prompt building
- Claude CLI execution
- JSON repair and parsing
- review comment formatting
- cost calculation
- provider-specific PR or MR description updates
- extractor and global-context helpers

### What we want `ai_reviewer.py` to become
We want `AIReviewer` to stay the main review orchestrator only.

It should mainly keep:

- initialization
- CLAUDE stage switching
- `analyze_mr(...)`
- `_standard_review(...)`
- `_chunked_review(...)`
- thin coordination for multi-round review

### Files to create or update for `ai_reviewer.py`

| File | What it should contain |
| --- | --- |
| `backend/app/utils/json_repair_handler.py` | low-level JSON string cleanup and repair helpers |
| `backend/app/review/services/claude_response_parser.py` | Claude wrapper extraction, JSON parsing, validation of review output |
| `backend/app/review/services/review_prompt_builder.py` | build final review prompts using MR data, JIRA context, framework rules, and diff text |
| `backend/app/prompt/services/prompt_service.py` | extend this existing file to own prompt retrieval and fallback selection |
| `backend/app/review/services/review_comment_formatter.py` | markdown comment rendering, sanitization, badges, output presentation |
| `backend/app/review/services/review_cost_service.py` | token and cost calculation |
| `backend/app/review/services/review_analysis_merger.py` | round summary helpers, issue merge helpers, dedup helpers if kept shared |
| `backend/app/review/services/claude_cli_service.py` | subprocess execution, retry logic, session reuse, CLI env setup |
| `backend/app/review/services/review_context_extractor.py` | extractor-agent flow and global-context formatting |
| `backend/app/integrations/gitlab/gitlab_service.py` | GitLab MR description update method |
| `backend/app/integrations/github/github_service.py` | GitHub PR description update method |
| `backend/app/integrations/bitbucket/bitbucket_service.py` | Bitbucket PR description update method |

### What should stay in `ai_reviewer.py`
- high-level review routing
- review-mode selection
- orchestration calls into helper services

### What should move out of `ai_reviewer.py`
- JSON repair details
- response parsing details
- large markdown formatting
- provider HTTP description updates
- prompt persistence access
- cost logic
- raw CLI execution details

## Target 2: `chunked_reviewer.py`

### Current role
`ChunkedReviewer` is the large-PR execution engine, but today it also owns:

- R0 context resolution
- chunk prompt building
- issue normalization
- issue merge logic
- non-issue merge logic
- verification prompt building
- verification execution
- verification parsing
- result shaping

It also depends too heavily on private methods inside `AIReviewer`.

### What we want `chunked_reviewer.py` to become
We want `ChunkedReviewer` to stay the chunked-review orchestrator only.

It should mainly keep:

- constructor
- `review_large_pr(...)`
- `_review_chunks_parallel(...)`
- `_review_single_chunk_with_retry(...)`
- `_review_single_chunk(...)`

Everything else should become a focused helper or service.

### Files to create or update for `chunked_reviewer.py`

| File | What it should contain |
| --- | --- |
| `backend/app/review/services/chunk_review_models.py` | `ChunkReviewResult`, `MergedIssue`, `ChunkedReviewResult` |
| `backend/app/review/services/chunk_prompt_builder.py` | chunk diff slicing, file-change text building, R1 chunk prompt creation |
| `backend/app/review/services/chunk_result_merger.py` | issue normalization, issue merge, non-issue merge |
| `backend/app/review/services/chunk_verification_service.py` | R3 verification pipeline, verification prompts, verification execution, verification parsing |
| `backend/app/review/services/chunk_analysis_mapper.py` | convert merged issues and verification results into final analysis shapes |
| `backend/app/review/services/chunk_context_service.py` | R0 cache lookup, extractor trigger, cache storage, global context preparation |
| `backend/app/review/services/review_analysis_merger.py` | shared issue deduplication and verification-input formatting if both reviewers use it |
| `backend/app/review/services/claude_cli_service.py` | shared Claude session execution if chunk verification uses common runner logic |
| `backend/app/review/services/review_context_extractor.py` | shared extractor logic if reused between `AIReviewer` and `ChunkedReviewer` |

### What should stay in `chunked_reviewer.py`
- large-PR pipeline order
- chunk execution orchestration
- retry coordination
- bounded parallelism coordination

### What should move out of `chunked_reviewer.py`
- pure helper logic
- pure merge logic
- verification sub-pipeline details
- result conversion helpers
- R0 context handling details

### Important structural change
`ChunkedReviewer` should stop depending on `AIReviewer` private methods and should use shared collaborators instead.

## Target 3: `webhook_service.py`

### Current role
`WebhookService` is the controller-facing webhook entry point, but today it also contains:

- GitLab-specific webhook flow
- GitHub-specific webhook flow
- Bitbucket-specific webhook flow
- provider action and state mapping
- comment-trigger detection
- idempotency logic
- review creation logic
- author resolution
- merge-triggered R0 scheduling

### What we want `webhook_service.py` to become
We want `WebhookService` to stay a thin webhook orchestrator.

It should mainly keep:

- `health_check(...)`
- `process_gitlab_webhook(...)`
- `process_gitlab_comment_webhook(...)`
- `process_github_webhook(...)`
- `process_bitbucket_webhook(...)`

Those methods should become thin entry points that delegate most work.

### Files to create or update for `webhook_service.py`

| File | What it should contain |
| --- | --- |
| `backend/app/review/services/webhook_review_service.py` | create or update `Review`, repository-review workflow, comment-trigger review creation or reuse |
| `backend/app/review/services/webhook_event_service.py` | webhook idempotency lookup, event record creation, processed event bookkeeping |
| `backend/app/review/services/webhook_trigger_service.py` | shared bot mention detection and keyword trigger rules |
| `backend/app/review/services/webhook_author_resolver.py` | author email and author identity resolution |
| `backend/app/review/services/webhook_r0_service.py` | shared merge-triggered R0 extraction scheduling |
| `backend/app/integrations/gitlab/gitlab_service.py` | GitLab payload helpers, note trigger helpers, state mapping, clone URL extraction, MR context helpers |
| `backend/app/integrations/github/github_service.py` | GitHub signature validation use, PR comment helpers, owner-repo parsing, state mapping, PR context helpers |
| `backend/app/integrations/bitbucket/bitbucket_service.py` | Bitbucket event-key mapping, comment trigger parsing, workspace-repo helpers, state mapping, PR context helpers |

### What should stay in `webhook_service.py`
- route-level orchestration
- coordination across provider adapter + shared review helpers
- stable response shapes returned to controllers

### What should move out of `webhook_service.py`
- duplicated provider detail logic
- duplicated R0 logic
- duplicated trigger detection logic
- provider-specific payload interpretation
- shared idempotency bookkeeping
- author resolution logic

## Cross-File Shared Direction
These three refactors are related. The new structure should follow a consistent rule across the whole review system:

### Keep orchestrators thin
- `AIReviewer`
- `ChunkedReviewer`
- `WebhookService`

These should coordinate, not own every helper.

### Move provider-only behavior to provider services
- GitLab logic -> `backend/app/integrations/gitlab/gitlab_service.py`
- GitHub logic -> `backend/app/integrations/github/github_service.py`
- Bitbucket logic -> `backend/app/integrations/bitbucket/bitbucket_service.py`

### Move pure helpers into focused review-service files
- parsing
- formatting
- merging
- verification
- trigger detection
- event handling
- R0 context handling

## Practical Refactor Direction
The agent should not try to clean all three files at once.

Best approach:

1. pick one target file
2. read its reference doc and strategy doc
3. use this master context for naming and boundary alignment
4. extract helpers first
5. keep old behavior stable

## Master Summary
This is the target architecture:

- `ai_reviewer.py` becomes the review orchestrator
- `chunked_reviewer.py` becomes the chunked-review orchestrator
- `webhook_service.py` becomes the webhook orchestrator

Everything else that is helper-heavy, provider-heavy, parsing-heavy, or formatting-heavy should move into smaller focused files.
