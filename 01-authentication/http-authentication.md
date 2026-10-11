# HTTP Authentication — Vulnerability Reference

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## HTTP Basic Authentication — Vulnerability Reference

### Where It's Used (Context Matters for Targeting)
Basic Auth is favored in environments with **limited processing power** and where **session management/user tracking is unnecessary or handled elsewhere** — it's "simple to implement, simple to manage." Typical targets:
- **Network devices** (routers, switches) — controls access to admin/configuration interfaces.
- **IoT and embedded devices** — low overhead fits constrained hardware.
- **Internal/management interfaces** generally — accessed occasionally for config changes rather than continuous use, so maintaining session state is seen as unnecessary complexity.

**Why this matters offensively:** These are exactly the devices likely to be shipped with default or weak credentials, rarely patched, and often exposed on internal networks (or sometimes externally) without HTTPS. Seeing a Basic Auth prompt is a strong signal you're looking at infrastructure/admin-tier access, not a regular user-facing app — high-value, often under-defended.

### Core Issue
Basic Auth transmits credentials as `base64(username:password)` in the `Authorization` header. Base64 is **encoding, not encryption** — it's trivially reversible. The scheme has no built-in protection against credential guessing, interception, or replay; its security depends entirely on external controls (HTTPS, rate limiting, strong passwords) that are often missing, especially on embedded/network devices.

---

#### 1. Base64 "Encryption" Misconception
**Issue:** Credentials are only encoded, not encrypted. Anyone who sees the header can decode it instantly.
**Attack vector:** Capture the `Authorization: Basic <string>` header (via proxy, network capture, logs, browser history) and decode it directly — no cracking needed.
**Strategy:** Whenever Basic Auth is in use, assume the credential is as exposed as if sent in plaintext. Always check the transport layer (HTTP vs HTTPS) first.

#### 2. Credential Brute-Force
**Issue:** No complexity/rate enforcement on the authentication endpoint; weak credentials are common on admin/device interfaces.
**Attack vector:** Capture a Basic Auth request → send to Intruder → set payload on the base64 blob → use payload processing rules to (1) template the username with each candidate password, then (2) base64-encode the combined string → strip `=` padding from the encoding charset (it collides with base64's own padding) → fire against the endpoint → distinguish success by status code (200) or response difference from the standard 401 challenge.
**Strategy:** Treat the encoded blob as a credential pair to reconstruct per-attempt, not a single opaque token — the attack is really "username:password brute force," just wrapped in an encoding step.

#### 3. Interception Over Unencrypted Transport
**Issue:** No confidentiality guarantee from the scheme itself; relies entirely on HTTPS.
**Attack vector:** Sniff traffic on the network (MITM, ARP spoofing, rogue AP) and decode captured Authorization headers.
**Strategy:** Check for HTTP-only admin panels (common on routers/IoT) — this is often the easiest win, no cracking required.

#### 4. No Session Management / No Lockout
**Issue:** Basic Auth is stateless by design — no session tracking, no built-in lockout after failed attempts.
**Attack vector:** Unlimited automated guesses against the endpoint since there's no session state to throttle or invalidate.
**Strategy:** Always test whether repeated failed attempts trigger any blocking; absence of lockout turns weak-credential risk into a near-certain compromise given enough time.

#### 5. Credential Reuse Across Requests
**Issue:** Once obtained, the same header value authenticates every subsequent request — no token rotation, no expiry tied to activity.
**Attack vector:** A single captured header can be replayed indefinitely until the password is changed.
**Strategy:** Treat a captured Basic Auth header as a long-lived credential, not a session artifact — persistence value is high.

---

#### Quick-Reference: Weakness → Attack

| Weakness | Attack |
|---|---|
| Base64 is reversible, not encrypted | Direct decode of captured header (#1) |
| Weak/common passwords + no rate limiting | Brute force via Intruder (#2) |
| No transport encryption | Network interception (#3) |
| Stateless, no lockout | Unlimited guessing (#4) |
| No token rotation/expiry | Indefinite replay (#5) |

---

## Modern HTTP Authentication Protocols — Vulnerability Reference & Sources

### 1. OAuth 2.0 / OpenID Connect
**Where used:** Social login, "Sign in with Google/GitHub/etc.", API delegated access, SSO across client apps and services.

**Common issues (client-side):**
- **Improper redirect_uri validation** → attacker manipulates `redirect_uri` to steal authorization codes/tokens (CVE-2021-43777, CVE-2022-36087).
- **Implicit flow token leakage** → access token sent via URL fragment, exposed to browser history, referrer headers, open redirects.
- **Weak/missing CSRF protection** (`state` parameter absent or not validated) → account hijacking via forced linking.
- **SSRF via dynamic client registration** → client-supplied data used unsafely server-side.

**Common issues (provider-side):**
- Flawed scope/consent handling, token leakage via proxy pages, insecure transport exposing codes/tokens to MITM (CVE-2022-32217, CVE-2022-32227, CVE-2022-39222, CVE-2023-22492).
- Client secret handling/rotation failures (CVE-2022-31186, CVE-2019-5625).

**Links:**
- https://portswigger.net/web-security/oauth
- https://arxiv.org/pdf/2301.12496 (CVE/CWE-mapped OAuth vulnerability survey)
- https://github.com/PortSwigger/oauth-scan (Burp extension for OAuth/OIDC issues)

---

### 2. JWT (JSON Web Tokens)
**Where used:** Stateless session tokens, API auth, often paired with OAuth/OIDC as the access/ID token format.

**Common issues:**
- **`alg:none`** — library accepts unsigned tokens, trusting attacker-declared algorithm in the header (CVE-2015-2951; resurfaced in Hono, 2026).
- **RS256→HS256 downgrade (algorithm confusion)** — server uses its public RSA key as an HMAC secret, letting attackers forge signatures with the public key (CVE-2016-10555, CVE-2019-20933).
- **Weak/brute-forceable HMAC secret** — crackable via `hashcat -m 16500` against a wordlist.
- **`kid` / `jwk` / `jku` header injection** — attacker points the server to a key it controls, or injects SQLi via `kid` if keys are looked up in a DB.
- **Unverified signature entirely** — server decodes but never checks the signature.

**Links:**
- https://portswigger.net/web-security/jwt
- https://github.com/PortSwigger/jwt-editor (Burp extension automating these attacks)
- https://dev.to/roxdavirox/jwt-vulnerabilities-common-attacks-and-how-to-prevent-them-372k
- jwt_tool: https://github.com/ticarpi/jwt_tool (brute-force, alg confusion, tamper/replay)

---

### 3. WebAuthn / FIDO2 / Passkeys
**Where used:** Phishing-resistant MFA/passwordless login, increasingly default for consumer and enterprise auth as of 2026.

**Common issues:**
- **"Pass-the-Passkey" attacks (2026)** — 20+ techniques across Windows, Entra ID, browsers, password managers; includes assertion replay via Windows event-log disclosure (CVE-2026-34348) and browser/extension hooking of `navigator.credentials.get()`/`.create()`.
- **Downgrade/fallback abuse** — forcing authentication to fall back to password when WebAuthn unavailable, bypassing the strong factor entirely (AiTM kits).
- **Malicious browser extensions hijacking WebAuthn calls** — "Passkeys Pwned" (DEF CON 33) shows attacker-controlled key pairs substituted during registration.
- **Synced passkey cloud-account risk** — security inherited from the cloud account/recovery flow protecting the sync, not the cryptography itself.

**Links:**
- https://cybersecuritynews.com/pass-the-passkey-attacks/
- https://thehackernews.com/2025/10/how-attackers-bypass-synced-passkeys.html
- https://labs.sqrx.com/passkeys-pwned-turning-webauth-against-itself-0dbddb7ade1a
- https://arxiv.org/pdf/2604.20826 (FIDO2 attack vector analysis)

---

### 4. HTTP Message Signatures (RFC 9421) — emerging
**Where used:** Signing API requests (replacing/complementing OAuth bearer tokens for non-browser/agent traffic); underlies newer proposals like the **HTTP Agent Profile (HAP)** for authenticating LLM/bot traffic.

**What to know as an attacker:** Still early/low real-world deployment, so tooling and reported-CVE history is thin — but the attack surface to watch is signature-base construction errors (which headers/components are actually covered) and replay if no freshness/nonce binding is enforced.

**Links:**
- https://www.ietf.org/ietf-ftp/internet-drafts/draft-dhir-http-agent-profile-00.txt
- https://datatracker.ietf.org/doc/draft-schinazi-httpbis-transport-auth/07/ (related: non-probeable cryptographic HTTP auth proposal)

---

### 5. Basic/Digest Auth → OAuth migration (legacy deprecation, relevant for enumeration)
Major providers are actively killing Basic Auth in mail/legacy protocols: Microsoft enforced full rejection by April 30, 2026; Google eliminated it March 14, 2026. Seeing Basic Auth still active anywhere in 2026 is now itself a strong signal of an outdated/neglected system.

**Link:**
- https://www.getmailbird.com/email-authentication-standards-oauth-guide/

---

### General reference / ongoing source
- https://github.com/afwu/awesome-web-security — curated running list of web auth/security writeups including OAuth, JWT, and related CVE writeups.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)