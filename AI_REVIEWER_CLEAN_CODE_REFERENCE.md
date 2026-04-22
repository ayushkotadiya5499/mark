# AIReviewer Clean Code Reference

## Purpose
This document is the working reference for cleaning `backend/app/review/services/ai_reviewer.py` without changing review behavior. The goal is to preserve the good review logic, identify the mixed responsibilities inside the file, and show how that file fits into the whole project.

## Repository Snapshot
- Root layout:
  - `backend/`: FastAPI + Celery + SQLAlchemy backend
  - `frontend/`: React + Vite + TypeScript dashboard
  - `review-repos/`: local clone workspace for reviewed repositories
- Backend stack:
  - FastAPI, SQLAlchemy async, Celery, Redis, PostgreSQL
  - Loguru for logging
  - `requests` for provider APIs
  - `json-repair` for malformed JSON recovery
  - Claude Code CLI for AI review execution
- Frontend stack:
  - React 18, Vite, TypeScript, Tailwind
  - React Query, Recharts, Firebase
- Backend structure follows domain folders such as:
  - `app/review`
  - `app/integrations`
  - `app/prompt`
  - `app/repositories`
  - `app/credentials`
  - `app/analytics`
  - `app/users`, `app/teams`, `app/project`
- Important architectural note:
  - The codebase is hybrid. Domain code mostly lives under `app/*`, but shared database/helpers/constants still live under `backend/src/*`.
  - `ai_reviewer.py` is one of the biggest examples of this mixed boundary because it imports both `app.*` and `src.*`.

## Main Review Pipeline
Current end-to-end flow:

1. Webhook/controller creates a review record.
2. `backend/app/review/services/review_tasks.py` loads review + repository + credential data.
3. Provider service is selected:
   - `GitLabService`
   - `GitHubService`
   - `BitbucketService`
4. Provider service fetches PR/MR metadata and changed files, then clones the repo.
5. `AIReviewer` is created with the clone directory.
6. `AIReviewer.analyze_mr()` decides:
   - standard review
   - chunked review
7. AI analysis result is stored in DB, scored, costed, and formatted.
8. Provider service posts the review comment.
9. `AIReviewer.update_pr_description()` also updates the PR/MR description using provider-specific API calls.

That means `AIReviewer` sits in the middle of the pipeline, but today it also contains logic that already belongs to neighboring services.

## Related Files Around `AIReviewer`
These files are especially relevant to the cleanup:

| File | Current Role | Why It Matters |
| --- | --- | --- |
| `backend/app/review/services/review_tasks.py` | Orchestrates the review lifecycle | Shows that `AIReviewer` should stay focused on review logic, not whole-job orchestration |
| `backend/app/review/services/chunk_manager.py` | Splits large PRs into chunks | Already a good extraction of size/chunking logic |
| `backend/app/review/services/chunked_reviewer.py` | Runs large-PR hybrid R0/R1/R2/R3 pipeline | Already owns a large part of chunked-review behavior |
| `backend/app/review/services/framework_rules_service.py` | Detects frameworks and fetches rules | `AIReviewer` already depends on this instead of implementing framework logic itself |
| `backend/app/prompt/services/prompt_service.py` | CRUD for prompts | Prompt lookup is still duplicated in `AIReviewer` instead of being centralized here or in a prompt bundle loader |
| `backend/app/review/services/review_logger.py` | Stage logging helpers | Useful destination for some review-stage logging standardization later |
| `backend/app/integrations/gitlab/gitlab_service.py` | GitLab API + clone operations | Natural home for GitLab description-update logic |
| `backend/app/integrations/github/github_service.py` | GitHub API + clone operations | Natural home for GitHub description-update logic |
| `backend/app/integrations/bitbucket/bitbucket_service.py` | Bitbucket API + clone operations | Natural home for Bitbucket description-update logic |
| `backend/src/core/common_helpers.py` | API key rotation, balance handling, review context | Shows that some logic has already been extracted successfully from `AIReviewer` |
| `backend/src/constants/json_config.py` | JSON force instructions and known JSON-start patterns | Confirms JSON parsing is already a standalone concern |

## What `AIReviewer` Looks Like Today
Facts:

- File: `backend/app/review/services/ai_reviewer.py`
- Size: 4238 lines
- Shape: 1 class, `AIReviewer`
- Method count: roughly 40+ methods
- Main imported dependencies:
  - configuration: `app.core.config`
  - exceptions: `app.core.exceptions`
  - DB prompt model: `app.prompt.models.prompt`
  - integration services: JIRA + GitLab
  - review helpers: `FrameworkRulesService`, `ChunkManager`, `ChunkedReviewer`
  - legacy/shared infra: `src.core.common_helpers`, `src.core.database`, `src.constants.*`

This file is not just "big". It is currently a mix of:

- review orchestration
- prompt loading
- prompt construction
- Claude CLI execution
- retry logic
- API key rotation coordination
- JSON parsing and JSON repair
- validation
- comment formatting
- PR/MR description update
- token/cost calculation
- extractor/global-context formatting

## Responsibility Map Inside `AIReviewer`

| Section | Approx. Lines | Main Methods | Responsibility |
| --- | --- | --- | --- |
| Initialization and CLAUDE.md stage control | 50-166 | `__init__`, `set_claude_md_for_stage`, deprecated swap/restore methods | Local reviewer state + stage template switching |
| Review entry and routing | 168-443 | `analyze_mr`, `_standard_review`, `_chunked_review` | Chooses standard vs chunked review and wires shared context |
| Extraction helpers | 446-609 | `_extract_jira_context_from_prompt`, `extract_json_string`, `_extract_result_from_wrapper` | Pulls structured fragments out of prompt/output text |
| Prompt loading and prompt creation | 612-900 | `get_prompt_template`, `get_all_prompts`, fallback prompt getters, `create_review_prompt` | Reads prompt templates, loads JIRA/framework context, builds Round 1 prompt |
| Review-round orchestration | 902-1488 | `run_three_round_review`, `_create_issues_for_verification`, `_extract_code_changes_from_prompt`, `_extract_framework_rules_from_prompt`, `_create_round1_summary`, `_deduplicate_issues`, `_combine_analyses` | Runs R1/R2/R3 and merges results |
| Claude CLI execution engine | 1491-2194 | `run_claude_cli` | Subprocess execution, retries, session handling, auth env, timeout handling, wrapper parsing, API-key failure handling |
| JSON parse/repair/validation | 2196-2618 | `_parse_claude_response`, `_fix_json_escaping`, `_fix_newlines_in_strings`, `_repair_json`, `_clean_json_string`, `_validate_analysis`, `_is_valid_feedback_item` | Tries to salvage malformed AI output |
| Sanitization and presentation helpers | 2661-2841 | `_get_rule_badge`, `_sanitize_suggested_code`, `_strip_html_tags`, `_sanitize_analysis_text` | Cleans output for markdown rendering |
| Provider-specific description updates | 2845-3145 | `update_pr_description`, `_update_gitlab_mr_description`, `_update_github_pr_description`, `_update_bitbucket_pr_description` | Calls provider APIs to update PR/MR description |
| Review comment rendering | 3150-3647 | `format_review_comment` | Builds the full markdown comment posted back to provider |
| Cost and async wrappers | 3649-3868 | `calculate_cost`, `_run_claude_with_session`, `_run_verification_session` | Token accounting + async wrappers for CLI execution |
| R0 extractor and global context formatting | 3874-4304 | `run_extractor_agent`, `_get_fallback_extractor_prompt`, `format_global_context_for_prompt`, `run_two_round_review_async` | Extracts project-wide context and supports chunked review rounds |

## What Is Already Good and Worth Preserving
Even though the file is overloaded, several design ideas are solid and should be preserved:

1. `analyze_mr()` is a good single entry point for the review engine.
2. The split between standard review and chunked review is sensible.
3. Explicit `set_claude_md_for_stage()` is cleaner than the old swap/restore model.
4. `ChunkManager` and `ChunkedReviewer` are already strong examples of logic that was moved out of this file.
5. JIRA and framework enrichment already use dedicated services instead of reimplementing that logic inline.
6. API key rotation and insufficient balance handling have already been extracted to `src/core/common_helpers.py`.
7. The R0 extractor/global context idea is a meaningful architectural feature, not accidental complexity.

The cleanup mission should protect these ideas and mainly remove the unrelated concerns that accumulated around them.

## Where the File Breaks Clean-Code Boundaries

### 1. Review domain logic is mixed with transport/client logic
`AIReviewer` both decides how to review code and also knows:
- how to run a subprocess
- how to parse Claude CLI wrapper output
- how to talk to GitLab/GitHub/Bitbucket APIs
- how to build markdown comments

This makes the class hard to reason about and hard to test.

### 2. Provider-specific code sits in the wrong layer
`_update_gitlab_mr_description`, `_update_github_pr_description`, and `_update_bitbucket_pr_description` belong conceptually beside:
- `post_merge_request_comment`
- `post_pull_request_comment`
- `get_pull_request`
- `get_merge_request`

Those methods are already inside provider integration services, so keeping description updates inside `AIReviewer` creates a broken boundary.

### 3. JSON repair and response parsing are embedded inside the reviewer
The JSON handling code is large enough to be its own concern:
- wrapper extraction
- malformed escape fixing
- newline fixing
- repair fallback
- structural validation

There is even evidence that a dedicated helper may have existed before:
- `backend/app/utils/__pycache__/json_repair_handler.cpython-310.pyc`

But there is no current source file under `backend/app/utils/`.

### 4. Prompt access is mixed with prompt composition
`get_all_prompts()` directly queries the DB and applies fallback logic.
`create_review_prompt()` then enriches the prompt with:
- JIRA context
- framework rules
- diff formatting
- feature-MR diagram toggling

Those are related, but not identical responsibilities.

### 5. Presentation formatting is mixed with analysis logic
`format_review_comment()` is a large rendering function that also depends on:
- text sanitization
- code block sanitization
- metadata handling
- framework badge rendering

This is closer to a formatter/presenter than to a review engine.

### 6. The class mixes synchronous, asynchronous, and subprocess concerns
The same class contains:
- async DB prompt fetching
- sync subprocess execution
- async wrappers over sync subprocess execution
- chunked parallel review coordination

That makes lifecycle and testing complexity higher than necessary.

## Existing Natural Boundaries the Project Already Gives Us
The codebase already suggests where logic should live:

- Provider HTTP update logic -> `app/integrations/*/*_service.py`
- Prompt storage/access -> `app/prompt/services`
- Shared low-level utilities -> `app/utils` or `src/constants`
- Review orchestration -> `app/review/services`
- Chunking -> `chunk_manager.py`
- Large-review pipeline -> `chunked_reviewer.py`
- Shared API key/balance policy -> `src/core/common_helpers.py`

So the cleanup plan does not need a redesign. It mostly needs to finish boundaries that the project already started.

## What `AIReviewer` Should Eventually Be
After cleanup, `AIReviewer` should ideally act as a facade/orchestrator for review behavior:

- accept review inputs
- decide standard vs chunked path
- coordinate prompt builder, Claude runner, parser, and formatter
- preserve the existing public contract used by `review_tasks.py` and `chunked_reviewer.py`

It should not be the permanent home for:
- provider-specific REST updates
- JSON repair primitives
- large markdown rendering functions
- raw prompt bundle lookup
- standalone token cost calculation

## Most Important Takeaway
`ai_reviewer.py` already contains good review logic, but it is carrying too many adjacent concerns that the rest of the project has already given proper homes for. The clean-code mission should keep the class as the review orchestrator and steadily move the non-review concerns into focused helpers and existing provider services.
