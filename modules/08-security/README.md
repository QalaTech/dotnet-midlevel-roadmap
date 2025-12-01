# 08 — Security, Authentication & Identity

## Why This Module Exists

Security breaches destroy companies. Not "might" — they do:
- Customer data leaked
- Regulatory fines (GDPR: up to 4% of global revenue)
- Reputation destroyed
- Years of work undone

Most security failures aren't sophisticated attacks. They're:
- Exposed secrets in git history
- SQL injection in search fields
- Missing authorization checks
- Weak password requirements

This module teaches you to build secure systems from the start, not bolt security on later.

---

## What You'll Build

OrderFlow's security infrastructure:
- **Threat Model** — Identify and mitigate risks
- **Authentication** — JWT, OAuth2/OIDC integration
- **Authorization** — Policies, roles, resource ownership
- **API Protection** — Rate limiting, secrets, monitoring

---

## Module Structure

### [Part 1 — Security Foundations](./part-1.md)
- OWASP Top 10 for .NET developers
- Threat modeling with STRIDE
- Secure coding practices
- Static analysis and secure defaults

### [Part 2 — Authentication](./part-2.md)
- JWT bearer authentication
- OAuth2/OIDC integration
- Token validation and refresh
- Session management

### [Part 3 — Authorization](./part-3.md)
- Policy-based authorization
- RBAC and ABAC patterns
- Resource-based authorization
- Multi-tenant security

### [Part 4 — API Security Operations](./part-4.md)
- Rate limiting strategies
- Secrets management
- Security monitoring
- Incident response

---

## Prerequisites

Before starting this module:
- Complete Module 03 (Building APIs) — understand middleware
- Complete Module 07 (Testing) — test security features
- Have an identity provider account (Azure AD, Auth0, or Keycloak)

---

## Key Concepts You'll Master

1. **Defense in Depth** — Multiple layers of security
2. **Principle of Least Privilege** — Minimum necessary access
3. **Secure by Default** — Safe configuration out of the box
4. **Zero Trust** — Verify everything, trust nothing

---

## Tools You'll Use

- **Azure AD / Auth0 / Keycloak** — Identity providers
- **ASP.NET Core Authentication** — JWT/Cookie handling
- **ASP.NET Core Authorization** — Policy engine
- **Azure Key Vault** — Secrets management
- **OWASP ZAP** — Security scanning

---

## Exit Criteria

You can:
1. Perform threat modeling for new features
2. Implement JWT authentication with proper validation
3. Build authorization policies for complex scenarios
4. Configure rate limiting and secrets management
5. Detect and respond to security incidents

---

**Start:** [Part 1 — Security Foundations](./part-1.md)
