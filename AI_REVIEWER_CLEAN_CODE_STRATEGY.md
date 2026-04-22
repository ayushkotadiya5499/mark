# AIReviewer Clean Code Strategy

## Goal
Refactor `backend/app/review/services/ai_reviewer.py` into a clean orchestration file that keeps the existing review behavior but pushes non-review concerns into focused helpers or existing services.

The rule for this cleanup should be:

> move existing code first, change behavior later only if needed

This project already has useful boundaries. The safest strategy is to finish those boundaries instead of inventing a brand-new architecture.

## Refactor Rules

1. Keep public behavior stable for current callers:
   - `review_tasks.py`
   - `chunked_reviewer.py`
2. Prefer extraction over rewrite.
3. Move provider logic into provider services, not into a new random helper.
4. Move low-level JSON repair into a dedicated utility file.
5. Keep `AIReviewer` as the review facade/orchestrator.
6. Do not combine this cleanup with a large namespace migration from `src.*` to `app.*`.
7. Do not redesign prompts, chunking rules, or scoring while doing the structural cleanup.

## Recommended End State
`AIReviewer` should mostly keep:

- reviewer initialization
- CLAUDE stage switching
- `analyze_mr()`
- `_standard_review()`
- `_chunked_review()`
- thin coordination for multi-round review
- thin coordination for extractor/global-context review path

Everything else can be delegated to collaborators.

## Best Extraction Map

| Current Concern | Current Methods | Destination File | Why This Is the Right Home |
| --- | --- | --- | --- |
| JSON repair primitives | `_fix_json_escaping`, `_fix_newlines_in_strings`, `_repair_json`, `_clean_json_string` | `backend/app/utils/json_repair_handler.py` | Low-level string/JSON repair is utility logic, not reviewer domain logic |
| Claude response parsing and validation | `extract_json_string`, `_extract_result_from_wrapper`, `_parse_claude_response`, `_validate_analysis`, `_is_valid_feedback_item` | `backend/app/review/services/claude_response_parser.py` | Parsing Claude CLI output is a review-support concern, but still separate from the high-level reviewer flow |
| Prompt bundle loading | `get_prompt_template`, `get_all_prompts`, fallback prompt getters | Prefer extending `backend/app/prompt/services/prompt_service.py`, or create `backend/app/review/services/review_prompt_loader.py` | Prompt retrieval belongs near prompt storage logic, not inside the reviewer class |
| Prompt assembly and context enrichment | `create_review_prompt`, `_extract_jira_context_from_prompt`, `_extract_code_changes_from_prompt`, `_extract_framework_rules_from_prompt` | `backend/app/review/services/review_prompt_builder.py` | Building the final review prompt is a separate concern from executing the review |
| Provider-specific description updates | `update_pr_description`, `_update_gitlab_mr_description`, `_update_github_pr_description`, `_update_bitbucket_pr_description` | Existing provider services:<br>`backend/app/integrations/gitlab/gitlab_service.py`<br>`backend/app/integrations/github/github_service.py`<br>`backend/app/integrations/bitbucket/bitbucket_service.py` | These methods are provider API operations and belong beside comment posting and PR/MR fetch methods |
| Review markdown rendering | `format_review_comment`, `_get_rule_badge`, `_sanitize_suggested_code`, `_strip_html_tags`, `_sanitize_analysis_text` | `backend/app/review/services/review_comment_formatter.py` | This is presentation logic, not review orchestration |
| Cost/token accounting | `calculate_cost` | `backend/app/review/services/review_cost_service.py` | Token accounting is independent and reusable |
| Round summaries and merge helpers | `_create_round1_summary`, `_create_issues_for_verification`, `_deduplicate_issues`, `_combine_analyses` | `backend/app/review/services/review_analysis_merger.py` or `review_round_helpers.py` | These are pure helpers and can be extracted safely without changing orchestration |
| Claude CLI process execution | `run_claude_cli`, `_run_claude_with_session`, `_run_verification_session` | `backend/app/review/services/claude_cli_service.py` | Subprocess execution, retries, env setup, timeout handling, and session handling are infrastructure concerns |
| R0 extractor/global-context formatting | `run_extractor_agent`, `format_global_context_for_prompt`, `_get_fallback_extractor_prompt` | `backend/app/review/services/review_context_extractor.py` | This is a specialized sub-flow of the review system and deserves its own focused module |

## What Should Stay in `AIReviewer`
These methods are the best candidates to remain, at least after the first cleanup passes:

- `__init__`
- `set_claude_md_for_stage`
- deprecated stage wrappers for backward compatibility
- `analyze_mr`
- `_standard_review`
- `_chunked_review`

These methods can remain temporarily, but ideally become thin wrappers:

- `run_three_round_review`
- `run_two_round_review_async`
- `run_extractor_agent`
- `update_pr_description`

That gives us a clean facade: the class still feels like "the review engine", but it no longer owns every low-level detail itself.

## Safest Refactor Order

### Phase 1: Move the Clearly Wrong Concerns First
This phase is the lowest-risk cleanup because it mostly moves code to more obvious homes.

1. Extract JSON repair helpers into `backend/app/utils/json_repair_handler.py`
2. Extract review comment formatting into `backend/app/review/services/review_comment_formatter.py`
3. Move provider description-update methods into:
   - `GitLabService`
   - `GitHubService`
   - `BitbucketService`
4. Extract cost calculation into `backend/app/review/services/review_cost_service.py`

Why Phase 1 first:
- almost no review-flow behavior changes
- clear single-responsibility win
- small caller surface
- easy to test in isolation

### Phase 2: Split Parsing From Execution
After the obvious extractions, separate output parsing from CLI execution.

1. Create `claude_response_parser.py`
2. Move:
   - wrapper extraction
   - JSON parse strategies
   - validation helpers
3. Make `run_claude_cli()` depend on that parser instead of owning the parsing logic itself

Why Phase 2 second:
- keeps retry and subprocess logic intact
- reduces the biggest method complexity without touching business flow yet

### Phase 3: Split Prompt Loading and Prompt Building
1. Centralize prompt retrieval:
   - extend `PromptService`, or
   - add `review_prompt_loader.py`
2. Move prompt assembly to `review_prompt_builder.py`
3. Keep `AIReviewer.create_review_prompt()` only as a temporary wrapper or remove it after callers are updated

Why Phase 3 third:
- prompt code has dependencies on DB, JIRA, framework rules, and fallback constants
- it is separable, but it touches more collaborators than Phase 1

### Phase 4: Extract Claude CLI Runner
1. Move `run_claude_cli()`, `_run_claude_with_session()`, `_run_verification_session()` into `claude_cli_service.py`
2. Inject or compose that service into `AIReviewer`
3. Keep `AIReviewer` focused on stage order, not subprocess management

Why Phase 4 later:
- this is one of the highest-risk pieces
- many retry/session edge cases live here
- best handled after parser extraction has already reduced complexity

### Phase 5: Optional Final Split for Round Helpers
Move merge and round helper methods if the file is still too large:

- `_create_round1_summary`
- `_create_issues_for_verification`
- `_deduplicate_issues`
- `_combine_analyses`

This can go into `review_analysis_merger.py` or `review_round_helpers.py`.

Why this is optional:
- these methods are still close to review behavior
- they are less urgent than provider, JSON, and formatting extractions

## Provider-Specific Strategy
This is one of the cleanest wins in the whole refactor.

### Bitbucket
Move `_update_bitbucket_pr_description()` into:
- `backend/app/integrations/bitbucket/bitbucket_service.py`

Recommended method name:
- `update_pull_request_description()`

Why:
- the file already has `get_pull_request()`
- the file already has `post_pull_request_comment()`
- the file already owns Bitbucket auth headers and request conventions

### GitHub
Move `_update_github_pr_description()` into:
- `backend/app/integrations/github/github_service.py`

Recommended method name:
- `update_pull_request_description()`

Why:
- same service already owns GitHub PR fetch and comment posting

### GitLab
Move `_update_gitlab_mr_description()` into:
- `backend/app/integrations/gitlab/gitlab_service.py`

Recommended method name:
- `update_merge_request_description()`

Why:
- same service already owns GitLab MR fetch and comment posting

### After the move
Then either:

- delete `AIReviewer.update_pr_description()` and let callers use provider services directly, or
- keep `AIReviewer.update_pr_description()` as a thin dispatcher only

For a low-risk first pass, the thin dispatcher approach is safer.

## JSON-Specific Strategy
Since your example explicitly mentions JSON repair, this is the cleanest concrete extraction.

### Recommended split

`backend/app/utils/json_repair_handler.py`
- `_fix_json_escaping`
- `_fix_newlines_in_strings`
- `_repair_json`
- `_clean_json_string`

`backend/app/review/services/claude_response_parser.py`
- `extract_json_string`
- `_extract_result_from_wrapper`
- `_parse_claude_response`
- `_validate_analysis`
- `_is_valid_feedback_item`

### Why split it this way
- the utility file should contain repair mechanics only
- the parser file should contain Claude-specific wrapper parsing and review-schema validation
- this keeps the JSON utility generic and the parser domain-aware

### Why not keep everything in one new file
Because "JSON repair" and "Claude response parsing" are related, but not identical:
- repair code is low-level string cleanup
- parser code knows the AI output schema and review validation rules

## Prompt Strategy
Prompt-related code is the second biggest mixed responsibility after CLI/parsing.

### Suggested split

`review_prompt_loader.py` or an extended `PromptService`
- fetch active prompts
- apply fallback prompt selection

`review_prompt_builder.py`
- assemble MR metadata
- format changed-file diffs
- enrich with JIRA context
- enrich with framework rules
- apply feature-MR diagram toggle

### Why this matters
Right now `AIReviewer` knows too much about:
- prompt persistence
- prompt fallback policy
- prompt formatting
- external context enrichment

Those are different reasons to change.

## Comment-Formatting Strategy
`format_review_comment()` is large enough to deserve its own module.

### Move to `review_comment_formatter.py`
Move:
- `format_review_comment`
- `_get_rule_badge`
- `_sanitize_suggested_code`
- `_strip_html_tags`
- `_sanitize_analysis_text`

### Why
- it is pure presentation logic
- it can be unit tested without cloning repos or running Claude
- it reduces the "god class" feeling immediately

## Suggested Final Shape
After the cleanup, `AIReviewer` should read more like this:

```python
class AIReviewer:
    def __init__(self, project_dir, session_maker=None, changed_file_paths=None):
        ...
        self.prompt_builder = ReviewPromptBuilder(...)
        self.cli_service = ClaudeCliService(...)
        self.response_parser = ClaudeResponseParser(...)
        self.comment_formatter = ReviewCommentFormatter()
        self.cost_service = ReviewCostService()

    async def analyze_mr(...):
        ...

    async def _standard_review(...):
        ...

    async def _chunked_review(...):
        ...
```

The exact wiring can vary, but the idea is the same:
- `AIReviewer` coordinates
- helpers implement

## What Not To Do During This Refactor

1. Do not rewrite review prompts at the same time.
2. Do not change provider request behavior while moving methods.
3. Do not merge `app/*` and `src/*` namespaces in the same PR.
4. Do not change chunking thresholds or R0 behavior while cleaning structure.
5. Do not rename everything just because it is being moved.

If the mission is "use old code, make it clean", then preserving behavior is more important than making every name perfect in the first pass.

## Best First PR
If this cleanup is done incrementally, the best first PR is:

1. Add `backend/app/utils/json_repair_handler.py`
2. Add `backend/app/review/services/review_comment_formatter.py`
3. Move description-update methods into provider services
4. Leave `AIReviewer` calling those extracted helpers

Why this is the best first PR:
- immediate size reduction
- clear clean-code win
- very low conceptual risk
- directly matches the exact cleanup examples you called out

## Final Recommendation
Do this cleanup as a sequence of extractions, not a rewrite. The strongest first moves are:

1. JSON repair out of `AIReviewer`
2. Provider description updates into provider services
3. Comment formatting out of `AIReviewer`
4. Prompt loading/building split
5. Claude CLI runner split

That path keeps the old code, improves boundaries, and steadily turns `ai_reviewer.py` into the clean orchestration file it should be.
