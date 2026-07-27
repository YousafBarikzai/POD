# 15 — Security Architecture

**Version:** 1.0

---

## 1. Security posture

Phase 1 is a single-operator system, but it holds assets whose compromise is materially damaging: **live marketplace credentials with write access to a revenue-generating shop**, AI provider keys with real spend attached, and a proprietary dataset that is the business's core asset. It is therefore built to a multi-tenant SaaS standard from the start — retrofitting security is harder than retrofitting features.

**Assets, ranked by consequence of compromise:**

| Rank | Asset | Consequence |
|---|---|---|
| 1 | Etsy OAuth tokens (`listings_w`) | Attacker can create, alter or delete listings on a live shop; reputational and revenue loss |
| 2 | Printify token (account-wide) | Attacker can create products and potentially incur production charges |
| 3 | AI / Ideogram API keys | Direct financial loss through spend |
| 4 | Proprietary outcome dataset | Loss of the moat |
| 5 | Operator account credentials | Full system access |
| 6 | Competitor market data | Commercially sensitive, low absolute severity |

---

## 2. Threat model (STRIDE, scoped)

| Threat | Vector | Control |
|---|---|---|
| **Spoofing** | Credential stuffing, session theft | Argon2id, mandatory TOTP, rate-limited login with lockout, opaque server-side sessions, cookie rotation on privilege change |
| **Tampering** | Modifying scores, prices, publish payloads | Server-side re-validation of every gate; audit log; append-only ledgers; optimistic concurrency on drafts |
| **Repudiation** | Disputing a publish or a legal override | Immutable `audit_log` with actor, payload, justification, IP hash |
| **Information disclosure** | Cross-tenant reads, log leakage, error messages | Workspace scoping at the data layer, RLS policies, log redaction, mapped errors, `NOT_FOUND` for cross-tenant access |
| **Denial of service** | Cost exhaustion via runaway AI/image spend | Per-run and per-workspace budgets enforced *before* spend; concurrency caps; rate limits |
| **Elevation of privilege** | Role bypass, IDOR | Authorisation derived from session only; every entity id re-resolved against `workspace_id`; role checks in middleware, not handlers |
| **Prompt injection** (LLM-specific) | Malicious text in competitor listings | Structural quarantine, no tool access on untrusted-input prompts, enum-constrained output, generation stage never sees raw external text (doc 10 §6) |
| **Supply chain** | Malicious dependency | Lockfile enforcement, SBOM, dependency scanning, provenance checks, minimal dependency surface in adapters |
| **SSRF** | Attacker-controlled URLs from imported data | Egress allowlist, URL scheme/host validation, no redirects followed to private ranges, DNS rebinding protection |

---

## 3. Authentication

| Control | Implementation |
|---|---|
| Password hashing | Argon2id, m=64 MiB, t=3, p=1, 16-byte salt; parameters versioned for future upgrade-on-login |
| Password policy | ≥ 12 characters, checked against a breached-password list (k-anonymity range query), no composition rules |
| Second factor | **TOTP mandatory**, RFC 6238, 30 s window, ±1 step drift, secret envelope-encrypted, 10 single-use recovery codes shown once and stored hashed |
| Brute force | Per-account exponential lockout (5 failures → 1 min, doubling to 1 h), per-IP sliding-window limit, constant-time comparison, no user enumeration in responses or timing |
| Sessions | Opaque 256-bit token, SHA-256 hashed at rest, `HttpOnly`/`Secure`/`SameSite=Lax`, 7-day idle expiry, 30-day absolute expiry, rotated on login and on privilege change |
| Session management | Active-session list with device/IP metadata and per-session revoke; revoke-all on password change |
| Step-up auth | Re-authentication required for: changing credentials, disconnecting an integration, overriding a `high` legal risk, and raising the monthly budget cap |

---

## 4. Authorisation

**Phase 1:** one user, role `owner`. The mechanism is nonetheless complete, because switching it on later must not require touching handlers.

```ts
// Every protected procedure resolves the workspace from the session, never from input.
const ctx = { userId, workspaceId, role }

// Every entity access re-checks tenancy at the repository layer:
async function getRun(id: string, ctx: Ctx) {
  const run = await db.run.findFirst({ where: { id, workspaceId: ctx.workspaceId, deletedAt: null } })
  if (!run) throw notFound()          // never "forbidden" — no existence disclosure
  return run
}
```

**Permission matrix (defined now, enforced from Phase 6):**

| Capability | owner | admin | member | viewer |
|---|---|---|---|---|
| View reports, concepts, artwork | ✅ | ✅ | ✅ | ✅ |
| Start research run (spends) | ✅ | ✅ | ✅ | ❌ |
| Generate artwork (spends) | ✅ | ✅ | ✅ | ❌ |
| Override legal risk | ✅ | ✅ | ❌ | ❌ |
| Publish to Etsy | ✅ | ✅ | ❌ | ❌ |
| Manage integrations/credentials | ✅ | ✅ | ❌ | ❌ |
| Change budgets / fee model | ✅ | ❌ | ❌ | ❌ |
| Activate a scoring config | ✅ | ✅ | ❌ | ❌ |
| Manage members / billing | ✅ | ❌ | ❌ | ❌ |

**Defence in depth:** application-layer scoping *and* Postgres RLS. RLS policies are written and tested in CI from day one; production flips to an RLS-enforced database role at Phase 6 by changing the connection role — a deployment change, not a code change.

---

## 5. Secrets and credential management

### 5.1 Envelope encryption

```
KMS master key (never leaves the KMS)
   └─ wraps → per-workspace DEK (AES-256, stored wrapped in integration_credentials.dek_wrapped)
         └─ encrypts → credential bundle (AES-256-GCM, integration_credentials.ciphertext)
```

| Control | Detail |
|---|---|
| Decryption | Only in worker/API process memory, on demand, via `CredentialService.get(workspaceId, provider)` |
| In-memory cache | 60 seconds maximum, keyed by workspace+provider, cleared on integration change |
| Never | Written to logs, traces, error messages, client responses, or environment dumps |
| Client display | Masked only (`sk-…4f2a`), served from a separate non-decrypting endpoint |
| Rotation | `key_version` column supports re-wrapping without re-authentication; a rotation job re-encrypts under a new master key version |
| AAD | Additional authenticated data binds ciphertext to `(workspace_id, provider, key_version)`, so a ciphertext cannot be replayed into another workspace's row |

### 5.2 Application secrets

- Injected as environment variables from the platform's secret store; never in source, never in images.
- Boot-time Zod validation; the process refuses to start on a missing or malformed secret.
- CI secret scanning (gitleaks) on every commit and on the full history; a hit fails the build.
- Rotation runbook per secret with a documented blast radius and rollback.

---

## 6. Data protection

| Layer | Control |
|---|---|
| In transit | TLS 1.3 everywhere including internal service-to-database traffic; HSTS with `preload`; certificate pinning not used (operationally brittle) but CAA records set |
| At rest | Full-disk encryption on managed Postgres and object storage; column-level envelope encryption for credentials and TOTP secrets |
| Object storage | Private buckets, no public ACLs, access exclusively via short-lived signed URLs (15 min for uploads, 1 h for reads), server-side encryption enabled, versioning on |
| Backups | Encrypted, access-controlled, restore-tested monthly |
| Deletion | Soft delete → 30-day window → hard purge including object storage; export before delete offered |
| Minimisation | No buyer PII, no reviewer identities, no competitor owner personal data (NFR-P1) |

---

## 7. Input, output and injection defence

| Vector | Control |
|---|---|
| SQL injection | Prisma parameterised queries only; a lint rule bans `$queryRawUnsafe`; the two places raw SQL is needed (vector search, partition management) use parameter binding and are individually reviewed |
| XSS | React escaping by default; `dangerouslySetInnerHTML` banned by lint; AI-generated text rendered as plain text or through a sanitising markdown renderer with an allowlist; CSP with per-request nonce and no `unsafe-inline`/`unsafe-eval` |
| CSRF | Double-submit token on all cookie-authenticated mutations; `SameSite=Lax`; SSE endpoints are GET-only and side-effect-free |
| SSRF | Any outbound URL derived from user or imported data is validated against an allowlist of schemes and hosts; private/link-local/metadata ranges blocked at the resolver; redirects not followed cross-origin; separate egress path for image fetching with its own allowlist |
| File upload | Magic-byte type sniffing (not extension or declared MIME), size caps (50 MB CSV, 40 MB image), stored outside any executable path, served only via signed URLs with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` |
| CSV injection | Exported CSVs prefix cells beginning `= + - @ \t \r` with a single quote |
| Zip/decompression bombs | Not accepted (no archive uploads) |
| Deserialisation | JSON only, schema-validated; no `eval`, no dynamic `require`, no prototype-pollution-prone merges (`Object.assign` on validated shapes only) |
| Path traversal | Object keys are generated server-side from UUIDs; user input never contributes to a storage path |

---

## 8. AI-specific security

Covered fully in [doc 10 §6](10-ai-orchestration.md); the security-relevant summary:

1. **Untrusted external text is quarantined** in delimited blocks, appended after instructions, never interpolated into them.
2. **Prompts that consume untrusted data have no tool access** and emit enum-constrained classification output only.
3. **The generative stage never receives raw external text** — only aggregated statistics. Injection therefore has no path to influence what the system creates.
4. **Model output is never executed**, never used to build a URL, query, path or provider parameter without allowlist validation.
5. **Output enums are validated, not coerced.** An out-of-vocabulary value is an error, not a new category.
6. **Spend is bounded before the call**, so a prompt that induces long generation cannot produce unbounded cost.

---

## 9. Legal & IP safety as a security control

The Legal & Safety Engine is treated as a security gate, not a feature:

- It is enforced **at the service layer** (FR-901). A client that bypasses the UI still cannot generate artwork for a blocked concept — the check happens where the external call is made.
- Overrides require step-up authentication, a typed confirmation and a written justification, all recorded immutably.
- Screening records and registry responses are retained for 7 years to support any future dispute.
- A workspace-level "burned terms" list permanently blocks anything that previously caused a takedown.

---

## 10. Network and infrastructure

| Control | Detail |
|---|---|
| Ingress | Only ports 443 exposed; platform WAF with managed rulesets; DDoS protection at the edge |
| Egress | Explicit allowlist of provider hostnames from worker processes; anything else denied and logged as an anomaly |
| Database | No public IP; access via private networking or an authenticated proxy; separate credentials per service with least-privilege grants (the web role cannot `DROP`, the worker role cannot read `audit_log`) |
| Redis | Password + TLS, private networking, no public exposure |
| Containers | Non-root user, read-only root filesystem, no capabilities beyond defaults, minimal base image, pinned digests |
| Admin access | No SSH into production; all operations via the platform console with MFA and audit; break-glass procedure documented with a mandatory post-incident review |

---

## 11. Security headers

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'nonce-{random}';
  img-src 'self' data: https://{signed-asset-host};
  connect-src 'self';
  frame-ancestors 'none'; base-uri 'self'; form-action 'self';
  object-src 'none'; upgrade-insecure-requests
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
```

---

## 12. Audit logging

**Always logged** (immutable, 7-year retention): authentication events (success, failure, lockout, TOTP enrolment/reset), session creation and revocation, credential connect/disconnect/rotate, budget changes, **every publish**, **every legal override**, scoring-config activation, data export, deletion requests, and role changes.

**Record shape:** actor (user id or `system`), action, entity type and id, before/after state, justification where applicable, IP hash (not raw IP), user agent, correlation id, timestamp.

**Integrity:** append-only enforced by revoking `UPDATE`/`DELETE` grants on the table; a nightly job verifies a rolling hash chain over the day's entries and alerts on any discontinuity.

---

## 13. Vulnerability management

| Activity | Cadence | Gate |
|---|---|---|
| Dependency scanning (`pnpm audit`, Dependabot/Renovate) | Every PR + daily | Critical/high blocks merge |
| SAST (CodeQL / Semgrep with custom rules) | Every PR | High blocks merge |
| Secret scanning (gitleaks) | Every PR + full history weekly | Any hit blocks |
| Container image scanning (Trivy) | Every build | Critical blocks deploy |
| SBOM generation (CycloneDX) | Every release | Stored with the release artefact |
| Manual security review | Every phase gate | Sign-off required |
| Penetration test | Before SaaS launch, annually thereafter | Findings triaged to closure |
| Dependency freshness | Monthly | No dependency more than two majors behind without a documented reason |

**Custom Semgrep rules** enforce the architecture's security invariants: no vendor SDK imported outside its adapter; no `dangerouslySetInnerHTML`; no `$queryRawUnsafe`; no credential access outside `CredentialService`; no `fetch` to a non-allowlisted host constructed from a variable.

---

## 14. Incident response

**Severity ladder**

| Sev | Definition | Response |
|---|---|---|
| **1** | Credential compromise, data breach, unauthorised marketplace write | Immediate: revoke all tokens, invalidate all sessions, disable publishing, preserve logs, notify operator within 1 h, ICO notification assessment within 72 h if personal data is implicated |
| **2** | Unauthorised access attempt succeeded but was contained; significant unexplained spend | Rotate affected credentials, audit the blast radius, patch, report within 24 h |
| **3** | Vulnerability discovered, not exploited | Patch within the SLA for its severity; document |
| **4** | Policy or hygiene issue | Backlog with an owner and a date |

**Standing runbooks:** `credential-compromise`, `unexpected-spend`, `unauthorised-publish`, `provider-breach-notification`, `data-restore`, `rls-misconfiguration`. Each names the detection signal, the containment step, the eradication step, the recovery step, and the evidence to preserve.

**Kill switches (single-toggle, tested quarterly):** disable all publishing; disable all AI spend; disable all amber-tier data adapters; revoke all sessions; enter maintenance mode.

---

## 15. Compliance readiness

| Requirement | Status in Phase 1 | Needed for SaaS |
|---|---|---|
| Lawful basis documented | ✅ | ✅ |
| Data inventory / RoPA | ✅ | ✅ |
| Subprocessor register | ✅ | ✅ + public page |
| DPAs with subprocessors | Operator-level | ✅ signed, all |
| Export (portability) | ✅ FR-1705 | ✅ |
| Erasure | ✅ soft+hard delete | ✅ with SLA |
| Breach notification process | ✅ documented | ✅ tested |
| Cookie consent | N/A (strictly necessary only) | ✅ if analytics added |
| Terms & privacy policy | Operator-only | ✅ published |
| Pen test | — | ✅ pre-launch |
| SOC 2 readiness | — | Stage 4 consideration |
