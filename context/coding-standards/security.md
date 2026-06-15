# Security

Security defects are exploitable in production, expensive to remediate, and — unlike most bugs — actively sought out by adversaries; one missing check is enough. Violations are BLOCK-severity for the safety reviewer lens.

## Anchors

- **OWASP Top 10 2025** — the recurring web defect classes (injection, broken access control, cryptographic failures, SSRF) behind most breaches
- **CWE Top 25** — the most dangerous software weaknesses, ranked by real-world exploitation
- **OWASP Top 10 for LLM Applications 2025** — prompt injection, insecure output handling, excessive agency
- **OWASP ASVS** — the verification standard for what "validated input" and "correct authentication" actually require
- **Saltzer & Schroeder (1975)** — Least Privilege, Economy of Mechanism, Complete Mediation, Fail-Safe Defaults
- **Parse, Don't Validate** (Alexis King, 2019) — turn untrusted input into a domain type at the boundary; never propagate unparsed data

> Three concerns are owned in depth by sibling standards and restated here only at their security-relevant edge: software supply chain (see Dependencies Q1), fail-closed defaults and parse-at-boundary for persisted state (see Data Integrity Q13.3 and Q13.6), and boundary error mapping (see Error Handling Q7). Where this file repeats a rule, it composes with — never contradicts — the owning standard. The rules below are almost entirely pure safety/correctness, so they are MUST-strength and absolute; the few taste/operational judgment calls are SHOULD-strength and each carries an explicit, documented escape hatch.

## Q10.1 — Secrets, Configuration, and Transport

- **MUST NOT** put secrets (API keys, tokens, passwords, private keys, connection strings, signing material) in source, fixtures, committed config, logs, error messages, analytics, or client-shipped bundles — use environment variables, a secret manager, or build-time injection.
- **MUST NOT** hardcode production URLs, hostnames, account identifiers, or endpoints — resolve them through configuration.
- **MUST** use TLS (HTTPS, WSS, or equivalent) for all network communication; cleartext HTTP and any path that silently downgrades to HTTP on failure **MUST NOT** ship to production.
- **MUST NOT** disable TLS certificate or hostname verification (no "trust all certs", no `verify=False`, no accept-all `TrustManager`, no disabled hostname check); pinning, where used, adds to verification, never replaces it.
- **MUST** treat any secret that appeared in source, a log, or a screenshot as compromised and rotate it.
- **SHOULD** load secrets once at a single composition point and pass them as typed values rather than re-reading from the environment scattered through the code. *Escape hatch: a process needing lazy/late binding (a credential refreshed mid-run, a secret rotated without restart) MAY re-read at a documented refresh site.*

```typescript
// WRONG — secret in source, plaintext transport, verification disabled
const API_KEY = "sk_live_EXAMPLE_hardcoded_secret"       // committed forever
await fetch("http://payments.prod.internal/charge", {     // cleartext
  agent: new https.Agent({ rejectUnauthorized: false }),  // accepts any cert
  headers: { Authorization: `Bearer ${API_KEY}` },
})

// RIGHT — injected secret, TLS enforced, verification on (default)
const apiKey = requireEnv("PAYMENTS_API_KEY")             // from secret manager
const baseUrl = config.paymentsBaseUrl                    // https://… resolved per environment
await fetch(`${baseUrl}/charge`, {
  headers: { Authorization: `Bearer ${apiKey}` },         // no agent override; cert + host verified
})
```

## Q10.2 — Cryptography

- **MUST NOT** create custom cryptographic primitives (ciphers, hashes, MACs, KDFs, RNGs, padding schemes) — use the platform/vetted library exclusively.
- **MUST NOT** use broken or deprecated-for-security algorithms: MD5, SHA-1, DES, 3DES, RC4, ECB mode, PKCS#1 v1.5 encryption — use an authenticated mode (AES-GCM, ChaCha20-Poly1305).
- **MUST** store passwords with a memory-hard or purpose-built password hash (Argon2id, scrypt, bcrypt) — never a raw or fast hash (SHA-256/SHA-512 alone), never encryption.
- **MUST** source security-relevant random values (tokens, session ids, nonces, salts, IVs, password-reset codes) from a cryptographically secure RNG (`crypto.randomBytes`, `SecureRandom`, `secrets`, `SystemRandom`, `os.urandom`); a non-crypto PRNG (`Math.random`, `java.util.Random`, `rand()`, `random.random`) **MUST NOT** be used for these.
- **MUST NOT** reuse a nonce/IV with the same key, and **MUST NOT** hardcode key material or derive it from a low-entropy constant.
- **MUST** use a constant-time comparison for secret comparisons (MACs, tokens, password hashes).

```kotlin
// WRONG — fast hash for passwords, broken cipher mode, predictable token, leaky compare
val stored = MessageDigest.getInstance("SHA-256").digest(password.toByteArray())   // unsalted, fast
val cipher = Cipher.getInstance("AES/ECB/PKCS5Padding")                            // ECB leaks structure
val resetToken = Random().nextInt().toString()                                     // predictable
if (providedToken == expectedToken) grantAccess()                                  // timing side-channel

// RIGHT — memory-hard hash, authenticated mode, CSPRNG token, constant-time compare
val stored = Argon2id.hash(password)                          // salted, memory-hard, tunable cost
val cipher = Cipher.getInstance("AES/GCM/NoPadding")          // authenticated; fresh 96-bit IV per message
val bytes = ByteArray(32).also { SecureRandom().nextBytes(it) }              // 256 bits of CSPRNG entropy
val resetToken = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes)
if (MessageDigest.isEqual(provided, expected)) grantAccess()  // constant-time
```

## Q10.3 — Injection Prevention

- **MUST** trace user-controlled input to every sink: SQL, NoSQL query objects, OS commands, LDAP, XPath, XML/XXE, template engines, regular expressions, and any `eval`-class evaluator.
- **MUST NOT** build SQL, shell commands, HTML, URLs, LDAP/XPath filters, or template source from user data via string concatenation, interpolation, or format strings — use parameterized queries, argument-vector process execution, contextual auto-escaping template engines, and URL builders.
- **MUST NOT** dynamically evaluate any externally-influenced string (`eval`, `Function`, `exec`, `system`, shell `-c`, reflective class loading by name).
- **MUST** disable external entity and DTD processing (XXE) by default in XML parsers.
- **MUST** escape a user-supplied value used inside a regular expression, and patterns applied to user input **MUST NOT** contain catastrophic-backtracking constructs (nested unbounded quantifiers / ReDoS). *Escape hatch: an intentionally pattern-accepting feature (an admin-defined search filter) MAY accept a regex only after compiling it with a length cap and a match timeout, documented at the call site.*

```python
# WRONG — SQL by interpolation; shell from a concatenated string; HTML sink from user data
db.execute(f"SELECT * FROM users WHERE email = '{email}'")        # ' OR '1'='1
os.system("convert " + filename + " out.png")                     # filename = "x; rm -rf /"
el.innerHTML = f"<div>{user_comment}</div>"                       # stored XSS

# RIGHT — parameterized query; argument vector with no shell; text sink, not markup
db.execute("SELECT * FROM users WHERE email = %s", (email,))      # driver binds, never parses email as SQL
subprocess.run(["convert", filename, "out.png"], shell=False)     # no shell to inject into
el.textContent = user_comment                                     # browser never parses it as markup
```

## Q10.4 — Trust Boundaries, SSRF, and Path Traversal

- **MUST** parse every input crossing a trust boundary — network, file system, environment, IPC, deep links, push notifications, clipboard, query parameters, headers, deserialized blobs — into a rejecting domain type at the boundary (parse-don't-validate is owned by data-integrity.md Q13.3; this section adds the security-specific sinks that compose on top of it).
- **MUST** validate any user-influenced server-side request target (host, URL, address) against an explicit allowlist and reject private, loopback, link-local, and metadata-endpoint addresses (SSRF); a denylist of "bad" hosts **MUST NOT** be the primary control.
- **MUST** canonicalize a user-derived file path and confine it to an intended base directory, re-checking the resolved real path is inside that base before any open; `..`, absolute paths, and base-escaping symlinks **MUST** be rejected (path traversal / zip-slip).
- **MUST NOT** allow an open redirect: a post-login or callback `returnTo`/`next` URL from a request **MUST** be validated against an allowlist of in-app destinations before redirecting.
- **MUST NOT** deserialize untrusted bytes into arbitrary object graphs via a deserializer that can instantiate attacker-chosen types (native Java/.NET serialization, `pickle`, unsafe YAML) — use a data-only format (JSON) parsed into known domain types.
- **SHOULD** bound boundary-accepted input in size at the edge (max body, max field length, max collection size, max nesting depth). *Escape hatch: a streaming endpoint that genuinely must accept large payloads MAY raise or remove the size cap, provided it streams rather than buffers and the decision is documented.*

```typescript
// WRONG — fetches whatever URL the user supplies (SSRF: hits the cloud metadata endpoint)
async function fetchPreview(url: string): Promise<Preview> {
  const res = await fetch(url)            // url = "http://169.254.169.254/latest/meta-data/iam/..."
  return parsePreview(await res.text())
}

// RIGHT — allowlist host, resolve the IP, reject internal ranges, then connect to the pinned IP
async function fetchPreview(rawUrl: string): Promise<Preview> {
  const url = new URL(rawUrl)                                  // parse → throws on garbage
  if (!ALLOWED_PREVIEW_HOSTS.has(url.hostname)) throw new ForbiddenTarget(url.hostname)
  const ip = await resolveToPublicIp(url.hostname)            // throws on private/loopback/link-local/metadata range
  // Connect to the exact resolved IP so a second DNS lookup cannot rebind to an internal address (closes the TOCTOU/DNS-rebinding gap).
  const res = await fetchWithPinnedIp(url, ip, { redirect: "manual" })   // do not silently follow into internal space
  return parsePreview(await res.text())
}
```

```python
# WRONG — path traversal: user controls the filename, escapes the upload dir
def read_upload(name: str) -> bytes:
    return open(os.path.join(UPLOAD_DIR, name), "rb").read()   # name = "../../etc/passwd"

# RIGHT — canonicalize and confine to the base directory
def read_upload(name: str) -> bytes:
    base = os.path.realpath(UPLOAD_DIR)
    target = os.path.realpath(os.path.join(base, name))
    if os.path.commonpath([base, target]) != base:            # resolved path must stay inside base
        raise ValueError("path escapes upload directory")
    return open(target, "rb").read()
```

## Q10.5 — Authentication, Authorization, and Session Integrity

- **MUST** perform a server-side authorization check on every request that reads or mutates protected data; an absent check **MUST** default to deny (fail closed, see Data Integrity Q13.6). UI hiding is not authorization.
- **MUST** enforce object-level authorization for every accessed object — the caller proven to own or be permitted the specific id, not merely authenticated (returning `/orders/{id}` for any id the caller names is broken-object-level-authorization / IDOR).
- **MUST** derive authorization decisions from server-side identity and server-held permissions, never from a client-supplied role, flag, price, user id, or `isAdmin` parameter.
- **MUST** expire session tokens and credentials, invalidate them on logout and credential change, and transmit them only over TLS; cookies carrying them **MUST** be `HttpOnly`, `Secure`, and `SameSite`.
- **MUST** protect state-changing requests served from a browser session against cross-site request forgery (anti-CSRF token or equivalent same-site enforcement).
- **SHOULD** enforce rate limiting / lockout on authentication and other abusable endpoints. *Escape hatch: an internal-only endpoint behind an authenticated trusted boundary MAY omit per-endpoint rate limiting where an upstream gateway provides it; record where the control lives.*

```kotlin
// WRONG — trusts a client-supplied role and never checks object ownership
fun getInvoice(req: Request): Invoice {
    if (req.query("role") == "admin") return invoiceRepo.find(req.query("id"))   // client says "admin"
    throw Forbidden()
}

// RIGHT — server identity + object-level ownership check, fail closed.
// Unauthorized and not-found return the SAME response so an attacker cannot
// enumerate which ids exist (a distinct Forbidden vs NotFound is an existence
// oracle; auth/authz failures stay indistinguishable per error-handling.md Q7.9).
fun loadInvoice(req: AuthenticatedRequest, id: InvoiceId): Invoice {
    val invoice = invoiceRepo.find(id)
    val authorized = invoice != null &&
        (invoice.ownerId == req.principal.userId || req.principal.can(VIEW_ANY_INVOICE))
    if (!authorized) throw NotFound()                  // identical for "missing" and "not yours"
    return invoice
}
```

## Q10.6 — Sensitive Data Handling and Logging

- **MUST NOT** write secrets, credentials, full payment card numbers, government identifiers, auth tokens, or session ids to logs, traces, analytics, crash reports, or error responses — redact or omit them at the logging boundary.
- **MUST** map internal failures returned to external consumers to a generic client-safe message plus a correlation id (owned by error-handling.md Q7; the secrets/PII case is covered above).
- **MUST** encrypt personal and sensitive data in transit (Q10.1) and **SHOULD** encrypt it at rest where the platform/store supports it, with access minimized and logged. *Escape hatch: a store with no encryption-at-rest capability MAY hold sensitive data unencrypted only when the gap is compensated (e.g. disk-level encryption, restricted access) and documented at the persistence site.*
- **SHOULD** keep sensitive in-memory values in narrow scope and not copy them into long-lived caches, URLs (which land in server and proxy logs), or client-persisted storage. *Escape hatch: a value that must be cached for a legitimate session MAY be, provided the cache is access-controlled, TLS-only, and has a bounded lifetime documented at the cache site.*
- **MUST NOT** ship debug/verbose logging or developer backdoors (test accounts, auth bypass flags, `if (debug) skipAuth`) enabled to production.

```typescript
// WRONG — logs the full request including the Authorization header and card number
logger.info("charge request", { headers: req.headers, body: req.body })   // secret + PAN now in logs forever

// RIGHT — log only non-sensitive fields; redact known-sensitive keys at the boundary
logger.info("charge request", {
  correlationId: req.id,
  amount: req.body.amount,
  cardLast4: req.body.card.number.slice(-4),       // never the full PAN; Authorization header omitted
})
```

## Q10.7 — LLM Application Security

- **MUST** keep user-supplied and externally-retrieved content (RAG passages, tool outputs, web pages, file contents, prior untrusted turns) in a separate, clearly delimited channel from system instructions, never concatenated into the trusted instruction region (prompt injection / indirect injection).
- **MUST** treat LLM output as untrusted input to whatever consumes it — validated, escaped for its sink (HTML, SQL, shell; all of Q10.3 applies), and constrained to an expected length/format before it drives any action.
- **MUST** run an LLM-driven agent under least privilege — granted only the specific tools and scopes the task requires; any high-impact or irreversible action (payment, deletion, external send, privilege change) **MUST** require an explicit out-of-band confirmation or human approval step rather than auto-executing from model output.
- **MUST** validate and authorize model-produced tool/function arguments server-side exactly as if a hostile user supplied them (Q10.4, Q10.5) — the model's "deciding" to call a tool grants no authority.
- **MUST NOT** place secrets or system-prompt contents where the model can echo them to the user; assume any token in context can be exfiltrated by a crafted prompt.

```python
# WRONG — untrusted retrieved text concatenated into the system instruction; output run as a shell command
prompt = SYSTEM_RULES + "\n" + retrieved_doc + "\n" + user_msg     # doc can override the rules
os.system(llm.complete(prompt))                                    # arbitrary command execution

# RIGHT — separate channels, validate output, gate the irreversible action
messages = [
    {"role": "system", "content": SYSTEM_RULES},
    {"role": "user", "content": user_msg},
    {"role": "user", "content": f"<retrieved untrusted=\"true\">{retrieved_doc}</retrieved>"},
]
choice = llm.complete(messages, tools=[SEARCH_TOOL])               # only the low-risk tool is exposed
call = parse_tool_call(choice)                                     # reject anything off-schema
if call.name == "delete_account":                                  # high-impact → never auto-run
    require_human_approval(call.args)
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | No secrets in source, fixtures, config, logs, or client bundles | Required | Q10.1 |
| 2 | No hardcoded production URLs/hosts/account ids — resolved via config | Required | Q10.1 |
| 3 | TLS everywhere; no cleartext HTTP and no silent downgrade in production | Required | Q10.1 |
| 4 | TLS certificate and hostname verification never disabled | Required | Q10.1 |
| 5 | Any exposed secret treated as compromised and rotated | Required | Q10.1 |
| 6 | Secrets loaded once at a single composition point, not re-read scattered (or refreshed at a documented site) | Required | Q10.1 |
| 7 | Platform/vetted crypto only — no custom primitives | Required | Q10.2 |
| 8 | No broken/deprecated algorithms (MD5, SHA-1, DES/3DES, RC4, ECB, PKCS#1 v1.5) | Required | Q10.2 |
| 9 | Passwords hashed with Argon2id/scrypt/bcrypt, never a fast hash or encryption | Required | Q10.2 |
| 10 | Security-relevant randomness from a CSPRNG, never a non-crypto PRNG | Required | Q10.2 |
| 11 | No nonce/IV reuse under one key; no hardcoded/low-entropy key material | Required | Q10.2 |
| 12 | Secret/MAC/token comparisons are constant-time | Required | Q10.2 |
| 13 | User input traced to every injection sink (SQL/NoSQL/OS/LDAP/XPath/XML/template/regex) | Required | Q10.3 |
| 14 | No concatenation building SQL/shell/HTML/URL/LDAP/template from user data; parameterized/escaped instead | Required | Q10.3 |
| 15 | No dynamic eval/exec/system of externally-influenced strings | Required | Q10.3 |
| 16 | XML parsers disable external entities and DTDs (no XXE); no ReDoS-prone patterns on user input | Required | Q10.3 |
| 17 | Every trust-boundary input parsed into a domain type (shape/type/range/length/encoding) | Required | Q10.4 |
| 18 | User-influenced outbound targets allowlisted; private/loopback/link-local/metadata addresses rejected (SSRF) | Required | Q10.4 |
| 19 | User-derived file paths canonicalized and confined to a base directory (no traversal/zip-slip) | Required | Q10.4 |
| 20 | Redirect targets allowlisted (no open redirect); no unsafe deserialization of untrusted bytes | Required | Q10.4 |
| 21 | Boundary inputs size/length/depth bounded (or streamed under documented escape hatch) | Required | Q10.4 |
| 22 | Server-side authorization on every protected read/mutate; absent check defaults to deny | Required | Q10.5 |
| 23 | Object-level ownership/permission checked per accessed id (no IDOR) | Required | Q10.5 |
| 24 | Authorization from server-held identity/permissions, never a client-supplied role/flag | Required | Q10.5 |
| 25 | Sessions/tokens expire, invalidate on logout/credential change; cookies HttpOnly/Secure/SameSite | Required | Q10.5 |
| 26 | State-changing browser requests protected against CSRF; auth endpoints rate-limited (or upstream-documented) | Required | Q10.5 |
| 27 | No secrets/PAN/identifiers/tokens in logs, traces, analytics, crash reports, or error responses | Required | Q10.6 |
| 28 | No internal error detail (stack/SQL/paths) returned externally — generic message + correlation id | Required | Q10.6 |
| 29 | Sensitive data encrypted at rest where supported (or compensated/documented); not placed in URLs or client storage | Required | Q10.6 |
| 30 | No debug/verbose logging shipped enabled and no auth-bypass backdoors in production | Required | Q10.6 |
| 31 | LLM: untrusted/retrieved content separated from system instructions (prompt + indirect injection) | Required | Q10.7 |
| 32 | LLM output treated as untrusted — validated/escaped for its sink before driving any action | Required | Q10.7 |
| 33 | LLM agents least-privileged; high-impact/irreversible actions gated by human/out-of-band approval | Required | Q10.7 |
| 34 | Model-produced tool arguments validated and authorized server-side; system prompt/secrets not echoable | Required | Q10.7 |
