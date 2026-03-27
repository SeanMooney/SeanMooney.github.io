# Plan: Fix review findings + model optimisation (amend commit)

## Context

The teim-review-agent ran a self-review of commit 471467b and found four issues.
The user also wants the lightweight context/guidelines subagents to default to
`haiku` (cheap/fast) while only the code-review-agent inherits the session model
(the capable model chosen by the user or CI job).

**Design constraint**: Claude Code resolves a subagent's model from its own
`model:` frontmatter — the orchestrating agent cannot override it at call time.
So model assignment must live in the individual agent definitions. The
teim-review-agent can document the convention and note that users may select a
more capable context model by setting `ANTHROPIC_DEFAULT_HAIKU_MODEL` in their
environment.

---

## Changes (all in one amended commit)

### 1. Fix: Amend commit with DCO sign-off and AI attribution

The amended commit message must add:
- `Signed-off-by: <name> <email>` (DCO — required per CLAUDE.md)
- `Generated-by: claude sonnet 4.6` (OpenInfra AI policy attribution,
  per `docs/quick-rules.md` convention)

Use `git commit --amend` with the full updated message (not `--signoff` alone,
since we also need to insert `Generated-by`). Run `git log -1` first to get the
author name/email for the Signed-off-by trailer.

### 2. Fix: Replace stale post-task status block with jq-based stats

**File:** `playbooks/teim-code-review/run.yaml` lines ~96-111

The `post_tasks` block references `ai_code_review_issue_statistics`,
`ai_code_review_has_critical_issues`, and `ai_code_review_has_issues` which are
never set by the new single-invocation flow. Replace with:

1. A `ansible.builtin.command` task that runs `jq` to extract issue counts
   from `review_report_file` (critical, high, warnings, suggestions, total).
   Use `extract_review_data` logic: try `.statistics` directly, fall back to
   `.structured_output.statistics` if the Claude CLI wrapper is present.
2. A `debug` task that prints the counts and an overall status line
   (CLEAN / HAS SUGGESTIONS / NEEDS ATTENTION) derived from the counts.

### 3. Fix: Update stale comment in ai_code_review defaults

**File:** `roles/ai_code_review/defaults/main.yaml` line 7

Comment references the deleted `ai_context_extraction` role. Update to describe
the current flow (inventory is copied by the playbook before the role runs).

### 4. Fix: Remove or replace `${CLAUDE_PLUGIN_ROOT}` in skill

**File:** `skills/teim-review/SKILL.md` line ~21

`${CLAUDE_PLUGIN_ROOT}` is not a documented Claude Code runtime variable. Remove
the fallback sentence — the skill is only meaningful when installed from the
plugin (the docs/ files will always be present relative to the plugin root).

### 5. Optimise: Set lightweight subagents to `model: haiku`

Agents that do cheap extraction/summarisation tasks should default to `haiku`
so CI uses `glm-4.7-flash` (via `ANTHROPIC_DEFAULT_HAIKU_MODEL`) and local use
gets `claude-haiku-4-5-*` automatically:

| File | Change |
|------|--------|
| `agents/zuul-context-extractor.md` | `model: inherit` → `model: haiku` |
| `agents/commit-summary.md` | `model: inherit` → `model: haiku` |
| `agents/project-guidelines-extractor.md` | `model: inherit` → `model: haiku` (revert #1 from prior session) |

**Keep as `model: inherit`** (needs the full session model):
- `agents/code-review-agent.md`
- `agents/security-auditor.md`
- `agents/code-maintainability-auditor.md`
- `agents/teim-review-agent.md`

### 6. Document the model strategy in teim-review-agent

**File:** `agents/teim-review-agent.md`

Add a brief note in the Parameters section explaining:
- Context/extraction subagents use `model: haiku` (maps to
  `ANTHROPIC_DEFAULT_HAIKU_MODEL` in CI, claude-haiku locally)
- The code-review-agent inherits the session model
- To use a more capable model for context extraction, set
  `ANTHROPIC_DEFAULT_HAIKU_MODEL` to the desired model name

---

## Critical files

| File | Change |
|------|--------|
| `playbooks/teim-code-review/run.yaml` | Remove stale post_tasks status block |
| `roles/ai_code_review/defaults/main.yaml` | Fix stale comment |
| `skills/teim-review/SKILL.md` | Remove `${CLAUDE_PLUGIN_ROOT}` fallback |
| `agents/zuul-context-extractor.md` | `model: haiku` |
| `agents/commit-summary.md` | `model: haiku` |
| `agents/project-guidelines-extractor.md` | `model: haiku` |
| `agents/teim-review-agent.md` | Add model strategy note |

---

## Verification

1. `uvx tox -e linters` — all pre-commit hooks pass
2. `uvx tox -e py3` — all unit tests pass
3. `git show --stat HEAD` — amended commit includes Signed-off-by
4. Inspect `agents/zuul-context-extractor.md`, `commit-summary.md`,
   `project-guidelines-extractor.md` to confirm `model: haiku`
