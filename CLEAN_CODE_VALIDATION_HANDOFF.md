# Clean Code Validation Handoff

## Purpose
This document is the validation handoff for the clean-code work around:

- `backend/app/review/services/ai_reviewer.py`
- `backend/app/review/services/chunked_reviewer.py`
- `backend/app/review/services/webhook_service.py`

Use this after refactoring to verify that:

- imports still resolve
- names are still consistent across files
- moved code is still reachable from old call sites
- no `has no attribute` or wrong-import failures were introduced
- current behavior still matches old behavior as closely as possible

This is not a refactor strategy document.
This is the "prove the cleaned code still works" document.

## Read This Together With
Give the agent this file together with the existing clean-code docs:

- `CLEAN_CODE_DOCS_INDEX.md`
- `CLEAN_CODE_AGENT_GUIDELINES.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
- the target file's `*_CLEAN_CODE_REFERENCE.md`
- the target file's `*_CLEAN_CODE_STRATEGY.md`

## Main Goal
The goal is strict:

> the cleaned code must behave like the old code unless a change was explicitly approved

That means validation is not only:

- "does Python compile?"

It is also:

- "does every caller still call the same thing?"
- "did a moved method keep the same contract?"
- "did output shape stay compatible?"
- "did pipeline order stay the same?"

## Validation Scope
Validate all of the following after any clean-code extraction:

1. syntax validity
2. import validity
3. package export validity
4. caller-to-callee name consistency
5. method signature compatibility
6. return shape compatibility
7. orchestration flow compatibility
8. provider-specific behavior compatibility
9. chunked-review behavior compatibility
10. webhook behavior compatibility

## Current Real Callers And Contracts

### `AIReviewer`
Current real callers:

- `backend/app/review/services/review_tasks.py`
- `backend/app/r0_constructore/services/r0_context_scheduler.py`
- `backend/app/review/services/chunked_reviewer.py`

Current methods that callers depend on directly:

- `AIReviewer(...)`
- `analyze_mr(...)`
- `update_pr_description(...)`
- `calculate_cost(...)`
- `format_review_comment(...)`
- `run_two_round_review_async(...)`
- `run_extractor_agent(...)`
- `set_claude_md_for_stage(...)`
- `format_global_context_for_prompt(...)`

Current private methods that `ChunkedReviewer` still depends on:

- `_deduplicate_issues(...)`
- `_get_fallback_prompt()`
- `_create_issues_for_verification(...)`
- `_get_fallback_verification_agent_prompt()`
- `_run_verification_session(...)`
- `_run_claude_with_session(...)`

Important rule:
If these private methods are extracted or renamed, either:

- update every call site in the same change, or
- keep thin wrapper methods on `AIReviewer` until all callers are updated

### `ChunkedReviewer`
Current real caller:

- `backend/app/review/services/ai_reviewer.py`

Current public dependency points:

- `ChunkedReviewer(...)`
- `review_large_pr(...)`
- `ChunkedReviewResult`
- `ChunkReviewResult`

Important package export:

- `backend/app/review/services/__init__.py` currently exports `ChunkedReviewer` and `ChunkReviewResult`

If dataclasses move out of `chunked_reviewer.py`, update package exports too.

### `WebhookService`
Current real caller:

- `backend/app/review/controllers/webhooks.py`

Current controller-facing entry points:

- `health_check(...)`
- `process_gitlab_webhook(...)`
- `process_gitlab_comment_webhook(...)`
- `process_github_webhook(...)`
- `process_bitbucket_webhook(...)`

If orchestration is split into smaller helpers, these entry points must remain valid for the controller.

## Current Provider Service API Surface
The cleaned code must stay aligned with the provider service methods that already exist.

### GitLab service currently exposes

- `get_merge_request(...)`
- `get_merge_request_changes(...)`
- `post_merge_request_comment(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`
- `clone_repository(...)`
- `get_repository_info(...)`
- `validate_webhook_token(...)`
- `parse_webhook_payload(...)`

### GitHub service currently exposes

- `get_pull_request(...)`
- `get_pull_request_changes(...)`
- `post_pull_request_comment(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`
- `clone_repository(...)`
- `get_repository_info(...)`
- `parse_webhook_payload(...)`
- `validate_webhook_signature(...)`

### Bitbucket service currently exposes

- `get_pull_request(...)`
- `get_pull_request_changes(...)`
- `post_pull_request_comment(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`
- `clone_repository(...)`
- `get_repository_info(...)`
- `parse_webhook_payload(...)`
- `validate_webhook_signature(...)`

If code is moved into provider services, use names that fit these services and update all call sites together.

## Non-Negotiable Compatibility Checks

### 1. Imports must still work
After refactor, validate:

- moved helpers are imported from the correct module
- old modules do not keep stale imports to deleted code
- circular imports were not introduced
- package-level exports still work

### 2. Names must stay consistent across files
Common failure pattern:

- one file uses old name
- another file uses new name
- runtime fails with `AttributeError` or import error

Examples to watch carefully:

- moved helper classes
- moved dataclasses
- provider update methods
- fallback prompt helpers
- parser/formatter/helper service names

### 3. Signatures must stay compatible
Do not silently change:

- parameter names used by keyword calls
- parameter order for positional callers
- tuple return length
- dict keys expected by downstream logic

High-risk examples in this repo:

- `AIReviewer.analyze_mr(...)` returns 8 values
- `AIReviewer.run_two_round_review_async(...)` returns 4 values
- `AIReviewer.run_extractor_agent(...)` returns 3 values
- `ChunkedReviewer.review_large_pr(...)` returns `ChunkedReviewResult`

### 4. Output shape must stay compatible
Review analysis objects must still contain the expected keys used by:

- `review_tasks.py`
- `score_calculator.py`
- `format_review_comment(...)`
- chunked review merge and verification flow

Important keys to preserve:

- `critical_issues`
- `security_concerns`
- `bugs`
- `performance_notes`
- `code_quality`
- `integration_issues`
- `testing_gaps`
- `summary`
- `overall_score`
- `positive_aspects`
- `recommendations`
- `description`
- `flow_diagram`

If chunked review is used, preserve:

- `_chunked_review_stats`

## Required Validation Workflow

### Phase 1: Build the old-vs-new comparison map
Before testing, create a small table:

| Old location | New location | Old public name kept? | Call sites updated? |
| --- | --- | --- | --- |

Do this for every extracted method/class/dataclass.

This is mandatory because most post-refactor breakage in this repo will come from partial extraction.

### Phase 2: Run syntax checks
Run at minimum:

```powershell
@'
import py_compile
files = [
    r"backend\app\review\services\ai_reviewer.py",
    r"backend\app\review\services\chunked_reviewer.py",
    r"backend\app\review\services\webhook_service.py",
]
for path in files:
    py_compile.compile(path, doraise=True)
    print(f"OK {path}")
'@ | python -
```

If helpers were extracted, extend the file list to include the new helper files too.

### Phase 3: Run import smoke checks
Run package and module import checks:

```powershell
@'
import sys
from pathlib import Path
sys.path.insert(0, str(Path("backend").resolve()))

checks = [
    ("app.review.services.ai_reviewer", "AIReviewer"),
    ("app.review.services.chunked_reviewer", "ChunkedReviewer"),
    ("app.review.services.webhook_service", "WebhookService"),
    ("app.review.services", "AIReviewer"),
    ("app.review.services", "ChunkedReviewer"),
    ("app.review.services", "ChunkReviewResult"),
    ("app.review.services", "WebhookService"),
]

for module_name, attr in checks:
    module = __import__(module_name, fromlist=[attr])
    value = getattr(module, attr)
    print(f"OK {module_name}.{attr} -> {value}")
'@ | python -
```

If `ChunkReviewResult`, `ChunkedReviewResult`, or other exports move to new files, add import checks for the new module and the old package path.

### Phase 4: Search for stale names and stale imports
Run repo-wide searches for old and new names:

```powershell
rg -n "AIReviewer\(|ChunkedReviewer\(|WebhookService\(" backend
rg -n "update_pr_description\(|format_review_comment\(|calculate_cost\(" backend
rg -n "run_two_round_review_async\(|run_extractor_agent\(" backend
rg -n "process_gitlab_webhook\(|process_github_webhook\(|process_bitbucket_webhook\(" backend
rg -n "from app\.review\.services|from app\.review\.services\.chunked_reviewer import|from app\.review\.services\.ai_reviewer import|from app\.review\.services\.webhook_service import" backend
```

If you renamed a symbol, verify that:

- no old import remains
- no half-migrated call remains
- no package export still points to the wrong module

### Phase 5: Verify method signatures and return contracts
For each touched public or cross-file method, compare:

- old signature
- new signature
- caller usage

Check especially:

- keyword argument names
- tuple unpacking count
- dataclass field names
- return object type

Minimum methods to verify:

- `AIReviewer.analyze_mr`
- `AIReviewer.update_pr_description`
- `AIReviewer.calculate_cost`
- `AIReviewer.format_review_comment`
- `AIReviewer.run_two_round_review_async`
- `AIReviewer.run_extractor_agent`
- `ChunkedReviewer.review_large_pr`
- `WebhookService.process_gitlab_webhook`
- `WebhookService.process_gitlab_comment_webhook`
- `WebhookService.process_github_webhook`
- `WebhookService.process_bitbucket_webhook`

### Phase 6: Verify behavior parity by flow

#### AI reviewer flow
Confirm all of this still matches old behavior:

- `analyze_mr()` still decides standard vs chunked review the same way
- standard review still builds prompt and runs review the same way
- chunked review still prepares shared context and builds `ChunkedReviewer`
- `review_tasks.py` can still call:
  - `update_pr_description(...)`
  - `calculate_cost(...)`
  - `format_review_comment(...)`
- API key rotation and insufficient-balance handling still behave the same
- prompt fallback behavior still exists
- comment formatting output shape stays compatible

#### Chunked reviewer flow
Confirm pipeline order is unchanged:

1. R0 context
2. chunk creation
3. parallel chunk R1/R2
4. merge
5. global R3 verification
6. final analysis

Also confirm:

- same fallback behavior when all chunks fail
- same fallback behavior when verification fails
- same analysis dict category names
- same `_chunked_review_stats` shape
- if helpers moved out, no remaining broken private-method dependency exists

#### Webhook flow
Confirm for GitLab, GitHub, and Bitbucket:

- controller still calls the same `WebhookService` entry method
- request validation/parsing still happens
- repository and credential lookup still happens
- review creation/update flow still happens
- idempotency flow still happens where it existed before
- provider comments are still posted through provider services
- background task dispatch still happens through `process_code_review.delay(...)`
- merge/close state updates still happen

Important parity note:
If you intentionally fix a known bug while refactoring, record it clearly as an approved behavior change.
Do not hide it inside "cleanup".

### Phase 7: Verify package export compatibility
Check:

- `backend/app/review/services/__init__.py`

If any moved symbol used to be exported from the package, either:

- keep exporting it from the package, or
- update every package-level importer in the same change

### Phase 8: Verify no dead wrappers remain broken
If you keep temporary wrappers for backward compatibility, verify that they:

- import the new helper correctly
- forward parameters correctly
- return the same type/shape as before

Thin wrapper validation is especially important for:

- extracted AIReviewer helpers
- extracted chunked review models/helpers
- moved provider update methods

## Repo-Specific Risk Hotspots

### 1. `ChunkedReviewer` still depends on `AIReviewer` private methods
Current dependency points in `chunked_reviewer.py`:

- `_deduplicate_issues`
- `_get_fallback_prompt`
- `_create_issues_for_verification`
- `_get_fallback_verification_agent_prompt`
- `_run_verification_session`
- `_run_claude_with_session`

This is the highest-risk area for `AttributeError` after extraction.

### 2. `review_tasks.py` depends on AIReviewer utility methods after analysis
Current post-analysis calls:

- `update_pr_description(...)`
- `calculate_cost(...)`
- `format_review_comment(...)`

If these move out of `AIReviewer`, either:

- keep wrapper methods on `AIReviewer`, or
- update `review_tasks.py` in the same PR

### 3. `__init__.py` exports can become stale
If models/helpers move out of `chunked_reviewer.py`, package exports can still point to the old file and fail at import time.

### 4. Controller entry points must remain stable
`webhooks.py` directly instantiates `WebhookService()` and calls:

- `process_gitlab_webhook`
- `process_gitlab_comment_webhook`
- `process_github_webhook`
- `process_bitbucket_webhook`

These names must remain stable unless controller code is updated at the same time.

### 5. Provider-service method naming drift
If PR/MR description update logic is moved into provider services, make sure the final method names and the calling code agree.

## Lightweight Commands To Run Every Time

### Syntax
```powershell
@'
import py_compile
for path in [
    r"backend\app\review\services\ai_reviewer.py",
    r"backend\app\review\services\chunked_reviewer.py",
    r"backend\app\review\services\webhook_service.py",
]:
    py_compile.compile(path, doraise=True)
    print(path)
'@ | python -
```

### Import smoke
```powershell
@'
import sys
from pathlib import Path
sys.path.insert(0, str(Path("backend").resolve()))
from app.review.services.ai_reviewer import AIReviewer
from app.review.services.chunked_reviewer import ChunkedReviewer
from app.review.services.webhook_service import WebhookService
print(AIReviewer, ChunkedReviewer, WebhookService)
'@ | python -
```

### Stale-name search
```powershell
rg -n "ChunkReviewResult|ChunkedReviewResult|MergedIssue|AIReviewer|ChunkedReviewer|WebhookService" backend
```

## Baseline Already Confirmed On 2026-04-24
The following lightweight checks passed in the current workspace before new refactor validation:

- `python --version` -> `Python 3.12.2`
- `py_compile` succeeded for:
  - `backend/app/review/services/ai_reviewer.py`
  - `backend/app/review/services/chunked_reviewer.py`
  - `backend/app/review/services/webhook_service.py`
- import smoke succeeded for:
  - `app.review.services.ai_reviewer.AIReviewer`
  - `app.review.services.chunked_reviewer.ChunkedReviewer`
  - `app.review.services.webhook_service.WebhookService`
  - package exports from `app.review.services`

Use this as the baseline. After cleanup, these checks must still pass.

## Expected Final Report From The Agent
Ask the agent to return a report in this format:

### 1. Files touched
List every changed file.

### 2. Extraction map
For each moved method/class:

| Symbol | Old location | New location | Compatibility preserved? |
| --- | --- | --- | --- |

### 3. Validation results
For each validation step:

| Check | Command or method | Result | Notes |
| --- | --- | --- | --- |

### 4. Behavior parity notes
List:

- preserved behaviors confirmed
- intentional behavior changes, if any
- unresolved risks

### 5. Final decision
One of:

- `safe to continue`
- `safe with noted risks`
- `not safe yet`

## Final Rule
If there is a choice between:

- cleaner structure but uncertain compatibility

and

- slightly less elegant structure but proven compatibility

choose proven compatibility.
