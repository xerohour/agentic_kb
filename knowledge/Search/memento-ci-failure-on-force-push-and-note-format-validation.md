---
title: Memento CI failure on force-push and note format validation
type: note
domain: Search
tags:
  - github-actions
  - memento
  - git-notes
  - ci
  - force-push
status: draft
created: 2026-03-03
updated: 2026-03-03
---

# Memento CI failure on force-push and note format validation

## Overview
This note captures a repeatable CI failure pattern when using `mandel-macaque/memento` workflows on pull requests that are force-pushed (for example, after rebase). It also captures the strict git-note format required by `enforce-memento-notes`.

## Problem / Context
Two independent failures occurred:

1. `comment-memento-notes` failed with:
   - `fatal: Invalid revision range <old_sha>..<new_sha>`
2. `enforce-memento-notes` failed with:
   - `invalid-note ... missing-provider-marker,missing-session-id-marker`

The first was triggered by running comment workflow on `push` events after a force-push where the previous SHA was no longer reachable.  
The second was caused by a git note that existed but did not match required memento structure markers.

## Steps
1. Limit comment workflow to PR events only.
2. Push workflow fix and rerun failed checks.
3. Ensure each PR commit has a properly structured note in `refs/notes/commits`.
4. Push notes ref, rerun failed `enforce-memento-notes`, verify all checks pass.

### 1) Workflow trigger fix
In `.github/workflows/memento-note-comments.yml`, remove `push` trigger and keep `pull_request` only.

### 2) Add or fix commit note format
Use this structure (required markers):

```text
<!-- git-memento-sessions:v1 -->
<!-- git-memento-note-version:1 -->
<!-- git-memento-session:start -->
- Provider: codex
- Session ID: <uuid>

# Codex Session Transcript

Session metadata note attached for CI audit compliance.

Summary:
- <change 1>
- <change 2>
<!-- git-memento-session:end -->
```

### 3) Commands used
```powershell
# Add or replace note for commit
git notes --ref=refs/notes/commits add -f -F note.txt <commit_sha>

# Push notes so CI can fetch them
git push origin refs/notes/commits

# Rerun failed workflow
gh run rerun <run_id> --repo <owner>/<repo>

# Watch PR checks
gh pr checks <pr_number> --repo <owner>/<repo> --watch --interval 10
```

### 4) Verification checklist
- `comment-memento-notes` is green.
- `enforce-memento-notes` is green.
- No `invalid revision range` errors in logs.
- No `missing-provider-marker` / `missing-session-id-marker` errors in logs.

## References
- `agentic_kb/knowledge/Search/kb-search-trigger-policy-and-sandbox-safe-uv.md`
- `agentic_kb/knowledge/Search/agent-retrieval-workflow.md`

## Related
- [[kb-search-trigger-policy-and-sandbox-safe-uv]]
- [[agent-retrieval-workflow]]
