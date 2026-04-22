# WebhookService Current Code Reference

## Purpose
This document describes what is currently inside `backend/app/review/services/webhook_service.py` and why the file feels too large today.

Use this file as the "current state" reference before refactoring.

## File Facts
- File: `backend/app/review/services/webhook_service.py`
- Current size: 1582 lines
- Main class: `WebhookService`
- Main callers: `backend/app/review/controllers/webhooks.py`

## Controller Entry Points
`backend/app/review/controllers/webhooks.py` calls these methods:

| Route | WebhookService method |
| --- | --- |
| `/webhooks/gitlab` | `process_gitlab_webhook(...)` |
| `/webhooks/gitlab/comment` | `process_gitlab_comment_webhook(...)` |
| `/webhooks/github` | `process_github_webhook(...)` |
| `/webhooks/bitbucket` | `process_bitbucket_webhook(...)` |

So `WebhookService` is the application entry point for all webhook providers.

## Method Map

| Line | Method | Current responsibility |
| --- | --- | --- |
| 24 | `health_check` | Simple health response |
| 28 | `_trigger_r0_extraction_on_merge` | GitLab merge-triggered R0 refresh |
| 75 | `_trigger_r0_extraction_on_merge_github` | GitHub merge-triggered R0 refresh |
| 117 | `_trigger_r0_extraction_on_merge_bitbucket` | Bitbucket merge-triggered R0 refresh |
| 159 | `get_repository_token` | Credential lookup and token decrypt fallback |
| 180 | `resolve_author_email` | Resolve author email using DB lookup and webhook data |
| 255 | `handle_comment_event` | GitLab note trigger flow |
| 438 | `process_gitlab_webhook` | Main GitLab MR webhook flow |
| 682 | `process_gitlab_comment_webhook` | Dedicated GitLab comment webhook flow |
| 875 | `handle_github_comment_event` | GitHub PR comment trigger flow |
| 1027 | `process_github_webhook` | Main GitHub PR webhook flow |
| 1253 | `process_bitbucket_webhook` | Main Bitbucket PR webhook flow, including comment trigger |

## What The File Contains Today

### 1. Shared webhook helpers
These parts are not tied to one provider:

- `health_check(...)`
- `get_repository_token(...)`
- `resolve_author_email(...)`

What they do:
- decrypt stored credential tokens
- fall back to global token config
- query `User` records for author email resolution
- normalize author identity before creating `Review`

### 2. Three near-duplicate R0 merge handlers
These methods all do almost the same thing:

- `_trigger_r0_extraction_on_merge(...)`
- `_trigger_r0_extraction_on_merge_github(...)`
- `_trigger_r0_extraction_on_merge_bitbucket(...)`

Shared behavior:
- find repository by provider project identifier
- skip missing or inactive repositories
- enqueue `extract_single_repository_context_task.apply_async(...)`
- log failures without breaking webhook processing

Difference between them:
- only the project ID format and log message text

This is one of the clearest duplication areas in the file.

### 3. GitLab-specific webhook logic
GitLab logic is spread across three places:

- `handle_comment_event(...)`
- `process_gitlab_webhook(...)`
- `process_gitlab_comment_webhook(...)`

Current GitLab responsibilities inside `WebhookService`:
- validate GitLab webhook token
- parse GitLab payload using `GitLabService.parse_webhook_payload(...)`
- detect note events that mention `@webelight-bot`
- check trigger keywords like `review`, `analyze`, `check`
- fetch MR details using `GitLabService.get_merge_request(...)`
- auto-create `Repository` when missing
- auto-heal placeholder `clone_url` from live MR data
- resolve GitLab author email from multiple sources
- create `WebhookEvent`
- create `Review`
- post acknowledgment or "review started" comments
- enqueue `process_code_review.delay(...)`
- update MR state on close or merge
- trigger R0 refresh on merge

Important note:
- GitLab comment-trigger flow is duplicated in `handle_comment_event(...)` and `process_gitlab_comment_webhook(...)`
- `process_gitlab_webhook(...)` also routes note events

So GitLab comment handling is currently the most duplicated part of the file.

### 4. GitHub-specific webhook logic
GitHub logic is split into:

- `handle_github_comment_event(...)`
- `process_github_webhook(...)`

Current GitHub responsibilities inside `WebhookService`:
- read raw body and parsed JSON
- route `issue_comment` events to comment-trigger handling
- parse PR payload using `GitHubService.parse_webhook_payload(...)`
- detect PR comment trigger text
- verify the comment belongs to a PR and not a normal issue
- split `owner/repo`
- fetch PR details using `GitHubService.get_pull_request(...)`
- map PR state to internal `MRState`
- create `WebhookEvent`
- create `Review`
- load credential and decrypt token
- post GitHub comments through `GitHubService`
- enqueue `process_code_review.delay(...)`
- update latest review when PR closes
- trigger R0 refresh on merge

Important note:
- `x_hub_signature_256` and raw body are read in `process_github_webhook(...)`
- `GitHubService.validate_webhook_signature(...)` exists
- but GitHub signature validation is not actually called in this method

That is both a cleanup issue and a security consistency issue.

### 5. Bitbucket-specific webhook logic
Bitbucket logic is inside one large method:

- `process_bitbucket_webhook(...)`

Current Bitbucket responsibilities inside `WebhookService`:
- read raw body and JSON
- validate event key prefix like `pullrequest:*`
- parse payload with `BitbucketService.parse_webhook_payload(...)`
- map `fulfilled`, `rejected`, `declined`, `comment_created`
- update latest review state when PR closes or merges
- trigger R0 refresh on merge
- handle comment-trigger flow inline inside the same method
- create idempotency `WebhookEvent`
- create `Review`
- load credential and decrypt token
- validate Bitbucket webhook signature
- post Bitbucket comments through `BitbucketService`
- enqueue `process_code_review.delay(...)`

Important note:
- Bitbucket comment-trigger logic is not in a helper method
- it is embedded directly inside the main Bitbucket webhook processor

So Bitbucket is less duplicated than GitLab, but still too crowded.

## Shared Database And Workflow Logic Inside The File
`WebhookService` also owns application-domain work that is not provider API work:

- `Repository` lookup
- `Repository` auto-creation for GitLab
- `Credential` lookup
- `Review` creation
- latest `Review` state updates
- `WebhookEvent` idempotency storage
- Celery task dispatch through `process_code_review.delay(...)`

These are core application responsibilities, not integration-adapter responsibilities.

## Which Existing Integration Services Are Already Used
The file already depends on these integration services:

| File | Existing provider-specific capabilities already there |
| --- | --- |
| `backend/app/integrations/gitlab/gitlab_service.py` | `validate_webhook_token`, `parse_webhook_payload`, `get_merge_request`, comment posting helpers |
| `backend/app/integrations/github/github_service.py` | `validate_webhook_signature`, `parse_webhook_payload`, `get_pull_request`, comment posting helpers |
| `backend/app/integrations/bitbucket/bitbucket_service.py` | `validate_webhook_signature`, `parse_webhook_payload`, `get_pull_request`, comment posting helpers |

This means the project already has the right boundary started: provider-only behavior belongs in provider service files.

## Repeated Patterns Inside webhook_service.py
The same overall workflow repeats across GitLab, GitHub, and Bitbucket:

1. Validate webhook request.
2. Parse provider payload.
3. Extract repository and PR or MR identity.
4. Find repository in DB.
5. Find credential in DB.
6. Resolve access token.
7. Fetch live PR or MR details if needed.
8. Create or update `Review`.
9. Post provider comment.
10. Trigger background review task.

This repeated pattern is a strong sign that orchestration and provider-specific details should be separated more clearly.

## Current Clean Code Problems In The File

### Mixed responsibilities
One class currently mixes:
- provider payload parsing behavior
- provider state mapping
- provider comment-trigger rules
- DB writes
- token decryption
- task scheduling
- review orchestration

### Duplication
The biggest duplication spots are:
- three R0 trigger helpers
- GitLab comment flow in two methods
- repeated trigger-keyword detection
- repeated review creation pattern

### Inconsistent provider handling
- GitLab comment flow has two overlapping handlers
- GitHub comment flow has its own dedicated handler
- Bitbucket comment flow is inline inside the main method

### Debug and cleanup debt
Current file also contains:
- debug `print(...)` statements in `handle_comment_event(...)`
- unused imports: `get_db`, `WebhookPayload`
- missing GitHub signature validation even though helper exists

## What Should Stay True After Refactor
Even after cleanup, this file should still remain the webhook application entry point used by controllers. What should change is not the public role of the file, but how much provider-specific logic it directly owns.

## Bottom Line
`webhook_service.py` currently contains:

- shared webhook helpers
- shared DB and review workflow
- GitLab-specific webhook processing
- GitHub-specific webhook processing
- Bitbucket-specific webhook processing
- duplicate comment-trigger logic
- duplicate merge-trigger logic

That is why the file is hard to maintain today. The main clean-code opportunity is to keep orchestration here, while moving provider-specific webhook behavior closer to `gitlab_service.py`, `github_service.py`, and `bitbucket_service.py`, and moving shared DB workflow into smaller review-domain helper services.
