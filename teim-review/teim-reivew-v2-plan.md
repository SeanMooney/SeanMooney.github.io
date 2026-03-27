# Context

After running `/teim-review` on the `teim-review-agent` branch, the review surfaced 8 issues.
This plan addresses each finding — separating clear fixes from items needing user input.

---

## Finding Assessment

### Finding 1 (High) — SKILL.md: Missing `json_schema` parameter
**Verdict: Genuine, partial.**
- `json_schema` IS a documented parameter of `teim-review-agent.md` and `schemas/review-report-schema.json` exists.
- `tools_dir` is NOT a documented agent parameter — the reviewer hallucinated it. Ignore.
- **Fix**: Add `- **json_schema**: `./schemas/review-report-schema.json`` to `skills/teim-review/SKILL.md`.
- **File**: `skills/teim-review/SKILL.md`

### Finding 2 (Warning) — plugin.json: Missing `agents_dir`/`skills_dir`
**Verdict: False positive per canonical spec.**
- Canonical plugin spec uses field names `agents` and `skills` (not `agents_dir`/`skills_dir`).
- Both are optional overrides when using default locations — defaults ARE `agents/` and `skills/`.
- This repo uses default paths, so the plugin loads correctly without these fields.
- **Action**: No change required.

### Finding 3 (Warning) — `changed_when: true` in ai_review_setup
**Verdict: Genuine Ansible idempotency issue.**
- `plugin marketplace add` and `plugin install` both have `changed_when: true` unconditionally.
- Both commands are idempotent (add is a no-op if marketplace already registered; install updates if needed).
- **Files**: `roles/ai_review_setup/tasks/main.yaml` lines 76 and 85
- **Fix**: Register result, use `changed_when: _result.stdout is search('installed|updated')`.
  Note: exact stdout pattern may need adjustment after verifying actual claude CLI output.

### Finding 4 (Warning, HTML-only) — Stale `ai_context_extraction` reference
**Verdict: Genuine stale doc.**
- `docs/future-test-improvements.md` line 34 still names the deleted `ai_context_extraction` role.
- **Fix**: Remove the `ai_context_extraction` bullet from the list; update the section header if needed.
- **File**: `docs/future-test-improvements.md`

### Finding 5 (Suggestion) — AGENTS.md: Model strategy undocumented
**Verdict: Valid suggestion.**
- `AGENTS.md` exists but doesn't mention the haiku/inherit model split or `ANTHROPIC_DEFAULT_HAIKU_MODEL`.
- This info is in `agents/teim-review-agent.md` but not in the repo-level AGENTS.md.
- **Proposed fix**: Add a "Model Selection" section to `AGENTS.md` explaining the strategy.
- **File**: `AGENTS.md`

### Finding 6 (Suggestion) — HTML renderer script copied to home dir
**Verdict: Valid suggestion.**
- `roles/ai_html_generation/tasks/main.yaml:13` copies script to `{{ ansible_user_dir }}/render_html_from_json.py`.
- This leaves an artifact at `~/render_html_from_json.py` after the run.
- **Fix**: Remove the copy task entirely. Invoke the script directly via its fully-qualified
  source path: `{{ ai_html_generation_style_guide_project }}/tools/render_html_from_json.py`.
  The style guide repo is cloned at a known location in CI, so no copy is needed.

---

## Decisions Made

| Finding | Decision |
|---|---|
| plugin.json fields | Leave as-is (false positive per canonical spec) |
| `changed_when` | Parse stdout — `changed_when: _result.stdout is search('installed\|updated')` (pattern is a best guess; exact claude CLI output format unverified) |
| HTML script artifact | Remove copy task; invoke script directly from style guide project path |

---

## Files to Modify (confirmed fixes)

| File | Change |
|---|---|
| `skills/teim-review/SKILL.md` | Add `json_schema` parameter |
| `docs/future-test-improvements.md` | Remove stale `ai_context_extraction` reference |
| `AGENTS.md` | Add model selection strategy section |
| `roles/ai_review_setup/tasks/main.yaml` | Register result; `changed_when: _result.stdout is search('installed\|updated')` on both tasks |
| `roles/ai_html_generation/tasks/main.yaml` | Remove copy task; update invoke path to style guide project source |
