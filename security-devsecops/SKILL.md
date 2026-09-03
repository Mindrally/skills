---
name: security-devsecops
description: "DevSecOps, secure software development lifecycle (SSDLC), and application security (AppSec) practices covering secret handling, input validation, dependency hygiene, authentication/authorization, and CI/CD security tooling (SAST, SCA, DAST, secret scanning, IaC scanning). Use when writing code that handles credentials, user input, database queries, or authentication, when setting up a CI/CD pipeline, or when reviewing code or infrastructure for security issues."
---

# Security / DevSecOps

This skill covers secure coding practices, dependency and secret hygiene, authentication and authorization, and the security tooling that belongs in a secure software development lifecycle (SSDLC) — from local coding habits through CI/CD gates to production monitoring.

## Workflow for Adding a New Feature Securely

1. **Identify trust boundaries** — Note every place the new code accepts input from a user, another service, or a file, and every place it emits output (HTML, logs, another service).
2. **Validate and sanitize at the boundary** — Validate all untrusted input on entry; escape output for its destination context (HTML, JS, SQL, shell).
3. **Use existing security primitives** — Reach for the project's established auth framework, ORM, and secret-management approach rather than writing new ones.
4. **Keep secrets out of code and logs** — Read credentials from environment variables or a secrets vault; confirm nothing sensitive reaches logs or error messages.
5. **Run local security checks before pushing** — Lint, SAST, and secret-scanning tools where available (`gitleaks`, `semgrep`, `npm audit`, `pip-audit`, etc.).
6. **Let CI gates run** — SAST, SCA, secret scanning, and IaC scanning should run on every PR; treat a failure as a blocker, not a suggestion to suppress.
7. **Document the security-relevant decision** — Note in the PR description any auth/authz change, new dependency, or deviation from a default-secure pattern, so it's auditable later.

## General Security Principles

- Never hardcode secrets, credentials, or API keys in source code. Use environment variables or a secure vault (e.g. AWS Secrets Manager, HashiCorp Vault, Doppler) for sensitive data.
- Never commit `.env` files, secret config files, or unrecognized tokens to source control. Add them to `.gitignore` before they're ever staged.
- Never log sensitive data, secrets, or session tokens in application logs — redact or omit them at the point of logging, not after the fact.
- Validate and sanitize all user input at the point it enters the system. Escape output appropriately for its context: HTML-encode for HTML, JS-encode for inline scripts, parameterize for SQL.
- Avoid unsafe dynamic-execution functions such as `exec`, `eval`, `Function()`, `pickle.loads` on untrusted data, or shell interpolation of user input.

```python
# Unsafe — string interpolation into a query
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)

# Safe — parameterized query
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

## Secret Handling

- Load secrets from environment variables (`process.env.API_KEY`, `os.environ["API_KEY"]`) or a vault client, never as literals.
- Rotate any credential that was ever committed to version control, even if the commit was later removed — history retains it.
- Scan for accidentally committed secrets before merge, using tools like `gitleaks`, `trufflehog`, or the CI provider's built-in secret scanning.
- Scope credentials narrowly: a CI deploy key should be able to deploy, not administer the whole cloud account.

## Database Security

- Use parameterized queries or an ORM for all database access. Never build queries via string concatenation or f-strings with user input.
- Ensure database users have the least privilege required for their role (a reporting service should not have `DROP TABLE` rights).
- Regularly review and update database access policies as team membership and service responsibilities change.

## Dependency Management

- Only add packages from verified, reputable sources — check download counts, maintenance activity, and known-CVE history before adding a new dependency.
- Do not add new dependencies without explicit approval and a brief security review, especially for packages that will run with elevated privileges or process untrusted input.
- Regularly update dependencies and scan for known vulnerabilities using Software Composition Analysis (SCA) tools such as `npm audit`, `pip-audit`, `Dependabot`, `Snyk`, or `Trivy`.

```bash
# Node.js
npm audit --audit-level=high

# Python
pip-audit

# Container image
trivy image myapp:latest
```

## Authentication & Authorization

- Use established, audited authentication frameworks (e.g. Auth0, Passport, Devise, ASP.NET Identity) — never implement custom cryptographic authentication from scratch.
- Store passwords using strong, salted, adaptive hashes: Argon2id or bcrypt, never MD5, SHA1, or unsalted SHA256.
- Implement Role-Based Access Control (RBAC) — or attribute-based access control for finer granularity — for sensitive operations.
- Enforce the principle of least privilege for both API endpoints and UI actions: check authorization on every request server-side, never trust a hidden UI element as an access control.
- Re-check authorization on every request, even for actions previously permitted — session state and roles can change mid-session.

```js
// Missing authorization check — any authenticated user can access any order
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  res.json(order);
});

// Correct — verify the resource belongs to the requester
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order || order.userId !== req.user.id) {
    return res.status(404).end();
  }
  res.json(order);
});
```

## Secure SDLC Practices

Integrate these into the CI/CD pipeline, not as a manual pre-release checklist:

- **SAST** (Static Application Security Testing) — scan source code for known-bad patterns on every PR (e.g. Semgrep, CodeQL, SonarQube).
- **SCA** (Software Composition Analysis) — scan dependencies for known vulnerabilities and license issues (e.g. Snyk, Dependabot, OWASP Dependency-Check).
- **Secret scanning** — scan every commit and PR diff for accidentally committed credentials before merge (e.g. Gitleaks, GitHub secret scanning, TruffleHog).
- **IaC scanning** — scan Terraform/CloudFormation/Kubernetes manifests for misconfigurations before apply (e.g. Checkov, tfsec, Trivy config scan).
- **DAST** (Dynamic Application Security Testing) — scan a running instance of the deployed application for exploitable behavior in the CD pipeline (e.g. OWASP ZAP, Burp Suite).
- **Policy as Code (PaC)** — encode security and compliance policies as version-controlled, automatically enforced rules (e.g. OPA/Rego, Sentinel) rather than a document nobody rereads.

## Monitoring & Feedback

- Enable continuous vulnerability monitoring and alerting on both dependencies and running infrastructure.
- Integrate Runtime Application Self-Protection (RASP) and a Web Application Firewall (WAF) where the deployment target and traffic profile warrant it.
- Schedule regular vulnerability assessments and penetration tests, not just point-in-time audits before a big release.
- Maintain a feedback loop: recurring vulnerability classes found in production or pentests should update linting rules, SAST policies, and code review checklists — not just get patched once.

## Compliance & Documentation

- Align controls with recognized industry standards: OWASP Top 10, OWASP ASVS, NIST SSDF, ISO 27001, as relevant to the project's regulatory context.
- Document security controls and decisions as they're made (threat model notes, why an exception was granted, what compensating control offsets a known risk) so the reasoning is auditable later, not reconstructed from memory during an audit.

## Common Vulnerability Classes to Watch For

| Class | Watch for |
|---|---|
| Injection (SQL/NoSQL/command) | String-built queries or shell commands with user input |
| Broken access control | Missing per-request authorization checks, IDOR via predictable IDs |
| Cryptographic failures | Weak hashes, hardcoded keys, missing TLS, custom crypto |
| Insecure deserialization | `pickle`, unchecked `JSON.parse` into executable contexts, unsafe YAML loaders |
| Security misconfiguration | Default credentials, verbose error pages in production, permissive CORS |
| Vulnerable dependencies | Outdated packages with known CVEs, unpinned versions |
| Insufficient logging | No audit trail for auth events, sensitive data logged in plaintext |
