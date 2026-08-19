# Applied Cybersecurity for Software Engineers

A working curriculum, not a survey course. Each file teaches the *mechanism* (why an attack
works, why a defense actually stops it), not just a checklist of buzzwords. Sequence flows,
handshakes, and data models are drawn as Mermaid diagrams — they render natively on GitHub and in
most Markdown viewers/editors (VS Code needs the "Markdown Preview Mermaid Support" extension).
Read in order the first time; use as reference after.

## Contents

1. [Authentication](01-authentication.md) — passwords, sessions, tokens (JWT), OAuth2/OIDC, SAML,
   API keys, mTLS, WebAuthn/passkeys, MFA. What each one actually proves, what it doesn't, and an
   explicit **recommendation** (with a decision diagram) for which to use in which scenario —
   short version: **WebAuthn/passkeys for human login, mTLS for internal service-to-service.**
2. [Authorization](02-authorization.md) — RBAC, ABAC/PBAC, policy engines, the confused-deputy
   problem, why "authenticated" and "authorized" are different questions.
3. [TLS/SSL In Depth](03-transport-security-tls.md) — the handshake, why SSL is dead, cipher
   suites, certificate chains, mTLS, HSTS, and the misconfigurations that actually get exploited.
4. [Secrets & Key Management](04-secrets-and-key-management.md) — why `.env` files leak, KMS/HSM,
   rotation, envelope encryption.
5. [Secure Coding & the OWASP Top 10](05-secure-coding-owasp.md) — injection, XSS, CSRF, SSRF,
   deserialization, path traversal — with the actual payloads and the actual fixes.
6. [Session & Token Security](06-session-and-token-security.md) — session fixation, token storage
   (why `localStorage` is a mistake), refresh-token rotation, logout that actually logs out.
7. [Password & Credential Storage](07-password-storage.md) — bcrypt/scrypt/Argon2 explained from
   first principles, salting vs. peppering, why `MD5(password)` is a vulnerability.
8. [API & Service-to-Service Security](08-api-and-service-security.md) — rate limiting, zero
   trust, service identity, the "internal network is safe" fallacy.
9. [REST Endpoint Security](09-rest-endpoint-security.md) — security headers, CORS
   misconfiguration, mass assignment, HTTP verb tampering, pagination/DoS limits, error handling
   that doesn't leak, exposed API docs — the concrete, per-endpoint checklist.
10. [Infrastructure & Supply Chain](10-infrastructure-and-supply-chain.md) — dependency
    vulnerabilities, SBOM, container security, CI/CD secret leakage, SSRF from your own pipeline.
11. [Applied: This Codebase](11-applied-to-this-codebase.md) — every concept above, pointed at the
    real iNXG-backend auth implementation we already reverse-engineered together, so you can see
    the theory land on actual code instead of a toy example.
12. [Shipping Checklist](12-shipping-checklist.md) — a checklist you actually run before merging
    anything that touches auth, user input, or a network boundary.

## How to use this

- If you only read one file, read **03 (TLS)** and **05 (OWASP)** — they cover what actually gets
  exploited in practice.
- If you need one concrete recommendation right now: **for human authentication, use
  WebAuthn/passkeys** ([01](01-authentication.md#recommendation--what-to-actually-build-by-scenario)) —
  it's the only mechanism covered here that's structurally phishing-resistant rather than just
  harder to phish.
- File 11 is where this stops being abstract. Read it after at least 01, 02, and 03.
- Nothing here is exhaustive-for-its-own-sake. Every section exists because it maps to a real,
  common way software gets broken.
