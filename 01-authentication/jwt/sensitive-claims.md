# JWT Issue: Sensitive Information Disclosure via Claims

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## Issue
JWTs are not encrypted by default — the entire payload is Base64Url-encoded (not secret) and sent client-side with every request. Developers who are used to cookie-based server-side session storage (e.g., PHP's `$_SESSION['var'] = data`, which stays server-side and is never exposed to the client) sometimes carry that same habit into JWT claims — stuffing sensitive data directly into the payload, forgetting that **a JWT claim is visible to anyone holding the token**, unlike a server-side session variable.

**Real-world examples of what leaks this way:**
- Password hashes, or worse, clear-text passwords stored as a claim.
- Internal network details — private IPs, internal hostnames of the authentication server — embedded for convenience during development/debugging and left in.

**Worked example (lab):**
```python
payload = {
    "username": username,
    "password": password,
    "admin": 0,
    "flag": "[redacted]"
}
access_token = jwt.encode(payload, self.secret, algorithm="HS256")
```
Here, both the password and a sensitive flag value are placed directly into the signed-but-unencrypted payload.

**Attack vector:** Authenticate normally to obtain a valid JWT, then simply decode the payload (manually via Base64, or via jwt.io/Burp) — no cracking, no forgery, no exploitation of the signature required. The sensitive data is handed over voluntarily by design flaw, not stolen.

**Strategy:** Decoding and reading the claims should be the *very first* thing done with any captured JWT during an assessment, before attempting any signature attacks — it's free information and costs nothing to check.

**The fix (correct pattern):** Keep only non-sensitive identifiers (like `username`) in the claims. Sensitive values should remain server-side, looked up on demand using a claim from a *verified* token as the key:
```python
payload = jwt.decode(token, self.secret, algorithms="HS256")
username = payload['username']
flag = self.db_lookup(username, "flag")
```
This preserves the "claims aren't secret" property of JWTs while keeping actual sensitive data out of client reach entirely.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)