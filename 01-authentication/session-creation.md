# Session Creation — Full Issue Breakdown

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## The IAAA Model (Foundation Before the Lifecycle)

Authentication and authorization sound similar and are frequently confused, but they're distinct controls that map onto different stages of session management. The full model has four parts:

### Identification
The user *claims* an identity — typically by submitting a username or email. No proof yet, just a claim.

### Authentication
The user *proves* that claimed identity — e.g., by supplying the correct password. Once the application validates this, **session creation** kicks in. Authentication is the gate that session creation sits behind.

### Authorisation
Once identity is established, authorisation determines *what that identity is allowed to do* — e.g., all users can view data, but only some can modify it. This is where **session tracking** becomes critical: on every request, the server must not just confirm "who is this session" but also "what is this session permitted to do." A tracking flaw that resolves identity but mishandles permissions is an authorisation failure riding on a session bug.

### Accountability
A record of what each session actually did. Every request should be logged alongside the session that made it, so that in the event of an incident, actions can be reconstructed and attributed. Without this, even a well-secured session system leaves no trail to investigate abuse after the fact.

### Why this matters for session management specifically
- **Authentication → Session Creation**: proving identity is the trigger that creates the session in the first place.
- **Authorisation → Session Tracking**: every tracked request must be checked not just for "whose session is this" but "is this session allowed to do this."
- **Accountability → Logging**: requests and their associated session values must be logged, so incidents can be pieced together after the fact.

This distinction matters offensively: a **broken authentication** issue (weak login, credential stuffing) is a different bug class from a **broken authorisation** issue (a valid, authenticated session that can act beyond its permitted scope, e.g., IDOR, privilege escalation via session reuse). Many session attacks exploit the gap between the two — the session proves *who* you are without re-verifying *what you're allowed to do* on every action.

---

## Why Sessions Exist
HTTP is stateless — it has no memory between requests. Sessions are the mechanism that fakes continuity: instead of resending credentials on every request, the server issues a session identifier that the client resends to prove "I'm still the same authenticated user." Every session attack ultimately targets this illusion — convincing the server that an attacker *is* the legitimate continuity.

## The Lifecycle (4 stages — each is a distinct attack surface)

### 1. Session Creation
**What happens:** A session value is generated and handed to the client — sometimes *before* login (pre-auth tracking), always after successful authentication.
**Why it matters offensively:** How the session value is generated (entropy, predictability), transmitted, and stored determines whether it can be guessed, intercepted, or fixed by an attacker before the victim even authenticates.
**Note:** A pre-auth session that gets reused post-auth (rather than regenerated) is a specific weakness to watch for — this is the root of session fixation.

### 2. Session Tracking
**What happens:** On every subsequent request, the server extracts the session value and performs a server-side lookup to resolve identity and permissions.
**Why it matters offensively:** This is the "am I still me" check. Any weakness here — predictable values, insufficient binding (e.g., not tied to IP/device/fingerprint), insecure transmission — lets an attacker hijack or impersonate an active session without ever touching credentials.

### 3. Session Expiry
**What happens:** Session values should carry a lifetime. Once expired, resubmitting an old value should be rejected, forcing re-authentication.
**Why it matters offensively:** No expiry (or an excessively long one) means a captured session value stays useful to an attacker indefinitely — this is the "no TTL" weakness pattern, same root cause as the token-expiry issue in the password reset notes.

### 4. Session Termination
**What happens:** A user-initiated logout should kill the session immediately, even if its natural lifetime hasn't expired yet.
**Why it matters offensively:** Expiry and termination are *not* the same control. A session that's logically still "within its lifetime" but should have been invalidated by an explicit logout is a distinct failure mode — if termination isn't enforced server-side, a captured session value remains valid even after the legitimate user has logged out, giving an attacker persistent access.

---

## Transferable Pattern
This maps directly onto the general offensive framework: **a session is just another secret the system trusts.** Same four questions apply as to any trust object:
- **Generation** — is it predictable? (→ creation-stage attacks)
- **Transmission/validation** — can it be intercepted or doesn't get re-checked properly? (→ tracking-stage attacks)
- **Lifetime** — does it expire properly? (→ expiry-stage attacks)
- **Revocation** — can it be killed on demand, and is that actually enforced? (→ termination-stage attacks)

Any weakness at any one stage breaks the whole session security model, independent of how strong the other three are.

---

## What Can Go Wrong at Each Stage

### Session Creation
- **Predictable/low-entropy session identifiers** — sequential, timestamp-based, or short values that can be guessed or brute-forced.
- **Session fixation** — a pre-auth session ID is accepted and continues being valid after login instead of being regenerated, letting an attacker plant a known session value on a victim.
- **Session ID exposed at issuance** — sent over HTTP, logged, or placed in the URL where it can leak via Referer headers, browser history, or server logs.
- **Insufficient randomness source** — using a weak PRNG (e.g., `mt_rand`, time-seeded generators) instead of a cryptographically secure generator.

### Session Tracking
- **Session ID sent insecurely** — transmitted over HTTP, or without the `Secure`/`HttpOnly`/`SameSite` cookie flags, exposing it to interception or client-side script theft (XSS).
- **No binding to client context** — session not tied to IP, user-agent, or device fingerprint, making a stolen value fully portable to an attacker's machine.
- **Session puzzling/confusion** — application mixes session variables across different authentication contexts, letting an attacker reach a state meant for a different flow.
- **Predictable tracking tokens reused across requests without re-validation** — allows long-term hijacking once captured.
- **Authorisation not re-checked per request** — session confirms identity but the app fails to verify permissions on each action, enabling IDOR/privilege escalation through a perfectly valid session.

### Session Expiry
- **No server-side expiry enforcement** — lifetime checked only client-side (e.g., JS timer) and bypassable by replaying the raw session value.
- **Excessively long idle or absolute timeout** — widens the window an attacker has to use a stolen session.
- **Expired sessions not actually invalidated server-side** — the value still resolves to a valid user even past its intended lifetime.

### Session Termination
- **Logout only clears the client-side cookie, not the server-side session record** — the same session value remains valid if replayed (e.g., via a previously captured request).
- **No centralized session invalidation** — logging out in one place doesn't kill other active sessions/tokens for the same account, allowing persistent access elsewhere.
- **No "log out all devices" / session revocation capability** — a compromised session can't be forcibly killed by the legitimate user, even after they notice something's wrong.

---

## Cookies vs. Tokens — Session Management

### Core Distinction: Who Controls Attachment?
The single most important difference between cookies and tokens isn't where they're stored — it's **who decides to send them on a given request**:
- **Cookies** → the browser decides, automatically, based on domain and cookie attributes. No client-side code required.
- **Tokens (e.g., JWT in LocalStorage)** → JavaScript decides, manually pulling the value from storage and attaching it as a header (commonly `Authorization: Bearer <token>`).

This single difference is the root cause of which attack class each is exposed to.

---

## Cookie-Based Session Management

**Mechanism:** Server sends `Set-Cookie: session=12345;` in a response. The browser stores it and automatically attaches it to future requests matching the cookie's domain/attributes — no app code involved.

**Security-relevant attributes:**
| Attribute | What it does |
|---|---|
| `Secure` | Cookie only sent over verified HTTPS; dropped on HTTP or cert errors |
| `HttpOnly` | Blocks client-side JS from reading the cookie value |
| `Expire` | Sets cookie lifetime; browser removes it after |
| `SameSite` | Controls whether the cookie is sent on cross-site requests |

**Issue:** Because the browser attaches the cookie automatically to *any* request to the matching domain — regardless of which site initiated that request — an attacker's site can trigger a request that silently carries the victim's valid session.

**Worked example (CSRF):**
> You are logged into `your-bank.com`. You visit a hacker's site, `evil-site.com`, which has a hidden script submitting a POST request to `your-bank.com/transfer-money`. Because your browser sees a request going to `your-bank.com`, it automatically says "I have a cookie for that!" and attaches your login cookie. The bank thinks you made the request, and the money is stolen.

**Attack vector (CSRF):** Attacker hosts a page with a hidden form/script that submits a request to the target domain (e.g., a funds-transfer endpoint). Victim's browser, seeing the request is headed to a domain it holds a cookie for, attaches the session cookie automatically. The target server processes it as a legitimate authenticated request.
**Strategy:** When you see a cookie-based session, check for missing/weak CSRF protections immediately — specifically `SameSite` attribute and anti-CSRF tokens. These are the two controls standing between "browser does this automatically" and actual exploitation.

**Mitigation — SameSite (how it stops the example above):**
If the bank's auth cookie has a `SameSite` rule applied, the browser recognizes the request is heading to `your-bank.com` but was *initiated* by `evil-site.com` — not the same site — and strips the cookie before sending. The bank receives the transfer command with no credentials attached, and rejects it.

- `Strict` — cookie sent only on same-site navigation/requests (maximum security, but breaks cross-site link-click login persistence — e.g., a friend texts you a bank link, you click it, you're not logged in).
- `Lax` (modern default) — blocks cookie on cross-site background requests (forms, AJAX, images) but allows it on top-level navigation (clicking a link). Stops CSRF while preserving UX — the friend's link still logs you in.
- `None` — cookie sent everywhere, cross-site included; requires `Secure` flag; used deliberately for embeddable widgets (payment forms, video embeds, "Like" buttons).

**How to check if SameSite is enabled (practical testing):**
1. **Browser DevTools:** F12 → Application (Chrome/Edge) or Storage (Firefox) tab → Cookies → inspect the `SameSite` column. `Lax`/`Strict` = explicitly set; blank = not set by developer (though modern Chrome defaults blank to `Lax` — still flag as a finding since relying on browser defaults isn't robust practice).
2. **Raw HTTP traffic (pentester method):** Capture the login response in Burp Suite or DevTools Network tab, inspect the `Set-Cookie` header directly:
   - Missing: `Set-Cookie: session_id=12345abcd; Secure; HttpOnly`
   - Present: `Set-Cookie: session_id=12345abcd; Secure; HttpOnly; SameSite=Lax`

**Reporting framing:** Finding = "Missing SameSite Attribute on Session Cookie." Risk = older browsers may still permit CSRF; relying on modern-browser defaults isn't a robust control. Recommendation = explicitly set `SameSite=Lax` or `Strict` on all sensitive session/auth cookies.

**Issue (secondary):** Cookies are domain-locked, making them awkward in decentralized/multi-domain architectures.

---

## Token-Based Session Management

**Mechanism:** After authentication, the server returns a token (commonly a JWT) in the response body. Client-side JS stores it (typically LocalStorage/SessionStorage) and must manually attach it to every subsequent request as a header. Nothing happens automatically — if the JS doesn't do it, the token isn't sent.

**Issue:** Because the token lives in LocalStorage and is accessible to any JS running on the page, it's only as safe as the page's defenses against malicious script execution.

**Worked example (XSS → token theft):**
> If a hacker finds a way to inject malicious JavaScript into your actual web application — for example, by leaving a comment containing a malicious script — that script runs in the victim's browser. Because the token is accessible via JavaScript (`localStorage.getItem('token')`), the hacker's injected script simply reads the token and sends it to their own server. They have now stolen your identity.
> (If the value were instead an `HttpOnly` cookie, JavaScript would be physically blocked from reading it, preventing this specific theft method.)

**Attack vector (XSS → token theft):** If an attacker can inject JavaScript into the application (e.g., via a stored XSS in a comment field), that script runs with full access to `localStorage.getItem('token')` and can exfiltrate it to an attacker-controlled server — full identity theft, no cookie/CSRF mechanics involved.
**Strategy:** When you see tokens stored in LocalStorage, immediately pivot your assessment toward **XSS**, not CSRF — that's the actual exposure here.

**Why tokens resist CSRF:**
> If a hacker on `evil-site.com` tries to force your browser to make a request to `your-bank.com`, the browser will send the request, but it will not attach the token. The hacker's website cannot read your LocalStorage due to the Same-Origin Policy. The bank rejects the request as unauthorized.

**Benefit:** Works well in decentralized/multi-domain apps since the token is JS-managed and can be self-contained (e.g., a JWT carrying its own claims), rather than being tied to a single cookie domain.
**Drawback:** No automatic browser-enforced protections exist for tokens (no `Secure`, `HttpOnly`, `SameSite` equivalents) — safeguarding against disclosure is entirely the application's responsibility; there's no standard forcing consistent practice ("the wild west").

---

## Summary Cheat Sheet

| Storage Method | Who attaches it to requests? | Vulnerable to CSRF? | Vulnerable to token theft via XSS? |
|---|---|---|---|
| Cookies | Browser (automatically) | Yes (unless `SameSite` set) | No (if `HttpOnly` set) |
| Tokens (LocalStorage) | JavaScript (manually, via headers) | No | Yes |

**Vulnerability-assessment triage rule:** Session in a cookie → check `SameSite` + anti-CSRF tokens. Session in LocalStorage/JS-managed token → check for XSS.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)