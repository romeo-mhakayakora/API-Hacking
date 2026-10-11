# Password Reset Logic — Full Notes (Lab Walkthrough + Attack Catalog)

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## 1. Core Concept

**Root cause:** A password reset proves identity via "possession of a secret sent out-of-band." The whole model assumes that secret (token, link, code) can only be produced or received by the legitimate user. Break that assumption anywhere — generation, delivery, or validation — and the gate falls.

### Transferable vulnerability checklist
*(applies to email, SMS, security Q&A, even MFA recovery flows)*

| Weak point | What to check | Logic |
|---|---|---|
| Token entropy | Length, charset, randomness source | `mt_rand(100,200)` → only 101 values, numeric only → brute-forceable |
| Token expiration | Dies after use? After a time window? | Long-lived token = longer attack window |
| Validation strength | Security questions guessable/OSINT-able? | Weak secondary factor = bypass |
| Information disclosure | Does "user not found" differ from "reset sent"? | Enables account enumeration |
| Transport | HTTP vs HTTPS | Token sniffable in transit |
| Delivery channel trust | SIM-swap for SMS, email compromise | "Out-of-band" channel isn't always out of reach |

### The attack methodology (reusable, not lab-specific)
1. **Trigger the flow** — request a reset for a known/target account.
2. **Identify the secret's shape** — intercept the token/link with a proxy. Numeric? Short? Sequential? Timestamp-based?
3. **Estimate the keyspace** — small numeric range = trivially brute-forceable. Same logic as a weak SUID binary or a predictable session ID.
4. **Generate candidates** — tool-agnostic (Crunch, Python range, Hydra wordlist, Burp payload generator).
5. **Automate the guesses** — Intruder, ffuf, or a script against the validation endpoint.
6. **Distinguish success from failure** — compare response length/content, not just status code (error pages often still return 200).
7. **Consume the win** — set new password, log in.

> **Pattern to remember:** Weak secret + no rate limiting + distinguishable success response = brute-forceable gate. Same logic reused for OTPs, short API keys, session tokens, PIN-based MFA, sequential object IDs (IDOR).

---

## 2. Worked Example: Predictable Token Brute Force (Lab Walkthrough)

**Vulnerable code:**
```php
$token = mt_rand(100, 200);
$query = $conn->prepare("UPDATE users SET reset_token = ? WHERE email = ?");
$query->bind_param("ss", $token, $email);
$query->execute();
```
The weakness is immediately visible: `mt_rand(100,200)` → only **101 possible values**, numeric only, no mixed characters.

**Steps:**
1. On the target app, go to the password reset page, submit `admin@admin.com`.
2. App responds with a generic "reset link sent" message — the reset link format is `.../reset_password.php?token=123`.
3. Capture the token-validation request in Burp Suite.
4. Send to Intruder, mark the `token` parameter as the injection point.
5. Generate the candidate keyspace with Crunch:
   ```
   crunch 3 3 -o otp.txt -t %%% -s 100 -e 200
   ```
   → generates all 3-digit values from 100–200 (101 lines).
6. Load `otp.txt` as the Intruder payload list, run the attack.
7. **Differentiate success from failure**: valid token responses have a noticeably larger content length than "Invalid token" error responses — sort by response size to spot it.
8. Use the new password to log in and retrieve the flag.

**Note:** Real-world apps typically use 6-digit codes (1,000,000 combinations) rather than 3-digit ones — still brute-forceable without rate limiting, just slower. The lab uses 3 digits to stay practical for Burp Community Edition's throttled Intruder.

---

## 3. Full Attack Catalog

### 1. Predictable / Brute-forceable Token
**How:** Capture token format → generate full keyspace → automate guesses → differentiate response (see lab above).
**Weakness:** Low entropy generation, no rate limiting, distinguishable valid/invalid responses.

### 2. Token Never Expires / Long-Lived Token
**How:** Trigger a reset, save the token, use it hours/days later — still works, becomes a persistent backdoor.
**Weakness:** No TTL enforcement server-side, or TTL checked client-side only.

### 3. Token Not Invalidated After Use
**How:** Use a token once, then replay it — if not marked "spent," it works again (useful if captured earlier via logs, referrer leakage, shared device).
**Weakness:** Validation checks "is this value correct," not "has this value already been consumed."

### 4. Host Header Injection / Reset Link Poisoning
**How:** Modify the `Host` (or `X-Forwarded-Host`) header in the reset request. If the app builds the email link using that header, the victim gets a link pointing to your server. Victim clicks → token lands in your logs → use it on the real site.
**Weakness:** Trusting user-controllable headers to build security-sensitive URLs instead of a hardcoded/whitelisted domain.

### 5. Token Leakage via Referer Header
**How:** The reset page (loaded via `?token=xyz`) pulls in external resources (images, scripts, trackers). Browsers send the full URL — including the token — as `Referer` to those third parties.
**Weakness:** Sensitive data in the URL query string; no `Referrer-Policy` header restricting leakage.

### 6. Password Reset Poisoning via Email Parameter Tampering
**How:** Intercept the reset request — it may carry two email-like parameters (session-identified user + form field). Tamper with which one is used for "send to" vs. "account being reset" (e.g., duplicate `email` parameters) to redirect the token to an attacker-controlled inbox.
**Weakness:** Inconsistent trust between parameters used for delivery vs. account binding — a logic/parsing bug, not crypto.

### 7. Response-Based Account Enumeration
**How:** Submit a reset for a known-valid email, then a made-up one. Compare response time, text ("link sent" vs "not found"), or status codes to build a valid-user list.
**Weakness:** Differential responses leak account existence.

### 8. Weak Security Questions
**How:** OSINT the target (social media, breaches, public records) for answers to common questions, then submit them to bypass identity verification.
**Weakness:** Questions rely on guessable/public info or have a small answer-space.

### 9. SMS-Based Reset — Interception / SIM Swap
**How:** Social-engineer the carrier (or exploit SS7 weaknesses) to port the victim's number, or exploit malware/OS permissions reading SMS on a compromised device.
**Weakness:** The "out-of-band" phone channel is treated as unconditionally secure, but carrier-level identity binding is weak.

### 10. Response Manipulation / Client-Side Trust
**How:** Some apps validate the token/security question entirely client-side (JS returns `true`/`false`, or a hidden `isValid` flag) before allowing the "set new password" step. Intercept and flip the value, or navigate directly to that step.
**Weakness:** Security-critical validation enforced client-side instead of on every server-side request.

### 11. Race Condition on Reset Request
**How:** Fire multiple simultaneous reset/validation requests (Turbo Intruder) to hit shared state before it updates — can bypass lockouts or cause token reuse/overwrite bugs.
**Weakness:** No atomic/thread-safe handling of token creation or attempt-counting (TOCTOU).

### 12. Missing Rate Limiting / CAPTCHA
**How:** Even with decent entropy, unlimited token requests or validation guesses let automation eventually win, or flood the target's inbox/phone (notification DoS).
**Weakness:** No throttling, lockout, or CAPTCHA on sensitive endpoints.

---

## 4. Quick-Reference: Weakness → Attack Map

| Underlying weakness | Attacks it enables |
|---|---|
| Low token entropy | Brute force (#1) |
| No expiration / no single-use | Long-lived replay (#2, #3) |
| Trusting attacker-controlled headers | Host header poisoning (#4) |
| Sensitive data in URL | Referer leakage (#5) |
| Inconsistent parameter trust | Email parameter tampering (#6) |
| Differential responses | Enumeration (#7) |
| Guessable secondary auth | Weak security questions (#8) |
| Insecure out-of-band channel | SIM swap (#9) |
| Client-side enforcement | Response manipulation (#10) |
| No atomicity | Race conditions (#11) |
| No throttling | Any of the above, amplified (#12) |

---

## 5. Defensive Takeaways (for your fix/remediation notes)
- Cryptographically random tokens (≥128 bits), never `mt_rand` or similar weak PRNGs.
- Expire tokens on first use **and** after a short TTL (5–15 min typical).
- Enforce server-side rate-limiting/lockout after N failed attempts.
- Return uniform responses regardless of account existence.
- Always HTTPS; never put tokens in URLs if avoidable (or strip `Referrer-Policy`).
- Hardcode/whitelist the domain used to build reset links — never trust `Host`.
- Re-validate security-critical state server-side on every step, not just client-side.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)