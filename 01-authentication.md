# 1. Authentication

Authentication answers exactly one question: **who is making this request?** Nothing more. It
does not answer "should they be allowed to do this" (that's [authorization](02-authorization.md)).
Conflating the two is one of the most common design mistakes in software.

Every authentication mechanism is really just a way of proving possession of, or knowledge of,
some secret. The differences between them are: *what* the secret is, *where* it's checked, and
*what happens if it leaks*.

## Recommendation — what to actually build, by scenario

Don't read this as "pick your favorite." Different scenarios have different right answers, and
getting this wrong in either direction (over-engineering a low-stakes internal tool, or
under-engineering a consumer app holding real user data) is a real cost either way.

```mermaid
flowchart TD
    A[Who is authenticating?] --> B{Human end user?}
    B -->|yes| C{Can you require a modern browser/device?}
    C -->|yes| D["**WebAuthn / Passkeys** \n(phishing-resistant, no shared secret)"]
    C -->|no / must support legacy| E["Password + TOTP MFA \n(never SMS-only)"]
    B -->|no, another service| F{Same trust domain / internal?}
    F -->|yes, zero-trust internal| G["**mTLS** between services"]
    F -->|no, third-party delegated access| H["**OAuth2 Authorization Code + PKCE**, \nOIDC for identity"]
    F -->|yes, simple internal automation| I["Short-lived signed service token \n(OAuth2 client-credentials)"]
```

**The single best answer for human login today is WebAuthn/passkeys** ([§1.9](#19-passwordless--webauthn--passkeys)) —
they are the only mechanism in this file that is structurally phishing-resistant rather than just
"harder to phish." Where passkeys aren't yet feasible (legacy user base, no budget for the
migration this quarter), the recommended fallback is **password + TOTP-based MFA**
([§1.10](#110-multi-factor-authentication-mfa)) — never SMS-only, never password alone.

For **session transport**, prefer server-side sessions
([§1.2](#12-session-based-authentication-server-side-sessions)) or short-lived tokens with rotating
refresh tokens over long-lived stateless JWTs, specifically because of the revocation gap described
below — see [06](06-session-and-token-security.md) for the full reasoning and the recommended
storage location (`HttpOnly` cookie, not `localStorage`).

For **delegated third-party access**, OAuth2 Authorization Code + PKCE is the only flow that should
still be used for anything with a browser involved — the Implicit flow is formally deprecated
(tokens exposed in the URL fragment, no client authentication) and should be treated as a finding
if found in a codebase today.

For **service-to-service inside your own infrastructure**, mTLS is the strongest option where you
can afford the certificate-lifecycle operational cost ([§1.8](#18-mutual-tls-mtls),
[03 §3.5](03-transport-security-tls.md)); short-lived signed tokens (client-credentials grant) are
the pragmatic middle ground when full mTLS isn't yet in place.

## 1.1 Password-based authentication

The user proves they know a secret string. The server never stores the password itself (see
[07 - Password Storage](07-password-storage.md)) — it stores a hash and compares hashes.

**What breaks it:**
- **Credential stuffing** — attackers replay breached username/password pairs from other sites,
  because most people reuse passwords. Defense: rate limiting, breach-password checks (e.g.
  against Have I Been Pwned's k-anonymity API), MFA.
- **Brute force** — defense: lockouts, exponential backoff, CAPTCHAs after N failures.
- **Phishing** — the password itself is time-invariant and can be typed into a fake site. This is
  the fundamental weakness passwords have that possession-based factors (below) don't.

## 1.2 Session-based authentication (server-side sessions)

After login, the server creates a session record (in memory, Redis, or a DB) keyed by a random
**session ID**, and gives the browser that ID in a cookie. Every request, the server looks up the
session ID to find out who's logged in.

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Server
    participant DB as Session Store

    B->>S: POST /login (credentials)
    S->>S: verify password (hash compare)
    S->>DB: create session record (random ID)
    DB-->>S: sid = abc123...
    S-->>B: Set-Cookie sid=abc123 (HttpOnly, Secure, SameSite=Lax)
    Note over B,S: later request, same session
    B->>S: GET /api/data (Cookie: sid=abc123)
    S->>DB: look up session "abc123"
    DB-->>S: user = alice, roles = [...]
    S-->>B: 200 OK
```

**Key properties:**
- The session ID itself must be cryptographically random and long (128+ bits of entropy) —
  otherwise it can be guessed.
- Sessions are **revocable server-side** instantly — delete the session record, the user is
  logged out everywhere immediately. This is the single biggest advantage session auth has over
  stateless tokens.
- The cookie must be `HttpOnly` (JavaScript can't read it, so XSS can't steal it directly),
  `Secure` (only sent over HTTPS), and `SameSite=Lax` or `Strict` (mitigates CSRF — see
  [05](05-secure-coding-owasp.md)).

**Downside:** requires server-side state, which complicates horizontal scaling (needs a shared
session store like Redis) and doesn't fit stateless/serverless architectures as cleanly.

## 1.3 Token-based authentication (bearer tokens, JWT)

Instead of a session ID that means nothing on its own, the server issues a **token** that itself
encodes the claims (who the user is, their roles, an expiry). The most common format is a **JWT**
(JSON Web Token): three base64url segments — `header.payload.signature`.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6ImFkbWluIn0.4Adcj3UFYzPUVaVF43FmMab6RlaQD8A9V8wFzzht-KQ
   header (alg)              payload (claims)                      signature
```

The server verifies the **signature** (HMAC with a shared secret, or RSA/ECDSA with a public key)
to confirm the payload hasn't been tampered with, then trusts the claims inside without a DB
lookup.

**This is the critical tradeoff to understand: a JWT is "stateless" — the server does not need to
look anything up to validate it, which is fast and scales horizontally for free. But that also
means you cannot revoke a single JWT before it expires.** If a token leaks, it's valid until its
`exp` claim, full stop, unless you build a denylist (which reintroduces the state you were trying
to avoid).

**Real-world JWT vulnerabilities (all have caused real breaches):**
- **`alg: none` attack** — some libraries historically accepted a token with `"alg":"none"` and no
  signature at all, letting an attacker forge any payload. Fix: always pin the expected algorithm
  server-side; never let the token itself dictate which algorithm to verify with.
- **Algorithm confusion (RS256 → HS256)** — if a server uses RSA (`RS256`) and its public key is,
  well, public, an attacker can craft a token signed with `HS256` **using the public key as the
  HMAC secret**. A naive verifier that just checks "does the signature match, using whatever
  algorithm the header claims" will accept this forged token. Fix: hardcode the algorithm you
  expect per key type; reject anything else.
- **No expiry, or absurdly long expiry** — a stolen token that never expires is a stolen identity
  forever. Keep access tokens short-lived (minutes), use refresh tokens for longevity (see
  [06](06-session-and-token-security.md)).
- **Storing sensitive data in the payload** — the payload is **base64-encoded, not encrypted**.
  Anyone with the token can read every claim in it. Never put a password, SSN, or secret in a JWT
  payload.

**Where to put a JWT on the wire:** the `Authorization: Bearer <token>` header — never a URL query
string (URLs get logged everywhere: proxies, browser history, server access logs).

## 1.4 OAuth 2.0 — delegated authorization, not authentication

This is the single most misunderstood spec in the industry. **OAuth2 is not an authentication
protocol.** It's a way for a user to grant a third-party application limited access to their data
on another service, without handing over their password.

The core flow (Authorization Code + PKCE — **the only flow you should use today**, including for
plain server-rendered web apps; the old Implicit flow is deprecated and should be flagged if found):

```mermaid
sequenceDiagram
    participant U as User (browser)
    participant App as YourApp
    participant IdP as Google (Identity Provider)

    App->>App: generate code_verifier (random), code_challenge = SHA256(code_verifier)
    U->>App: clicks "Log in with Google"
    App->>U: redirect to Google, with code_challenge
    U->>IdP: logs in (Google authenticates the user — YourApp never sees the password)
    IdP->>U: consent screen ("allow YourApp to read your email?")
    IdP-->>App: redirect back with short-lived authorization `code`
    App->>IdP: exchange code + client_secret + code_verifier for tokens
    IdP->>IdP: verify SHA256(code_verifier) == code_challenge from step 1
    IdP-->>App: access_token (API scope) + id_token (OIDC identity, if requested)
    App->>IdP: call Google API with access_token, on the user's behalf
```

Notice: the OAuth2 part alone (access token) never tells YourApp *who the user is* — it only
proves YourApp has permission to call an API. That gap is exactly what the `id_token` (OIDC, next
section) exists to close.

**Why PKCE matters, mechanically:** without it, whoever intercepts the redirect in step 5 (a
malicious app registered for the same custom URL scheme on mobile, or a logged network proxy) can
take the authorization `code` and complete the exchange themselves. With PKCE, completing the
exchange also requires the original `code_verifier` — which never left YourApp's memory, was never
sent in a redirect, and can't be derived from the `code_challenge` that *was* sent (that's what
makes SHA-256 useful here: easy to check a match, computationally infeasible to reverse). PKCE was
originally built for public clients that can't hold a secret (mobile apps, SPAs) but is now
recommended for confidential clients too, as defense-in-depth.

## 1.5 OpenID Connect (OIDC) — authentication built on top of OAuth2

OIDC adds exactly what OAuth2 is missing: a standardized **ID token** (a JWT) that asserts identity
— `sub` (subject/user ID), `email`, `name`, `iss` (issuer), `aud` (audience), signed by the
identity provider. This is the actual "who is this person" answer. The access token from OAuth2 is
still separate and is for calling APIs, not for proving identity to your own backend.

**Common mistake:** using the OAuth2 *access token* as if it were proof of identity. Access tokens
are opaque-by-design in many providers and aren't guaranteed to contain or even relate to user
identity claims — always use the ID token for "who is this," the access token for "what can this
call do."

## 1.6 SAML

The enterprise predecessor to OIDC, still extremely common in corporate SSO (Okta, ADFS, Azure AD
classic integrations). XML-based assertions instead of JWTs, browser-redirect-based like OAuth2's
flow. Functionally does the same job as OIDC (federated identity) but with a much larger attack
surface historically due to XML parsing complexity — **XML Signature Wrapping** attacks (where an
attacker adds a forged assertion alongside a legitimate signed one, and a poorly-written verifier
checks the signature against the wrong node) have caused real breaches in SAML implementations.
Lesson: never hand-roll SAML or XML-DSig validation; use a maintained, audited library.

## 1.7 API keys

A long-lived, static secret string sent with every request (header, rarely a query param). Simple,
but has none of the properties of a good credential:
- No built-in expiry (unless you build it).
- No scoping unless you build it (one key = full access, typically).
- Compromise is silent and permanent until manually rotated.
Appropriate for server-to-server integrations where the "user" is a whole system, not a person —
never for end-user authentication. Always allow rotation without downtime (issue key #2, migrate,
revoke key #1) and log key usage so you can detect anomalous use.

## 1.8 Mutual TLS (mTLS)

Both client and server present X.509 certificates during the TLS handshake, so the server
authenticates the client cryptographically at the transport layer, before any application code
runs. Extremely strong (private key never leaves the client, no password to phish) but
operationally heavy — certificate issuance, rotation, and revocation (CRL/OCSP) for every client is
real infrastructure work. Common in service-to-service (zero-trust internal networks — see
[08](08-api-and-service-security.md)) and high-assurance B2B integrations, rare for consumer apps.

## 1.9 Passwordless / WebAuthn / Passkeys

The modern answer to phishing: **public-key cryptography instead of a shared secret.** On
registration, the user's device (a hardware key, phone, or platform authenticator like Touch ID)
generates a public/private keypair. The private key **never leaves the device** — often it's
sealed inside a secure enclave and can't be extracted even by the device owner. The server stores
only the public key.

On login, the server sends a random challenge; the device signs it with the private key; the
server verifies the signature with the stored public key.

```mermaid
sequenceDiagram
    participant D as User's Device (authenticator)
    participant Br as Browser
    participant S as Server

    Note over D,S: Registration (once)
    Br->>S: request registration challenge
    S-->>Br: random challenge + relying party ID (your domain)
    Br->>D: WebAuthn API call (browser binds this to the CURRENT origin)
    D->>D: generate keypair, private key stays in secure enclave
    D-->>Br: public key + signed attestation
    Br->>S: send public key
    S->>S: store public key against this user, discard nothing secret (there wasn't one)

    Note over D,S: Login (every time)
    Br->>S: request login challenge
    S-->>Br: random challenge
    Br->>D: sign challenge (browser again binds this to the CURRENT origin)
    D-->>Br: signature
    Br->>S: send signature
    S->>S: verify signature against stored public key
    S-->>Br: 200 OK, session established
```

**Why this actually kills phishing (not just makes it harder):** the browser's WebAuthn API
cryptographically binds the signed assertion to the origin (domain) that requested it. If you're
tricked into visiting `evil-bank.com` that looks like `bank.com`, your authenticator will not
produce a valid signature for `bank.com`'s challenge, because the origin doesn't match — the
browser itself enforces this, not the user's judgment. This is categorically different from a
password or even an OTP code, both of which a user *can* be tricked into typing into the wrong
site.

## 1.10 Multi-Factor Authentication (MFA)

Combine factors from different categories:
- **Something you know** — password, PIN.
- **Something you have** — phone (SMS/TOTP app), hardware key.
- **Something you are** — biometrics.

**SMS OTP is the weakest "second factor"** — vulnerable to SIM-swapping (a social-engineering
attack against the phone carrier, not the app) and SS7 network interception. Prefer TOTP (RFC
6238, e.g. Google Authenticator) or, better, WebAuthn hardware keys, which are phishing-resistant
in the way passwords and OTP codes are not (see 1.9).

**"MFA fatigue" attacks:** attackers who already have the password spam push-notification MFA
prompts until an annoyed user taps "approve." Defense: number-matching (the user must type a
number shown on the login screen into the push notification, so a blind tap can't approve it) and
rate-limiting the number of prompts.

## Summary table

| Mechanism | Proves | Revocable instantly? | Phishing-resistant? | Recommended for |
|---|---|---|---|---|
| Password | knowledge of a secret | yes (change password) | no | never alone — pair with MFA |
| Session cookie | possession of session ID | yes (delete session) | no (steals like a password) | traditional server-rendered apps |
| JWT / bearer token | possession of a signed claim set | no, without a denylist | no | stateless APIs, short-lived only |
| OAuth2 access token | delegated API scope | depends on provider | no | calling a third-party API on a user's behalf |
| OIDC ID token | identity, asserted by IdP | n/a (short-lived, re-issued) | no | federated login ("Sign in with X") |
| API key | possession of a static secret | only via manual rotation | no | server-to-server, non-human callers only |
| mTLS client cert | possession of private key | yes (CRL/OCSP) | yes | **service-to-service, zero-trust internal** |
| WebAuthn/passkey | possession of private key | yes (remove credential) | **yes** | **human login — the best available option today** |
