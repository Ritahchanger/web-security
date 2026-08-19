# 12. Shipping Checklist

Run this before merging anything that touches authentication, authorization, user input, or a
network boundary. Not every item applies to every change — use judgment, but don't skip the
judgment step itself.

## Authentication
- [ ] Is this actually an authentication change, or does it just look like one? (§ confusing auth
      with authz is the #1 design mistake — see [02](02-authorization.md))
- [ ] If issuing a token: does it have a short, sane expiry? Is `aud`/`iss` validated on every
      consumer, not just signature validity?
- [ ] If storing a token client-side: cookie (`HttpOnly`, `Secure`, `SameSite`) preferred over
      `localStorage` unless there's a specific reason otherwise — see
      [06 §6.2](06-session-and-token-security.md).
- [ ] Does logout actually invalidate server-side state, not just clear the client?

## Authorization
- [ ] Is every new endpoint/action explicitly `[Authorize]`-annotated (or explicitly, deliberately
      `[AllowAnonymous]`)? No endpoint should silently inherit a default.
- [ ] For any endpoint accepting an object ID: is there a check that the *authenticated user*
      actually has permission over *that specific object*, not just that the object exists?
      (IDOR — [05 §5.9](05-secure-coding-owasp.md))
- [ ] Any cross-module/cross-service call: does it propagate the real caller's identity, or does
      it use a broad service identity that could become a confused deputy?
      ([02 §2.4](02-authorization.md), [08 §8.3](08-api-and-service-security.md))

## Input handling
- [ ] Any raw SQL, shell command, or LDAP query construction? If yes: parameterized, not
      concatenated. ([05 §5.1](05-secure-coding-owasp.md))
- [ ] Any user data rendered into HTML/JS/attributes? Confirm the framework's default encoding is
      actually in effect — flag any `dangerouslySetInnerHTML`/`v-html`/raw HTML insertion for
      explicit sanitization review.
- [ ] Any feature that fetches a URL supplied by user input (webhooks, "import from URL," link
      previews)? Check for SSRF — allowlist destinations, block private/link-local IP ranges.
      ([05 §5.4](05-secure-coding-owasp.md))
- [ ] Any file path built from user input? Verify it can't escape the intended directory.

## Secrets
- [ ] No new hardcoded credentials, keys, or tokens in source, including test fixtures and
      example configs.
- [ ] Any new secret has a documented rotation path, not just an initial-issuance path.
- [ ] CI logs checked for accidental secret exposure if this change touches build/deploy scripts.

## Transport
- [ ] No new code that disables TLS/certificate validation, for any environment that isn't purely
      local dev. ([03 §3.4](03-transport-security-tls.md))
- [ ] Any new service-to-service call: does it run over TLS (or mTLS, for internal zero-trust
      architectures), even if it "looks internal"?

## Passwords / credentials at rest
- [ ] Any new place storing a password or credential? Confirm it's a proper slow hash
      (bcrypt/Argon2), never a fast general-purpose hash or reversible encryption.
      ([07](07-password-storage.md))

## Dependencies
- [ ] Any new dependency added — is it actively maintained, from a reputable source, pinned to an
      exact version?
- [ ] Does CI's dependency scan pass, or is a known vulnerability being knowingly, temporarily
      accepted with a tracked follow-up?

## General
- [ ] Would this change survive a code reviewer deliberately trying to break it, rather than just
      checking it does what the ticket asked?
- [ ] If this touches an area with an existing architecture/security test (a boundary test, an
      endpoint-policy test — see [11 §11.4](11-applied-to-this-codebase.md) for a real example),
      does the new code actually get covered by it, or does it slip through the same kind of gap
      that test exists to catch?

---
**Next:** you've reached the end of the curriculum — back to [the index](README.md), or revisit
[11. Applied: This Codebase](11-applied-to-this-codebase.md) to see it all land on real code.
