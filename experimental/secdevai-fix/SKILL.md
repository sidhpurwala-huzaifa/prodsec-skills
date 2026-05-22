---
name: secdevai-fix
description: >
  Apply suggested security fixes from a prior code review. Use when the user
  wants to remediate security findings with before/after code diffs, severity
  filtering, and explicit approval before modifying code.
category: "security_auditing"
subcategory: "audit-workflow"
---

# Security Fix Application

Applies security fixes identified by a prior code review. Always shows a diff
preview and requires explicit user approval before modifying any file.

## When to Use This Skill

- Applying fixes for findings from a prior `secdevai-review` or equivalent
  security review
- Applying a filtered subset of fixes (e.g., only critical severity findings)
- Previewing proposed code changes before deciding whether to apply them

## Workflow

### Step 1: Verify Prerequisites

Confirm that a prior security review has been run and findings are available.
If no prior review exists, inform the user and suggest running a security
review first.

### Step 2: Apply Severity Filter (optional)

If the user specifies a severity level (critical, high, medium, low), limit
fixes to findings at that level or above. If no filter is specified, present
all available fixes.

### Step 3: Present Fixes

For each fix, show:

- **Finding**: Description and location (file and line number)
- **Severity**: The severity of the finding being fixed
- **OWASP / CWE reference**: Category and ID
- **Before**: The vulnerable code
- **After**: The fixed code
- **Security impact**: What the fix prevents

Example:

```markdown
## Fix #1 (Critical)

**Finding**: SQL Injection in `app.py:42`
**OWASP Category**: [A03: Injection](https://owasp.org/Top10/A03_2021-Injection/) ([CWE-89](https://cwe.mitre.org/data/definitions/89.html))

**Before**:
query = f"SELECT * FROM users WHERE id = {user_id}"

**After**:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

**Security impact**: Eliminates SQL injection by using a parameterized query.
The database driver handles escaping, preventing attacker-controlled input from
modifying query structure.
```

### Step 4: Require Explicit Approval

Before applying any changes:

- Show all proposed diffs
- Ask the user to confirm which fixes to apply
- Never modify code without explicit approval
- Never apply fixes in bulk without showing each change first

### Step 5: Apply Approved Fixes

After the user approves:

- Apply the approved code changes exactly as shown in the preview
- Report which fixes were applied and which were skipped
- If a fix cannot be applied cleanly (e.g., the file has changed), report the
  conflict and leave the file unchanged

## Important Rules

- **Always preview before modifying**: Show the exact diff before touching any
  file.
- **Never apply without approval**: Require explicit confirmation, even when the
  user says "apply all".
- **Respect scope**: Only fix findings from the current review session unless the
  user explicitly provides findings from elsewhere.
- **Report outcomes**: After applying, list what changed and what was skipped,
  with reasons.
