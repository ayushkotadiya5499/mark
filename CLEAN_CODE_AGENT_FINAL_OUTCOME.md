# Clean Code Agent Final Outcome

## Purpose
This file defines the final outcome we want from the clean-code work on:

- `backend/app/review/services/ai_reviewer.py`
- `backend/app/review/services/chunked_reviewer.py`
- `backend/app/review/services/webhook_service.py`

This is the "what success looks like" document for the agent.

## Final Goal
After cleanup, the system should keep the old behavior, but the code should be much easier to understand, test, and extend.

The final result we want is:

- the main service files stay as orchestrators
- helper logic lives in helper files
- provider-specific logic lives in provider services
- private cross-class dependencies are reduced
- duplicated logic is extracted
- behavior remains compatible with current callers

## What We Want To Achieve

### For `ai_reviewer.py`
We want to achieve this:

- `AIReviewer` is clearly the facade for review orchestration
- JSON cleanup and Claude response parsing are no longer buried inside it
- prompt loading and prompt building are separated
- comment formatting is its own concern
- provider-specific description updates no longer live in `AIReviewer`
- Claude CLI execution is isolated from review business logic

### For `chunked_reviewer.py`
We want to achieve this:

- `ChunkedReviewer` clearly controls the large-PR pipeline only
- chunk prompt creation is separated
- merge logic is separated
- verification logic is separated
- result mapping is separated
- R0 context resolution is separated
- dependence on `AIReviewer` private methods is reduced or removed

### For `webhook_service.py`
We want to achieve this:

- `WebhookService` clearly coordinates provider webhook flow only
- GitLab, GitHub, and Bitbucket webhook-specific rules are moved closer to their provider files
- shared trigger logic is extracted
- shared idempotency logic is extracted
- shared author resolution is extracted
- shared merge-trigger R0 scheduling is extracted
- provider branches become consistent and easier to follow

## Non-Negotiable Guardrails
The cleanup should follow these rules:

1. Preserve current behavior unless a change is explicitly intended.
2. Preserve current public entry points and return shapes as much as possible.
3. Prefer extraction over rewrite.
4. Do not mix this cleanup with unrelated feature work.
5. Do not move SQLAlchemy and Celery application logic into provider service files.
6. Do not redesign prompts, scoring, chunking thresholds, or webhook contracts during structural cleanup.
7. Do not add new abstractions that are bigger and harder to understand than the old code.

## Done Definition For Each Target

### `ai_reviewer.py` is done when
- the file is much smaller and easier to scan
- JSON repair and response parsing are in dedicated helper files
- comment formatting is extracted
- provider description updates are moved to provider services
- prompt retrieval and prompt building are separated
- `AIReviewer` mostly reads like orchestration code

### `chunked_reviewer.py` is done when
- the large-PR pipeline order is still the same
- helper-heavy sections are extracted into dedicated files
- verification logic is not buried inside the main class
- result models and mapping logic are separated
- R0 context handling is separated
- the file depends on shared collaborators instead of many `AIReviewer` private methods

### `webhook_service.py` is done when
- provider-specific logic is thinner in the file
- duplicated GitLab comment flow is collapsed
- shared trigger logic is extracted
- shared event bookkeeping is extracted
- shared author resolution is extracted
- shared R0 merge scheduling is extracted
- the remaining methods read like orchestration, not giant provider scripts

## Overall Success Criteria
At the end of this work, the agent should be able to say all of the following are true:

- each main file has one clear reason to change
- helper code is grouped by concern
- provider integration code is closer to provider service files
- the review system is easier to test in smaller parts
- future changes will require editing fewer giant files
- new contributors can understand the code faster

## Expected Benefits
If this cleanup is done well, we should gain:

- faster onboarding for new agents and developers
- lower risk when changing one concern
- easier unit testing for parsing, formatting, merging, and trigger rules
- easier provider-specific changes
- less fear of touching `ai_reviewer.py`, `chunked_reviewer.py`, and `webhook_service.py`

## What The Agent Should Produce
When acting on one target file, the agent should ideally produce:

- extracted helper files with clear ownership
- thinner orchestration in the original main file
- minimal behavior drift
- small, logical commits or changesets
- clear explanation of what moved where

## Suggested Working Style For The Agent

1. Work on one target file at a time.
2. Move the easiest clearly-wrong concerns first.
3. Keep names practical and close to the current domain language.
4. Reuse existing service boundaries where they already exist.
5. Keep fallbacks and error handling intact in the first pass.

## Final Outcome Statement
The final outcome we want is not a rewrite.

The final outcome we want is this:

- old working code is preserved
- boundaries become clean
- helper modules become focused
- provider adapters become more complete
- the three large files become readable orchestrators

If the agent uses the per-file docs plus this outcome document, it should have enough context to clean the code with much less guesswork.
