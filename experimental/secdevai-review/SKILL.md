---
name: secdevai-review
description: >
  Use when performing security code review of source files, git commits, or
  entire codebases against OWASP Top 10 (2021), CWE/SANS Top 25, and OWASP
  WSTG v4.2 patterns; applicable to any language or framework.
category: "security_auditing"
subcategory: "audit-workflow"
---

# Security Code Review

AI-powered security review of source code using OWASP Top 10, CWE/SANS Top 25,
and OWASP WSTG patterns. Supports file review, commit review, and full codebase
scan modes across all major languages.

## When to Use This Skill

- Reviewing source code files or directories for security vulnerabilities
- Reviewing one or more git commits for security-relevant changes
- Scanning an entire codebase for OWASP Top 10 / CWE/SANS Top 25 issues
- Performing a security review pass before submitting a pull request

## Workflow

### Step 1: Determine Scope

Choose what to review (in priority order):

1. If a specific number of recent commits is requested: review those commits
2. If the last commit is requested: review the most recent commit
3. If a specific file or directory is specified: review it
4. Otherwise: scan the entire codebase

### Step 2: Apply Ignore Patterns

Before scanning, exclude:

- Binary and media files (images, fonts, compiled artifacts, archives)
- Generated code and build output (`dist/`, `build/`, `*.pyc`, `*.class`)
- Dependency trees (`node_modules/`, `venv/`, `__pycache__/`)
- IDE and OS metadata (`.vscode/`, `.idea/`, `.DS_Store`)

Note: documentation files (markdown, RST, plaintext) are **not** excluded —
they can contain hardcoded secrets, leaked infrastructure details, or AI agent
instructions.

If all files are excluded, report that no security-relevant content was found.
If some files are excluded, proceed with the remaining files and list the
skipped files at the end of the review.

### Step 3: Load Security Context

Load context based on what is being reviewed:

- **Always**: Apply OWASP Top 10 (2021) patterns from
  `./reference/security-review.context`

- **For web applications or services**: Additionally apply OWASP WSTG v4.2
  patterns from `./reference/wstg-testing.context`

- **For Go code** (`.go` files or `go.mod` present): Additionally apply Go
  security patterns from `./reference/golang-security.context`

- **For native/systems code** (C, C++, Rust, or build files such as
  `Makefile`, `CMakeLists.txt`, `Cargo.toml`): Additionally apply CWE/SANS
  Top 25 patterns

- **For container images or Kubernetes workloads** (Dockerfiles,
  Containerfiles, YAML manifests with `apiVersion`/`kind`, docker-compose):
  Invoke the `secdevai-oci-image-security` skill separately
  (`../secdevai-oci-image-security/SKILL.md`) to load OCI-specific patterns

### Step 4: Perform Analysis

- Scan code for security patterns from the loaded contexts
- Classify findings by severity: Critical / High / Medium / Low / Info
- Reference OWASP categories and CWE IDs where applicable
- Adapt examples and remediation to the language and framework being reviewed
- Verify all reported line numbers against the actual file content before
  reporting

### Step 5: Validate Findings

Before presenting results, for each finding:

1. Read the actual source code at the reported location
2. Confirm the vulnerable code pattern is present at that line
3. Check whether a mitigation is already in place nearby (e.g., validation
   upstream of the flagged call)
4. Assess whether a realistic attack path exists (exploitability)
5. Remove findings that are not exploitable or are false positives

### Step 6: Present Findings

Group findings by severity. For each finding include:

- File and line number
- Severity (Critical / High / Medium / Low / Info)
- OWASP category and/or CWE ID
- Description of the vulnerable pattern
- Vulnerable code snippet
- Risk: what an attacker could achieve
- Remediation: concrete code change or configuration fix
- References: OWASP, CWE, or language documentation links

At the end, summarize: total findings, severity distribution, and files
reviewed vs. skipped.

## Multi-Language Support

While context files use primarily Python examples, apply the same patterns to
the language being reviewed. Translate remediation to language-appropriate
idioms:

**SQL injection:**
- Python: parameterized queries with `cursor.execute("... %s", (val,))`
- Go: `db.Query("... ?", val)`
- Java: `PreparedStatement` with `setString(1, val)`
- Node.js: `db.query("... ?", [val])`

**Cryptographic randomness:**
- Python: `secrets.token_urlsafe(32)` instead of `random`
- Go: `crypto/rand` instead of `math/rand`
- Java: `SecureRandom` instead of `Random`
- Node.js: `crypto.randomBytes(32)` instead of `Math.random()`

**TLS verification:**
- Never disable certificate verification
- Always use the platform's trusted CA store unless explicitly overriding with
  a custom CA

## Verification Requirement

Before presenting any finding, confirm the line numbers and code snippets
match the actual file content. Do not report findings based solely on diff
context — read the live file.

## Reference Files

The following context files in `reference/` contain detailed patterns and
examples. Load them as described in Step 3:

- `security-review.context` — OWASP Top 10 patterns (always loaded)
- `wstg-testing.context` — OWASP WSTG v4.2 web testing patterns
- `golang-security.context` — Go-specific security patterns

## Upstream References

- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [OWASP Web Security Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/)
- [CWE/SANS Top 25](https://cwe.mitre.org/top25/)
