# Clean Code Docs Index

## Purpose
This is the master index for all clean-code handoff documents in this repository.

Use this file when you want to quickly understand:

- which markdown files exist
- what each markdown file is for
- which markdown files to give the agent for each target file

## Full Document List

| File | Purpose |
| --- | --- |
| `AI_REVIEWER_CLEAN_CODE_REFERENCE.md` | current-state reference for `backend/app/review/services/ai_reviewer.py` |
| `AI_REVIEWER_CLEAN_CODE_STRATEGY.md` | refactor strategy for `backend/app/review/services/ai_reviewer.py` |
| `CHUNKED_REVIEWER_CLEAN_CODE_REFERENCE.md` | current-state reference for `backend/app/review/services/chunked_reviewer.py` |
| `CHUNKED_REVIEWER_CLEAN_CODE_STRATEGY.md` | refactor strategy for `backend/app/review/services/chunked_reviewer.py` |
| `WEBHOOK_SERVICE_CLEAN_CODE_REFERENCE.md` | current-state reference for `backend/app/review/services/webhook_service.py` |
| `WEBHOOK_SERVICE_CLEAN_CODE_STRATEGY.md` | refactor strategy for `backend/app/review/services/webhook_service.py` |
| `CLEAN_CODE_AGENT_MASTER_CONTEXT.md` | big-picture context across all three target files plus target file split |
| `CLEAN_CODE_AGENT_FINAL_OUTCOME.md` | end-state goals, done-definition, success criteria |
| `CLEAN_CODE_AGENT_GUIDELINES.md` | strict do and do-not rules for safe clean-code refactoring |
| `CLEAN_CODE_VALIDATION_HANDOFF.md` | post-refactor validation guide for import safety, naming consistency, contract checks, and behavior parity |

## Which Files To Give The Agent

### If the target is `backend/app/review/services/ai_reviewer.py`
Give the agent these 5 files:

1. `AI_REVIEWER_CLEAN_CODE_REFERENCE.md`
2. `AI_REVIEWER_CLEAN_CODE_STRATEGY.md`
3. `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
4. `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
5. `CLEAN_CODE_AGENT_GUIDELINES.md`

### If the target is `backend/app/review/services/chunked_reviewer.py`
Give the agent these 5 files:

1. `CHUNKED_REVIEWER_CLEAN_CODE_REFERENCE.md`
2. `CHUNKED_REVIEWER_CLEAN_CODE_STRATEGY.md`
3. `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
4. `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
5. `CLEAN_CODE_AGENT_GUIDELINES.md`

### If the target is `backend/app/review/services/webhook_service.py`
Give the agent these 5 files:

1. `WEBHOOK_SERVICE_CLEAN_CODE_REFERENCE.md`
2. `WEBHOOK_SERVICE_CLEAN_CODE_STRATEGY.md`
3. `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
4. `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
5. `CLEAN_CODE_AGENT_GUIDELINES.md`

## What Each Group Gives The Agent

### Per-file reference doc
This tells the agent:

- what the file currently contains
- where the file fits in the system
- what responsibilities are mixed together
- what duplication or boundary problems exist

### Per-file strategy doc
This tells the agent:

- how to refactor the file safely
- what should stay in the file
- what should move out
- which new helper files should be created
- which logic belongs in provider files versus review-service files

### Master context doc
This tells the agent:

- the shared clean-code direction across all three targets
- the expected target structure
- which helper files should exist across the system

### Final outcome doc
This tells the agent:

- what success looks like
- what done means
- what the final cleaned architecture should achieve

### Guidelines doc
This tells the agent:

- only clean code
- do not change logic
- do not change database, migration, Docker, env, or infrastructure behavior
- prefer extraction over rewrite
- keep old behavior stable

### Validation handoff doc
This tells the agent:

- how to verify imports still work after extraction
- how to verify moved names are updated consistently
- how to verify contracts and return shapes are still compatible
- how to check that behavior still matches old flow as closely as possible

## Recommended Use Pattern

When starting work on one file, the agent should read in this order:

1. this index file
2. the target file's reference doc
3. the target file's strategy doc
4. `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
5. `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
6. `CLEAN_CODE_AGENT_GUIDELINES.md`

That gives the agent:

- current situation
- desired file split
- global architecture direction
- safety rules

After the refactor work is complete, use:

7. `CLEAN_CODE_VALIDATION_HANDOFF.md`

That gives the agent the post-refactor validation checklist.

## Short Version

### For `ai_reviewer.py`
Use:
- `AI_REVIEWER_CLEAN_CODE_REFERENCE.md`
- `AI_REVIEWER_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
- `CLEAN_CODE_AGENT_GUIDELINES.md`

### For `chunked_reviewer.py`
Use:
- `CHUNKED_REVIEWER_CLEAN_CODE_REFERENCE.md`
- `CHUNKED_REVIEWER_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
- `CLEAN_CODE_AGENT_GUIDELINES.md`

### For `webhook_service.py`
Use:
- `WEBHOOK_SERVICE_CLEAN_CODE_REFERENCE.md`
- `WEBHOOK_SERVICE_CLEAN_CODE_STRATEGY.md`
- `CLEAN_CODE_AGENT_MASTER_CONTEXT.md`
- `CLEAN_CODE_AGENT_FINAL_OUTCOME.md`
- `CLEAN_CODE_AGENT_GUIDELINES.md`

## Final Note
If the clean-code task is only for one file, do not overload the agent with unrelated file-specific strategy docs.

Give:

- the two docs for that target file
- the three shared agent docs

That is the cleanest handoff pack for focused refactoring.
