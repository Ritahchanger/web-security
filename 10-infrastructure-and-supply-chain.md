# 10. Infrastructure & Supply Chain Security

Most of the previous files are about code you write. This one is about code and systems you
*didn't* write but ship anyway — which, in a modern application, is the overwhelming majority of
what actually runs in production (a typical app is 1% first-party code, 99% dependencies).

## 10.1 Dependency vulnerabilities are your vulnerabilities

Using a package with a known CVE is functionally identical to writing that vulnerability yourself
— your users are exposed either way, and "I didn't write that bug" is not a mitigating factor in
an incident. Real, large-scale breaches (Equifax/Struts, Log4Shell) trace directly back to a known,
patched vulnerability in a dependency that simply wasn't updated in time.

**Practical controls:**
- Automated scanning in CI (`npm audit`, `dotnet list package --vulnerable`, GitHub Dependabot
  alerts, Snyk) — run on every build, not as an occasional manual check someone remembers to do.
- A defined policy for *how fast* a vulnerability of a given severity must be patched once
  disclosed — "critical, remotely exploitable, no auth required" should be a same-day patch, not
  "next sprint."
- Pin dependency versions (lockfiles — `package-lock.json`, `packages.lock.json`) so builds are
  reproducible and an unreviewed transitive-dependency update can't silently change what ships.

## 10.2 Software Bill of Materials (SBOM)

A machine-readable inventory of every component (direct and transitive) in a shipped artifact —
name, version, license, sometimes cryptographic hash. The practical value: when the next
Log4Shell-scale disclosure happens, "which of our 40 services actually use the vulnerable version
of this library, transitively, three dependencies deep" is a question you can answer in minutes by
querying SBOMs, instead of days of manual auditing under incident pressure. Increasingly a
contractual/regulatory requirement for software sold into government and critical-infrastructure
sectors (US Executive Order 14028), and good practice regardless.

## 10.3 Supply chain attacks — the dependency itself is malicious, not just outdated

A different, more severe threat than "an old version has a bug": the package itself is
compromised, deliberately, by an attacker.
- **Typosquatting** — publishing a malicious package with a name one character off from a popular
  one (`reqeusts` vs `requests`), hoping for a typo in someone's install command.
- **Maintainer account takeover** — an attacker compromises a legitimate, trusted maintainer's
  publish credentials and pushes a malicious version of a real, widely-used package directly (the
  `event-stream` npm incident, and the XZ Utils backdoor, are real examples of this exact pattern).
- **Build-time compromise (SolarWinds-style)** — the attacker doesn't touch the published package
  at all; they compromise the *build pipeline* that produces it, injecting malicious code only
  into the compiled artifact, which can evade source-code review entirely since the malicious code
  never appears in the reviewable source.

**Mitigations:** pin exact versions and verify checksums/lockfile integrity on install (don't
auto-accept `^`/`~` version ranges for anything security-sensitive without review); prefer
packages with a track record and active maintenance over convenience; consider a private package
registry proxy that caches and can gate/scan packages before they reach your build; verify
signed releases and provenance attestations (e.g. Sigstore/`cosign`) where publishers provide them.

## 10.4 Container security

- **Don't run containers as root** — a container escape (a real, if less common, vulnerability
  class) is far more damaging if the process inside was running as root, since it maps to root on
  the host in some misconfigurations, or at minimum grants the attacker root-equivalent capability
  within the container's own filesystem and any mounted volumes.
- **Minimal base images** — a `distroless` or `alpine`-based image has a dramatically smaller
  attack surface (fewer installed packages, no shell, no package manager for an attacker to
  `apt install` tools with post-compromise) than a full OS image.
- **Scan images for known-vulnerable OS packages**, not just application dependencies (Trivy,
  Grype) — a container's base OS layer accumulates CVEs over time just like any dependency does.
- **Immutable infrastructure** — rebuild and redeploy rather than patching a running container
  in-place, so there's no drift between what's documented/scanned and what's actually running.

## 10.5 CI/CD pipeline security

Covered partly in [04 §4.5](04-secrets-and-key-management.md) — additionally:
- **Least-privilege pipeline credentials** — a CI job that only runs tests should not hold
  production deploy keys; scope credentials to the specific stage that needs them.
- **Signed commits and branch protection** — require review before merge to protected branches,
  require signed commits where feasible, so "who actually approved this change" is verifiable, not
  just assumed.
- **Reproducible builds** where practical — the ability to independently verify that a published
  artifact actually corresponds to the source code it claims to, closing the gap a build-time
  supply-chain compromise (§9.3) exploits.

## 10.6 Infrastructure-as-Code (IaC) misconfiguration scanning

A huge share of real cloud breaches trace back not to a code vulnerability at all, but to a
misconfigured cloud resource — a publicly-readable S3 bucket, an overly permissive IAM policy, a
security group open to `0.0.0.0/0` on a database port. Tools like Checkov, tfsec, or cloud-native
posture-management scanners can catch these *before* apply, in the same CI pipeline that already
scans application dependencies — treating infrastructure configuration with the same rigor as
application code, because a misconfigured resource is exploitable exactly like a code
vulnerability is.

## Further reading

- [CycloneDX SBOM standard](https://cyclonedx.org/) · [SPDX SBOM standard](https://spdx.dev/)
- [CISA — Executive Order 14028 on Improving the Nation's Cybersecurity](https://www.cisa.gov/executive-order-improving-nations-cybersecurity)
- [CISA advisory — XZ Utils supply chain compromise (2024)](https://www.cisa.gov/news-events/alerts/2024/03/29/reported-supply-chain-compromise-affecting-xz-utils-data-compression-library)
- [CISA — what happened in the SolarWinds supply chain attack](https://www.cisa.gov/news-events/news/what-you-need-know-about-solarwinds-supply-chain-attack)
- [The `event-stream` npm incident — original GitHub issue thread](https://github.com/dominictarr/event-stream/issues/116)
- [Trivy — vulnerability/misconfiguration scanner for containers, IaC, and dependencies](https://trivy.dev/)
- [Sigstore / cosign — signing and verifying software artifacts](https://www.sigstore.dev/)
- [Checkov — static analysis for Infrastructure-as-Code](https://www.checkov.io/)
- [OWASP Software Component Verification Standard (SCVS)](https://owasp.org/www-project-software-component-verification-standard/)

---
**Next:** [11. Applied: This Codebase →](11-applied-to-this-codebase.md)
