# Session Management — Core Concepts (Foundation for Attack Notes)

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

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

### Session Expiry
- **No server-side expiry enforcement** — lifetime checked only client-side (e.g., JS timer) and bypassable by replaying the raw session value.
- **Excessively long idle or absolute timeout** — widens the window an attacker has to use a stolen session.
- **Expired sessions not actually invalidated server-side** — the value still resolves to a valid user even past its intended lifetime.

### Session Termination
- **Logout only clears the client-side cookie, not the server-side session record** — the same session value remains valid if replayed (e.g., via a previously captured request).
- **No centralized session invalidation** — logging out in one place doesn't kill other active sessions/tokens for the same account, allowing persistent access elsewhere.
- **No "log out all devices" / session revocation capability** — a compromised session can't be forcibly killed by the legitimate user, even after they notice something's wrong.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)