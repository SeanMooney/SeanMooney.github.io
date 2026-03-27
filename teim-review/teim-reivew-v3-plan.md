# Plan: Address teim-review findings iteratively

## Context

A teim-review run on the `teim-review-agent` branch surfaced 2 high and 3 warning findings.
Several are false positives; one is a real gap (no post-install verification). The goal is to:
- Fix the real issue (High #2)
- Make one small fix (SKILL.md schema path)
- Capture false positives in a new `HACKING.rst` so the review tool stops re-flagging them

---

## Changes

### 1. Create `HACKING.rst` (false positive catalogue)

**New file**: `HACKING.rst` (repo root)

The `project-guidelines-extractor` agent already reads `HACKING.rst` — this is the right place
for "known patterns that look wrong but are intentional."

Document the following false positives:

- **`plugin.json` missing `agents_dir`/`skills_dir`**: Auto-discovery uses `agents/` and
  `skills/` by default; explicit fields are only needed for non-default paths.
- **Jinja2 self-reference chain in `ai_code_review/defaults/main.yaml`**: Ansible resolves
  intra-file variable references at play time; this is valid and intentional.
- **`regex_search` may fail on empty stdout**: The playbook already guards with
  `| default('')` and `or ['0']` fallbacks; empty stdout is fully handled.

Use RST format (OpenStack convention). Keep entries concise.

---

### 2. Fix `skills/teim-review/SKILL.md` — schema path

**File**: `skills/teim-review/SKILL.md`

The line:
```
- **JSON schema**: `./schemas/review-report-schema.json`
```

is relative to CWD (the reviewed project), which is wrong. The schema lives in the plugin
repo. Fix: update the reference to be relative to the plugin installation directory.

When Claude Code installs the plugin, the plugin root is the repo itself (local install) or
`~/.claude/plugins/teim-review/` (marketplace install). The teim-review-agent already resolves
the schema for CI via role variables. For the SKILL.md instruction, change to:

```
- **JSON schema**: `schemas/review-report-schema.json` (relative to the plugin root)
```

This makes the intent clear without a broken CWD-relative path.

---

### 3. Fix High #2 — add post-install verification to `ai_review_setup` role

**File to locate**: the task file in `roles/ai_review_setup/tasks/` that runs
`claude plugin marketplace add` and `claude plugin install`.

Add assertion tasks after each install step:
- After `claude plugin marketplace add`: assert that the marketplace was added (check
  stdout or use a subsequent `claude plugin marketplace list` to verify)
- After `claude plugin install teim-review@openstack-ai-style-guide`: assert that the plugin
  appears in `claude plugin list` output

Use `ansible.builtin.assert` or `failed_when` on the register result with a meaningful
`fail_msg` so failures surface clearly in Zuul logs.

---

## Files to Modify

| File | Change |
|---|---|
| `HACKING.rst` | Create — false positive catalogue |
| `skills/teim-review/SKILL.md` | Update schema path reference |
| `roles/ai_review_setup/tasks/*.yaml` | Add post-install verification assertions |

---

## Verification

1. Run `uvx tox -e linters` — HACKING.rst should pass RST checks (if any) and markdownlint
   should be unaffected
2. Run `/teim-review` again after changes — verify the three false-positive findings no longer
   appear (project-guidelines-extractor will pick up HACKING.rst and suppress them)
3. Review the updated `ai_review_setup` task file visually to confirm assertions are
   syntactically valid Ansible
