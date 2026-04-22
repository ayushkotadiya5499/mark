# Clean Code Agent Guidelines

## Purpose
This document is the safety guide for any agent working on clean-code refactors in this repository.

The goal is very strict:

> clean the code structure, but keep the old logic and behavior

This means the agent should improve readability, file boundaries, and maintainability without turning the task into a feature rewrite or architecture redesign.

## Core Rule
Only do structural clean-code refactoring.

Do not change business behavior unless a tiny change is absolutely required to preserve the old behavior after extraction.

## Main Intent
The intent of this cleanup is:

- make large files easier to understand
- move code to better files
- separate responsibilities
- reduce duplication
- improve naming and file boundaries
- preserve current outputs, flow, and integrations

This cleanup is not meant to:

- redesign the product
- change review behavior
- change webhook behavior
- change chunking behavior
- change provider behavior
- change deployment or infrastructure behavior

## Allowed Work
These changes are allowed:

- extract helper methods into new files
- split large classes into smaller focused collaborators
- move provider-specific code into provider service files
- move formatting logic into formatter files
- move parsing logic into parser files
- move merge helpers into merger files
- move shared utility logic into utility files
- remove dead imports
- remove obviously unused local variables
- replace duplicated code with shared helper calls
- improve internal naming if behavior stays the same
- add small comments where needed for clarity

## Not Allowed
These changes are not allowed as part of this cleanup:

- changing business logic
- changing decision logic
- changing API response contracts
- changing review scoring rules
- changing prompt meaning
- changing chunking thresholds or flow
- changing webhook trigger rules unless the goal is only to preserve existing behavior during extraction
- changing retry policy unless required to keep old behavior exactly
- changing authentication flow
- changing provider request behavior intentionally

## Strictly Out Of Scope
Do not change any of the following unless the user explicitly asks for it in a separate task:

- database schema
- SQLAlchemy models for new product behavior
- Alembic migrations
- migration files
- Docker files
- docker-compose files
- environment variable names
- `.env` handling
- deployment scripts
- CI or CD pipelines
- Redis/Postgres setup
- infrastructure config
- app startup behavior

In short:

- no DB redesign
- no migration work
- no Docker work
- no env/config redesign

## Preserve These Things
The agent should preserve:

- public entry points
- current method contracts where callers depend on them
- return shapes
- existing fallback behavior
- existing error handling behavior
- existing logging intent
- existing review pipeline order
- existing webhook pipeline behavior
- existing chunked-review flow

If a method is moved, the behavior should stay the same even if the file changes.

## Safe Refactor Strategy
Use this order whenever possible:

1. identify a helper concern inside a large file
2. move that concern into a focused helper file
3. keep the old method as a thin wrapper if needed
4. update callers with minimal surface change
5. verify behavior is preserved

Preferred approach:

- extraction first
- rewrite later only if necessary

## Good Changes
These are examples of good clean-code changes:

- move JSON repair helpers out of `ai_reviewer.py` into a utility file
- move review comment formatting into a formatter service
- move provider description update methods into provider service files
- move webhook trigger keyword logic into a dedicated trigger service
- move chunk merge helpers into a merger file
- move R0 context handling into a shared service

## Bad Changes
These are examples of changes the agent should avoid:

- changing how reviews are scored while extracting code
- changing prompt content because the old prompt looks messy
- changing database fields because the current structure feels awkward
- adding migrations because a refactor could be "cleaner" with schema changes
- changing Docker configuration during a Python file cleanup
- renaming environment variables during a service refactor
- replacing working flow with a new architecture in one pass

## File Movement Rules
When moving code:

- keep related logic together
- move code to the file that already matches the concern
- prefer existing service boundaries over creating random new files
- create new helper files only when the concern is clearly separate

Examples:

- provider API logic belongs in provider service files
- parsing belongs in parser files
- formatting belongs in formatter files
- pure merge logic belongs in merger files
- DB workflow belongs in review or webhook domain services, not provider adapters

## Provider Boundary Rule
Provider service files may contain:

- provider payload parsing
- provider API calls
- provider request validation
- provider state mapping
- provider-specific formatting needed for that provider

Provider service files must not become the place for:

- SQLAlchemy review creation logic
- webhook event persistence
- Celery orchestration
- general application workflow

## Database Boundary Rule
Do not move application database workflow into integration files.

Database-related workflow should stay in application-domain services such as review or webhook services.

That means:

- `Review` creation stays in review or webhook domain services
- `WebhookEvent` persistence stays in webhook domain services
- repository lookup and workflow stays in application services

## Refactor Style Rule
Prefer:

- smaller focused extractions
- thin wrappers
- stable caller contracts
- incremental cleanup

Avoid:

- large rewrites
- sweeping renames with no behavioral need
- changing multiple unrelated layers in one task

## Testing And Verification Rule
After refactoring, the agent should verify as much as possible without changing system behavior.

Good verification:

- run targeted tests if they exist
- run lint or type checks if lightweight and relevant
- compare inputs and outputs of moved helpers where possible
- confirm imports and call sites are updated correctly

Do not invent new behavior just to make tests pass.

## If The Agent Finds Bad Old Code
If the old code looks messy, repetitive, or awkward, the agent should still follow this rule:

> preserve behavior first, improve structure second

If the code has a real bug that is unrelated to the requested clean-code task, do not silently fix it as part of the refactor unless:

- it blocks the refactor, or
- the user explicitly asks for bug fixing too

Otherwise, note it separately.

## When To Pause
The agent should pause and avoid hidden changes if a refactor would require:

- changing DB schema
- adding a migration
- changing env contract
- changing Docker behavior
- changing external API behavior intentionally
- changing user-visible product behavior

In those cases, the correct action is to keep the cleanup structural only.

## Final Done Checklist
Before calling the cleanup complete, the agent should be able to say:

- logic is preserved
- main file is thinner
- helper concerns were extracted cleanly
- no DB schema changes were introduced
- no migrations were added
- no Docker files were changed
- no env contract was changed
- no unrelated feature work was mixed in

## Final Guiding Principle
This repository wants conservative clean-code refactoring.

That means:

- use old code
- preserve old logic
- improve structure only

If there is a choice between "more elegant" and "less risky", choose less risky.
