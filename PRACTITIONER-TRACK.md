# The Practitioner Track

A 12-week route from cybersecurity theory to applied software security practice — built for
someone who already has the frameworks (a degree, a bootcamp, self-study) and now needs the
hands-on reps: reading real vulnerable code, breaking it properly, and writing the kind of
finding a hiring manager recognizes as professional work.

This isn't a repeat of the fundamentals — it skips the CIA triad, the compliance frameworks, the
networking basics. What's missing after most cybersecurity education is the specific muscle of
**application security work**: reading a real code path and knowing exactly where it breaks,
running the tools professionals actually reach for, and writing a finding someone can act on
without three follow-up questions.

Every week pairs a short read from this repo with a real, free hands-on lab — mostly
[PortSwigger's Web Security Academy](https://portswigger.net/web-security) (the industry-standard
free training ground) and [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) (a
deliberately vulnerable app built to be broken). It ends in one real deliverable: a full
vulnerability assessment report, portfolio-ready.

**Pace:** ~4-6 hours a week. Slower is fine — the order matters more than the speed.

---

## Phase 0 — Calibration

*Not a lesson — pointing existing knowledge at a new target before the reps start.*

**Week 0 (2-3 days): How academic security maps onto shipping software**

- **Read:** skim the [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
  Level 1 — the industry's own structured checklist. Recognize the shape from coursework; notice
  what's phrased differently.
- **Set up:** install [Burp Suite Community](https://portswigger.net/burp/communitydownload),
  clone [OWASP Juice Shop](https://github.com/juice-shop/juice-shop) locally, create a public
  GitHub repo for write-ups.

## Phase 1 — Think Like an Attacker

*The OWASP Top 10, but with hands actually on a vulnerable app instead of a slide about one.*

**Week 1: Injection & output encoding** — the two oldest bug classes in the book, still the most
common findings in real audits.
- Read: [05 §5.1-5.2](05-secure-coding-owasp.md#51-injection-sql-nosql-os-command-ldap) — injection, XSS.
- Lab: PortSwigger [SQL injection](https://portswigger.net/web-security/sql-injection) + [XSS](https://portswigger.net/web-security/cross-site-scripting) tracks.
- Write: one lab, in your own words — what broke, why, the one-line fix. First portfolio entry.

**Week 2: Forged requests & server-side trust** — both exploit the same blind spot: a server
trusting something it shouldn't.
- Read: [05 §5.3-5.4](05-secure-coding-owasp.md#53-cross-site-request-forgery-csrf) (CSRF, SSRF) + [02 §2.4](02-authorization.md#24-the-confused-deputy-problem) (confused deputy).
- Lab: PortSwigger [CSRF](https://portswigger.net/web-security/csrf) + [SSRF](https://portswigger.net/web-security/ssrf) tracks.

**Week 3: The rest of the Top 10** — deserialization, path traversal, misconfiguration, IDOR;
gamified, so this week is genuinely fun.
- Read: [05 §5.5-5.9](05-secure-coding-owasp.md#55-insecure-deserialization).
- Lab: [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) score-board — clear every
  challenge tagged for these categories.
- Track: screenshot the score-board progress.

## Phase 2 — Identity, Access & Trust

*OAuth2/OIDC already exist in the notes from the degree — this is watching them get attacked.*

**Week 4: Authentication, for real** — passwords, sessions, OAuth2/OIDC, and the one login
mechanism that's actually phishing-resistant.
- Read: [01 — Authentication](01-authentication.md), start to finish.
- Lab: register and log in with a real passkey at [webauthn.io](https://webauthn.io/). PortSwigger [Authentication](https://portswigger.net/web-security/authentication) track.

**Week 5: Authorization & sessions** — where "logged in" and "allowed to" get confused, and where
JWTs get forged.
- Read: [02 — Authorization](02-authorization.md) + [06 — Session & Token Security](06-session-and-token-security.md).
- Lab: PortSwigger [Access control](https://portswigger.net/web-security/access-control) + [JWT](https://portswigger.net/web-security/jwt) tracks.
- Write: pick one JWT lab exploit and name exactly which vulnerability from `01 §1.3` it is —
  `alg:none` or algorithm confusion.

## Phase 3 — The Wire & The Secrets

*Grading real infrastructure instead of a textbook diagram of it.*

**Week 6: TLS and where the keys actually live**
- Read: [03 — TLS/SSL](03-transport-security-tls.md) + [04 — Secrets & Key Management](04-secrets-and-key-management.md).
- Lab: grade three real sites on [Qualys SSL Labs](https://www.ssllabs.com/ssltest/). Run
  [gitleaks](https://github.com/gitleaks/gitleaks) against a large public repo's full history.
- Write: one paragraph — what did the gitleaks scan actually find, and was any of it real?

## Phase 4 — The Perimeter

*The concrete, per-endpoint things that get missed once an API ships.*

**Week 7: APIs and REST endpoints** — security headers, CORS, mass assignment; cheap fixes that
are constantly just missing.
- Read: [08 — API & Service Security](08-api-and-service-security.md) + [09 — REST Endpoint Security](09-rest-endpoint-security.md).
- Lab: grade five real sites on [securityheaders.com](https://securityheaders.com/). Proxy Juice
  Shop through Burp and read the raw requests.

## Phase 5 — The Delivery Pipeline

*The 99% of a codebase nobody wrote themselves — and how it quietly becomes the biggest attack
surface.*

**Week 8: Dependencies & supply chain** — "read a CVE and explain the actual impact" is a real
interview question; this week is that, for practice.
- Read: [10 — Infrastructure & Supply Chain](10-infrastructure-and-supply-chain.md).
- Lab: run `npm audit` (or `dotnet list package --vulnerable`) on a real project. Scan a public
  Docker image with [Trivy](https://trivy.dev/).
- Write: one CVE found, summarized in your own words.

## Phase 6 — Prove It

*Everything above, aimed at one deliverable a hiring manager can actually read.*

**Weeks 9-10 — Capstone: the assessment report**

A full, professional-grade vulnerability assessment of Juice Shop — findings ranked by severity,
each with a confirmed failure scenario and a concrete fix. This is the single artifact that proves
everything above actually landed.
- Deliver: a written report — summary → ranked findings → for each, the failure scenario, root
  cause, and fix. Style it like the real disclosed reports on
  [HackerOne Hacktivity](https://hackerone.com/hacktivity) — read three of those first as a model.

**Week 11 — Publish the portfolio**

The report plus the best 2-3 lab write-ups, made public, is the whole point of the last ten weeks.
- Publish: push the assessment report and top write-ups to the public GitHub repo. Link it from
  wherever you're applying.

**Week 12 — Choose what's next**

A decision to make deliberately, not a prescription — all three are legitimate next steps toward
an AppSec role:
- [picoCTF](https://picoctf.org/) as an ongoing weekly habit — free, beginner-friendly, genuinely fun.
- [TryHackMe](https://tryhackme.com/)'s Junior Penetration Tester path, for structured breadth.
- An entry practical certification (eJPT-class) as a resume signal, if job applications are the
  near-term goal.

---

**Next:** [1. Authentication →](01-authentication.md), or back to [the index](README.md).
