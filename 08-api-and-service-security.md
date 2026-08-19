# 8. API & Service-to-Service Security

## 8.1 "It's on the internal network" is not a security control

The single most common assumption that gets punished in real breaches: treating anything inside
the corporate/cloud VPC as inherently trusted, with authentication and authorization only enforced
at the public-facing edge. Once an attacker gets *any* foothold inside the perimeter — a
compromised laptop, one vulnerable internal service, a misconfigured VPN — everything behind that
soft internal boundary is wide open, because nothing inside was built to defend against a hostile
peer.

**Zero trust** is the architectural response: authenticate and authorize *every* request, including
service-to-service calls on the "internal" network, exactly as rigorously as you would a
public-facing request. In practice this looks like: mTLS between every service (see
[03 §3.5](03-transport-security-tls.md)), each service call still carries and validates an identity
token, and network segmentation is treated as one layer of defense, never the only one.

## 8.2 Rate limiting and abuse prevention

Without limits, any authenticated (or even unauthenticated) client can hammer an endpoint —
brute-forcing a login, scraping an entire dataset one paginated request at a time, or simply
degrading service for everyone else. Layer this at multiple levels:
- **Per-IP / per-API-key rate limits** at the edge (API gateway, reverse proxy, CDN) — cheap, stops
  the crudest abuse before it reaches application code.
- **Per-account limits** on sensitive actions specifically (login attempts, password reset
  requests, OTP verification) — an attacker with many IPs (a botnet) bypasses IP-based limiting
  entirely, so the sensitive-action limit must key on the account being targeted, not just the
  source.
- **Exponential backoff** on repeated failures, not just a hard cap — makes automated retry loops
  progressively more expensive for an attacker without permanently locking out a legitimate user
  who mistyped their password twice.

## 8.3 Service identity — a service account is not "no user," it's a different user

Every service-to-service call needs its own answer to "who is making this request," distinct from
the human user it might be acting on behalf of. Two common, legitimately different patterns, easy
to conflate:
- **Client-credentials flow (OAuth2)** — the service authenticates *as itself* (its own client
  ID/secret or certificate), representing "this backend job," with its own scoped permissions —
  appropriate for scheduled jobs, internal automation, anything not acting for a specific end user.
- **Token propagation / on-behalf-of** — Service A forwards or exchanges the *original end user's*
  token when calling Service B, so Service B's authorization checks still apply to that specific
  user, not to Service A's own (likely much broader) service identity. This matters enormously:
  if Service A instead calls Service B using its own powerful service credentials regardless of
  which user initiated the request, then any authorization bug or bypass in Service A effectively
  grants that user Service A's full privilege on Service B — a textbook confused deputy
  ([02 §2.4](02-authorization.md)).

## 8.4 Input validation at every trust boundary, not just the public edge

A service that only validates input at its outermost public API, and trusts anything arriving from
"another one of our own services," breaks the moment any upstream service has a bug, is
compromised, or is simply wrong about what it's forwarding. Validate at **every** boundary a
request crosses, not just the first one — defense in depth means a single compromised or buggy
component doesn't cascade into every downstream system trusting its output blindly.

## 8.5 API versioning and deprecation as a security practice, not just a compatibility one

An old, unmaintained API version left reachable "for backward compatibility" is frequently the
version that never received a later security fix (a stricter validation rule, a newly-required
auth check) that the current version has. Track and actively sunset old versions rather than
letting them accumulate indefinitely as a widening, unmonitored attack surface.

## 8.6 Least-exposure API design

- Don't return more data than the caller needs "because it's convenient" (a `/users/:id` endpoint
  returning the full user record including a password hash column, an internal-only flag, another
  user's data nested in a relation) — every extra field returned is something a client-side bug,
  a browser dev-tools inspection, or a network log can leak, regardless of whether the UI happens
  to only display a subset of it today.
- Don't expose internal IDs/implementation details unnecessarily (sequential integer IDs make
  enumeration trivial — `/orders/1001`, `/orders/1002`... — prefer UUIDs or opaque IDs where
  enumeration itself is a risk, combined with proper authorization checks regardless).
