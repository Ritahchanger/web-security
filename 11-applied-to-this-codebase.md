# 11. Applied: This Codebase (iNXG-backend)

Everything above is more useful once you can see it land on real code. Here's how the concepts map
onto the actual authentication/authorization implementation in `iNXG-backend`, which we
reverse-engineered together earlier in this project.

## 11.1 Which authentication type is this, exactly?

iNXG-backend's default scheme is **not** a self-contained JWT, even though the header looks like
one (`Authorization: Bearer <token>`). It's closer to classic **server-side session
authentication** ([01 §1.2](01-authentication.md)), just wearing a bearer-token header instead of
a cookie:

- `SharedModules/Security/IOM.Shared.Security.Api/SecurityAuthenticationHandler.cs` treats the
  token as an **opaque session key** — it looks it up via `ISessionService.Fetch` against
  `SEC.UserSession`, and builds the `ClaimsPrincipal` from whatever it finds in the database *at
  that moment*, on every single request.
- This is why the session is **instantly revocable server-side** (delete the `SEC.UserSession`
  row and every future request with that token fails immediately) — the single biggest practical
  advantage of session-style auth over a self-contained JWT, exactly as described in
  [01 §1.2-1.3](01-authentication.md). A pure JWT architecture would need a denylist to get this
  same property; this system gets it for free because it was never actually stateless.
- **Why this matters in practice:** if someone asks "can we force-logout a compromised user
  account right now," the honest answer here is yes, trivially — which would not be true if this
  were genuine stateless JWT auth without additional denylist infrastructure.

## 11.2 The RBAC model, matched to §2.2

`SEC.User → UserRole → Role → RolePermission → Permission`, resolved fresh per session lookup by
`UserSessionRepository.GetUserPermissions` — this is textbook RBAC
([02 §2.2](02-authorization.md)): permissions are never assigned to users directly, only to roles,
and users are assigned to roles. The catalogue of *possible* permissions is code-defined per
module (`PermissionPatcherHostedService` upserts each module's declared permissions at startup),
but *who has what* lives entirely in the database — which is the right split: the shape of what
can be granted is a deploy-time concern, who actually holds a grant is a runtime, admin-managed
concern.

`SecurityAuthorizationHandler` also implements a super-admin bypass keyed on
`ClaimTypes.Actor == UserType.SuperUser` — worth flagging as a pattern to watch, per
[02 §2.5](02-authorization.md) (least privilege): a hardcoded bypass that skips the normal
permission check entirely is powerful and convenient, but it's also a single point that, if its
condition is ever satisfiable unintentionally (a bug in how `UserType.SuperUser` gets assigned),
grants total access with no further check — exactly the kind of code path that deserves an extra
review pass and dedicated test coverage disproportionate to its size.

## 11.3 Cross-module HTTP calls — token propagation, matched to §8.3

`AuthHeaderPropagationHandler` (in both the Movement and MigrantTraining modules) forwards the
*original caller's own bearer token* onto outgoing integration-port HTTP calls between modules.
This is the **token propagation / on-behalf-of** pattern from
[08 §8.3](08-api-and-service-security.md), not client-credentials — and it's the right choice
here: the downstream module's `[Authorize(Policy)]` checks apply to the real originating user, not
to some broad "Movement module" service identity. If Movement instead called CaseManagement using
its own powerful internal service credentials regardless of who initiated the request, a bug in
Movement's own authorization would silently grant that user CaseManagement's data too — the
confused-deputy pattern from [02 §2.4](02-authorization.md).

## 11.4 A real gap we found together, matched to §2.6 (default-deny)

`RulesEndpointPolicyTests.cs`'s own comment documents that **modules currently disagree** on
default endpoint authorization: BusinessSystemConfiguration and Medical have `[AllowAnonymous]`
endpoints, DataIntegrationManagement requires an admin policy, CaseManagement and Individual
declare nothing and inherit whatever the host's default happens to be. There is no global
fallback/default authorization policy configured anywhere in the host.

This is exactly the "default-allow, not default-deny" risk described in
[02 §2.6](02-authorization.md): every new endpoint requires someone to remember to add the right
`[Authorize]` attribute, and nothing stops a forgotten one from silently defaulting to anonymous
access. The test file existing at all is the right defensive pattern (a ratcheted architecture
test that at least prevents *new* endpoints from silently regressing further) — but it's a
detection mechanism for a gap that ideally shouldn't exist in the first place; a genuine
`FallbackPolicy = RequireAuthenticatedUser()` at the host level would close this class of bug
structurally instead of relying on a test to catch each instance individually.

## 11.5 TLS in this specific system — the loopback HTTP integration ports

The cross-module HTTP integration ports (§10.3) call `http://localhost:3320` in the dev config —
plain HTTP, not HTTPS, because it's a loopback call within the same process's own listening port
in development. This is a legitimate, common dev-environment simplification, but it's worth being
explicit about the boundary: **in production, this same call pattern must run over TLS**, per
[03](03-transport-security-tls.md) — an internal loopback in dev becomes a real network hop
between service instances (or at minimum, between containers/pods) once actually deployed, and
"it's internal" is exactly the assumption [08 §8.1](08-api-and-service-security.md) warns against
resting security on.

## 11.6 What we didn't find: Row-Level Security

The architecture doc audit we did earlier found that **Row-Level Security is mentioned in this
project's own architecture documentation but doesn't exist anywhere in the actual schema** — no
migration creates a security policy or predicate function. Schema-per-module isolation is real and
does provide genuine separation between modules, but it's a coarser boundary than RLS — it stops
one *module* from directly querying another module's tables, but it does nothing to stop, say, one
mission's users from querying another mission's rows within the *same* module's own tables if the
application-layer `WHERE` clause is ever missing or wrong on a given query. That per-row filtering
is currently entirely the application layer's responsibility (e.g. the `missionCode` matching we
implemented for the Movement Number Allocation check) — worth knowing precisely where the
enforcement boundary actually sits, since "the database enforces this" and "our repository code
enforces this, we hope consistently" are very different guarantees.
