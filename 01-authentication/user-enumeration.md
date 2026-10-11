# User Enumeration via Verbose Errors & Authentication Enumeration — Notes

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## Core Issue
Verbose error messages leak information the application never intended to expose. The app assumes error detail only helps *developers* debug — it doesn't account for the same detail helping an *attacker* map the system. This is the same root cause pattern as before: **a violated assumption about who reads the output.**

Authentication enumeration broadly is the process of removing uncertainty — turning "I don't know if this account/password policy/rule exists" into "I know exactly what I'm working with" — before investing effort into brute force or exploitation.

---

## 1. Username/Email Enumeration via Differential Error Messages
**Issue:** The application returns a different error for "account doesn't exist" vs. "account exists, wrong credential" (e.g., "Email does not exist" vs. "Invalid password").
**Attack vector:** Submit a login/forgot-password request with a candidate email and a throwaway password; read the specific error text returned. Automate this against a wordlist (e.g., a Python script posting to the login/function endpoint and checking the JSON `message` field) to separate valid from invalid accounts at scale.
**Strategy:** Any flow that validates identity in two steps (does the account exist → is the credential correct) is a candidate for this. Always compare responses for a known-bad input against a guessed input — text, not just status code, is often where the leak lives.

## 2. SQL Injection-Induced Errors
**Issue:** Malformed input (e.g., a single quote) isn't sanitized before reaching the database layer, and the raw DB error is returned to the client.
**Attack vector:** Inject a `'` or similar into a login/search field; read the resulting stack trace or DB error for schema details (table names, column names, DB engine/version).
**Strategy:** Errors here aren't just a sign of SQLi risk — they're a reconnaissance tool in their own right, revealing structure to inform later injection payloads.

## 3. File Inclusion / Path Traversal-Induced Errors
**Issue:** File path handling isn't sanitized; attempting to traverse directories (`../../`) throws an error containing the real filesystem path.
**Attack vector:** Submit traversal sequences in any file-referencing parameter and read the error for internal path disclosure (web root, app structure, OS type).
**Strategy:** Even a "failed" traversal attempt is useful — the failure message itself is the payload's actual goal here, not successful file access.

## 4. Form/Parameter Manipulation-Induced Errors
**Issue:** Hidden fields, parameter types, or validation logic aren't designed to fail safely — tampering trips validation errors that describe the expected format or internal logic.
**Attack vector:** Modify hidden form fields, change parameter types/values, or omit expected fields; read the validation error for backend structure or logic hints.
**Strategy:** Don't just test visible inputs — hidden/disabled fields are often less hardened because the developer assumed users wouldn't reach them directly.

## 5. Application Fuzzing for Informative Errors
**Issue:** The app doesn't uniformly handle unexpected input across all endpoints/parameters.
**Attack vector:** Automate bombardment of varied payloads (Burp Intruder or similar) across inputs to surface which ones provoke verbose/informative errors.
**Strategy:** Treat this as broad reconnaissance before targeted exploitation — fuzzing finds *where* the verbose-error weakness exists before you decide *how* to exploit it.

## 6. Password Policy Disclosure
**Issue:** When a submitted password fails validation, the error message echoes back the exact policy requirements (e.g., "must contain one uppercase letter, one number, one symbol") rather than a generic failure.
**Attack vector:** Submit a deliberately weak/invalid password during registration or reset, read the returned policy description, and use it to generate a targeted wordlist/mask that satisfies the known constraints — shrinking the brute-force keyspace instead of guessing blind.
**Strategy:** Password policy is normally thought of as a defense; here it becomes offensive intel — it tells you the *shape* of the password space you need to search, which is a major efficiency gain for brute force.

---

## The Compounding Logic: Why Enumeration Precedes Brute Force
Authentication enumeration isn't an attack by itself — it's the **reconnaissance phase that makes every later attack cheaper**:
- Valid usernames (#1) → brute force only needs to guess passwords, not username+password pairs simultaneously.
- Password policy (#6) → brute force wordlist/mask can be constrained to the known format instead of guessing blind.
- Account lockout behavior (inferred from testing) → tells you how many guesses you can safely make before detection/lockout.

This mirrors the earlier framework: **enumerate → model → hypothesize → test → pivot.** Verbose errors are one of the richest "enumerate" sources because the system is telling you its internal state directly, just not on purpose.

---

## Quick-Reference: Weakness → Attack

| Weakness | Attack |
|---|---|
| Differential error text (exists vs. not) | Username/email enumeration (#1) |
| Unsanitized input reaching DB | SQLi-induced schema disclosure (#2) |
| Unsanitized file paths | Path traversal-induced path disclosure (#3) |
| Unsafe validation of hidden/manipulated fields | Logic/structure disclosure (#4) |
| Non-uniform error handling across inputs | Fuzzing reveals weak endpoints (#5) |
| Policy echoed in failure message | Targeted wordlist generation (#6) |

## Defensive Takeaways
- Generic, uniform error messages regardless of failure reason ("Invalid email or password").
- Never return raw DB/stack trace errors to the client — log server-side, show a generic message client-side.
- Rate-limit and monitor repeated failed attempts, including those probing for verbose errors (SQLi payloads, traversal sequences).
- State password policy at registration/input time (so legitimate users know it), but don't re-disclose the exact regex/rule on failure — use a generic "doesn't meet requirements" message with a link to policy docs shown once, upfront.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)