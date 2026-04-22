# WebhookService Clean Code Strategy

## Goal
Clean `backend/app/review/services/webhook_service.py` without breaking current webhook behavior.

The right rule is:

> move provider-specific webhook logic to provider services, keep shared DB and review workflow in review services

So the answer is:

- yes, some code should move to `gitlab_service.py`
- yes, some code should move to `github_service.py`
- yes, some code should move to `bitbucket_service.py`
- but no, not all code from `webhook_service.py` should move into provider files

## Main Boundary

### Code that should move to provider service files
Provider service files should own logic that depends on provider API shape or provider webhook rules:

- webhook signature or token validation
- provider payload parsing
- provider event and action mapping
- provider state mapping
- provider comment-trigger extraction
- provider-specific PR or MR context fetching
- provider-specific comment posting
- provider-specific clone URL normalization

### Code that should stay out of provider service files
These are application responsibilities and should stay in webhook or review-domain services:

- SQLAlchemy queries for `Repository`, `Review`, `Credential`, `WebhookEvent`
- repository auto-create policy
- token decrypt policy
- idempotency storage
- review creation and update
- Celery dispatch through `process_code_review.delay(...)`
- R0 scheduling through repository lookup plus task enqueue

If DB and Celery code are moved into provider files, those files stop being clean integration adapters.

## Recommended Destination Map

| Current concern | Best destination | Why |
| --- | --- | --- |
| GitLab-only webhook rules | `backend/app/integrations/gitlab/gitlab_service.py` | Depends on GitLab payload shape and GitLab API behavior |
| GitHub-only webhook rules | `backend/app/integrations/github/github_service.py` | Depends on GitHub payload shape and GitHub API behavior |
| Bitbucket-only webhook rules | `backend/app/integrations/bitbucket/bitbucket_service.py` | Depends on Bitbucket event keys and API behavior |
| Shared review creation and update | new `backend/app/review/services/webhook_review_service.py` | Shared application workflow, not provider-specific |
| Shared idempotency handling | new `backend/app/review/services/webhook_event_service.py` | Same logic pattern across providers |
| Shared trigger keyword logic | new `backend/app/review/services/webhook_trigger_service.py` | Bot mention and keyword rules should be reused |
| Shared author resolution | new `backend/app/review/services/webhook_author_resolver.py` | DB-backed identity resolution is not provider adapter logic |
| Shared R0 merge scheduling | new `backend/app/review/services/webhook_r0_service.py` | Same scheduling logic for all providers |

## Exactly What To Put In Each Provider File

### 1. `backend/app/integrations/gitlab/gitlab_service.py`
This file already has:
- `validate_webhook_token(...)`
- `parse_webhook_payload(...)`
- `get_merge_request(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`

Add more GitLab-only helpers here:

| Move or add | Why it belongs in `gitlab_service.py` |
| --- | --- |
| `map_merge_request_state(...)` helper | GitLab state strings are provider-specific |
| `extract_note_trigger_context(...)` helper | GitLab note payload shape is provider-specific |
| `build_merge_request_context(project_id, mr_iid)` helper | Uses GitLab API and GitLab response fields |
| `extract_clone_url_from_mr(...)` helper | Clone URL source is GitLab MR response data |
| `is_merge_request_note(...)` helper | GitLab `noteable_type` rules are provider-specific |

Current code in `webhook_service.py` that fits this bucket:
- GitLab state mapping inside `handle_comment_event(...)`
- GitLab state mapping inside `process_gitlab_webhook(...)`
- GitLab state mapping inside `process_gitlab_comment_webhook(...)`
- note-event interpretation and MR payload assumptions
- clone URL healing from live MR response

What should stay out of `gitlab_service.py`:
- `Repository(...)` creation
- `Review(...)` creation
- `WebhookEvent(...)` creation
- DB session logic
- Celery task dispatch

### 2. `backend/app/integrations/github/github_service.py`
This file already has:
- `validate_webhook_signature(...)`
- `parse_webhook_payload(...)`
- `get_pull_request(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`

Add more GitHub-only helpers here:

| Move or add | Why it belongs in `github_service.py` |
| --- | --- |
| `is_pull_request_issue_comment(payload)` helper | GitHub uses `issue_comment` for both issues and PRs |
| `extract_comment_trigger_context(payload)` helper | GitHub comment payload shape is provider-specific |
| `map_pull_request_state(pr_data)` helper | GitHub `state` plus `merged` rules are provider-specific |
| `split_owner_repo(project_id)` helper | `owner/repo` parsing is GitHub-specific |
| `build_pull_request_context(owner, repo, pr_number)` helper | Uses GitHub API and GitHub response fields |

Current code in `webhook_service.py` that fits this bucket:
- GitHub PR vs issue comment detection
- `owner/repo` parsing
- GitHub merged or closed state mapping
- GitHub comment trigger parsing rules
- live PR context fetch preparation

Very important cleanup:
- `process_github_webhook(...)` should actually call `GitHubService.validate_webhook_signature(...)`

What should stay out of `github_service.py`:
- DB lookup for `Repository`, `Credential`, `Review`, `WebhookEvent`
- token decrypt policy
- review status transitions
- Celery task dispatch

### 3. `backend/app/integrations/bitbucket/bitbucket_service.py`
This file already has:
- `validate_webhook_signature(...)`
- `parse_webhook_payload(...)`
- `get_pull_request(...)`
- `post_review_started_comment(...)`
- `post_review_acknowledgment_comment(...)`

Add more Bitbucket-only helpers here:

| Move or add | Why it belongs in `bitbucket_service.py` |
| --- | --- |
| `map_event_action(x_event_key)` helper | `pullrequest:*` event format is Bitbucket-specific |
| `map_pull_request_state(pr_data)` helper | Bitbucket state strings are provider-specific |
| `extract_comment_trigger_context(payload)` helper | Bitbucket comment payload shape is provider-specific |
| `extract_workspace_repo(payload)` helper | `workspace/repo_slug` handling is Bitbucket-specific |
| `build_pull_request_context(workspace, repo_slug, pr_id)` helper | Uses Bitbucket API and Bitbucket response fields |

Current code in `webhook_service.py` that fits this bucket:
- `pullrequest:` prefix checks
- `fulfilled`, `rejected`, `declined`, `comment_created` action mapping
- workspace and repo slug extraction assumptions
- inline Bitbucket comment-trigger parsing

What should stay out of `bitbucket_service.py`:
- DB queries and object creation
- review retry or reuse policy
- webhook idempotency records
- Celery task dispatch

## What Should Stay In `webhook_service.py`
Keep `WebhookService` as the orchestration layer used by controllers:

- `health_check(...)`
- `process_gitlab_webhook(...)`
- `process_gitlab_comment_webhook(...)`
- `process_github_webhook(...)`
- `process_bitbucket_webhook(...)`

But these methods should become thin. Their job should be:

1. call provider service to validate and parse
2. call shared review-domain helpers to create or update records
3. call provider service to post comment if needed
4. enqueue background task
5. return response

## What Should Move To Shared Review Services Instead
Some code in `webhook_service.py` should move out, but not into provider files.

### Move to `webhook_review_service.py`
Put shared review workflow here:
- create `Review`
- update latest review state
- maybe create or reuse review for comment-trigger retries

Why:
- this logic is repeated across providers
- it is application logic, not provider logic

### Move to `webhook_event_service.py`
Put shared idempotency logic here:
- create provider event IDs
- check `WebhookEvent` existence
- persist processed event records

Why:
- all three providers repeat the same event-bookkeeping pattern

### Move to `webhook_trigger_service.py`
Put shared bot trigger rules here:
- bot mention detection
- keyword detection for `review`, `analyze`, `check`

Why:
- provider payload extraction is provider-specific
- trigger decision rules are shared business rules

### Move to `webhook_author_resolver.py`
Move `resolve_author_email(...)` here.

Why:
- it uses DB lookups and identity resolution rules
- it should not stay buried inside a giant webhook orchestration class

### Move to `webhook_r0_service.py`
Collapse the three `_trigger_r0_extraction_on_merge*` methods into one shared helper.

Why:
- logic is nearly identical
- only input ID and log labels differ

## Recommended Method-Level Refactor

| Current method or block | Target after cleanup |
| --- | --- |
| `_trigger_r0_extraction_on_merge(...)` | move to shared `webhook_r0_service.py` |
| `_trigger_r0_extraction_on_merge_github(...)` | merge into same shared R0 helper |
| `_trigger_r0_extraction_on_merge_bitbucket(...)` | merge into same shared R0 helper |
| `get_repository_token(...)` | keep shared, or move to a small credential helper under review services |
| `resolve_author_email(...)` | move to `webhook_author_resolver.py` |
| `handle_comment_event(...)` | keep only orchestration here; move GitLab payload and MR context rules to `gitlab_service.py` |
| `process_gitlab_webhook(...)` | keep as thin entry point |
| `process_gitlab_comment_webhook(...)` | keep as thin entry point or delegate to a single GitLab comment flow |
| `handle_github_comment_event(...)` | keep orchestration here; move GitHub comment parsing and PR context rules to `github_service.py` |
| inline Bitbucket comment block | extract provider parsing to `bitbucket_service.py`, shared review actions to shared helper |
| `process_github_webhook(...)` | keep as thin entry point and add actual signature validation |
| `process_bitbucket_webhook(...)` | keep as thin entry point |

## Best First Cleanup Steps

### Phase 1: low-risk cleanup
Start with:
- remove unused imports `get_db` and `WebhookPayload`
- replace debug `print(...)` calls with `logger`
- make GitHub signature validation real
- collapse GitLab comment handling into one flow

Why first:
- low behavior risk
- immediate readability win

### Phase 2: extract shared services
Create:
- `webhook_review_service.py`
- `webhook_event_service.py`
- `webhook_trigger_service.py`
- `webhook_author_resolver.py`
- `webhook_r0_service.py`

Why second:
- repeated DB workflow shrinks fast
- makes later provider extraction much cleaner

### Phase 3: expand provider services
Add the provider-only helpers listed above to:
- `gitlab_service.py`
- `github_service.py`
- `bitbucket_service.py`

Why third:
- by then shared app logic is already separated
- easier to see what is truly provider-specific

### Phase 4: thin `WebhookService`
After extraction, each provider entry method should mainly read like:

1. validate request
2. parse provider payload
3. build provider context
4. call shared review workflow
5. post provider comment
6. enqueue task
7. return response

## What Not To Do

1. Do not move SQLAlchemy DB writes into provider service files.
2. Do not move Celery dispatch into provider service files.
3. Do not rewrite response JSON contracts during the cleanup.
4. Do not change credential storage format as part of this refactor.
5. Do not try to solve all duplication with one giant abstract base class immediately.

## Final Recommendation
Yes, `webhook_service.py` should be cleaned by moving GitLab, GitHub, and Bitbucket specific code closer to:

- `backend/app/integrations/gitlab/gitlab_service.py`
- `backend/app/integrations/github/github_service.py`
- `backend/app/integrations/bitbucket/bitbucket_service.py`

But the clean design is not "move everything there".

The clean design is:

- provider-specific webhook rules live in provider service files
- shared review and DB workflow live in smaller review service files
- `WebhookService` stays as a thin orchestrator

That split will make the code easier to read, easier to test, and much safer to extend later.
