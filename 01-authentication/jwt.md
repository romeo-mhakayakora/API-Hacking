# JSON Web Tokens (JWT) & API Session Management — Full Notes

> [⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## Why APIs Changed Session Management
APIs are built to serve multiple interfaces (web, mobile, third-party integrations) from a single centralized server-side logic layer. Cookie-based sessions are inherently browser-tied, so they don't generalize well across interfaces that aren't browsers. This is why **token-based session management** — and specifically JWTs — became the standard for API authentication: the token is self-contained and interface-agnostic, carried manually via the `Authorization: Bearer` header rather than relying on browser cookie mechanics.

---

## JWT Structure
A JWT has three Base64Url-encoded components separated by dots: `Header.Payload.Signature`

- **Header** — declares the token type (JWT) and the signing algorithm (`alg`).
- **Payload** — the claims (the actual data: user ID, role, permissions, etc.). Readable by anyone who has the token — Base64 is encoding, not encryption.
- **Signature** — proves the header+payload haven't been tampered with since issuance. This is where the actual security lives.

**Core principle — Integrity, not Secrecy:** A standard (unencrypted) JWT is not a secret container. Anyone who captures it can decode and read the payload. What the signature protects against is *tampering* — like a tamper-evident seal, not a locked box. If an attacker edits the payload (e.g., `"role": "user"` → `"role": "admin"`) and reattaches it without a valid signature, the math breaks and the server should reject it — **should**, being the operative word, since whether this actually happens correctly is exactly what gets attacked.

---

## Why JWTs Enable Stateless, Centralized Auth
Traditional session IDs require the server to query a database on every request ("is this session still valid?"). A signed JWT removes that lookup entirely — if the signature verifies, the server trusts the claims inside without checking anywhere else. This is what makes JWTs ideal for SSO: a central Auth Server (holding the private/shared key) issues the token once, and every downstream application (HR portal, finance app, email) independently verifies the signature using only a public key or shared secret — no need to call back to the Auth Server per request.

**Why this matters offensively:** Because every downstream app trusts the signature blindly rather than checking a central database, forging or bypassing that signature compromises *every* application that trusts it simultaneously — one successful forgery, network-wide impact.

---

## Signing Algorithms — Issue, Attack Vector, Strategy (per algorithm)

### 1. `alg: none`
**Issue:** The application accepts tokens with no signature at all — a developer misconfiguration allowing unsigned claims to be trusted.
**Attack vector:** Decode the payload, change any claim you want (e.g., `"username": "user"` → `"username": "admin"`), strip the signature section entirely, and submit the token. If the server doesn't explicitly reject `alg: none`, it's granted as valid.
**Strategy:** Always test this first — it's the cheapest possible win and requires no cracking or key material at all.

### 2. Symmetric Signing (HS256 — "shared password" model)
**Issue:** A single shared secret is used to both create and verify every token. Any system that needs to verify tokens must hold the exact same secret — meaning the secret is only as protected as the weakest server that stores it.
**Attack vector (Offline Brute Force):** Capture a token, save it locally, and run an offline cracking tool (Hashcat, John the Ripper) against a wordlist of common/weak secrets, checking which candidate secret reproduces the exact signature. If the developer used a weak secret (`secret123`, `company2023`), it cracks quickly — once known, you can forge arbitrary admin tokens.
**Strategy:** If a token uses HS256, always attempt an offline brute force before anything else — it's low-cost reconnaissance with potentially total-compromise payoff, and it doesn't touch the live application (no detection risk during cracking).

### 3. Asymmetric Signing (RS256 — "ID card" model)
**Issue:** Uses a private/public key pair — the private key signs, the public key verifies. Theoretically much safer since compromising a verifying server (which only holds the public key) doesn't let an attacker forge new tokens.
**Attack vector (Algorithm Confusion):** If the backend is misconfigured to accept *either* algorithm type, change the header from `RS256` to `HS256`, then use the (often publicly available) RSA public key as the "shared secret" for an HMAC signature. A poorly coded verifier treats the public key as a symmetric secret, validates the forged signature, and accepts the token.
**Strategy:** Always check if the API accepts multiple algorithms rather than strictly enforcing one. Locate the public key (often published openly, e.g., via a JWKS endpoint or company documentation) before attempting this — it's required material for the forgery.

### General Rule: Identify the Algorithm First
This is the **first step** in any JWT assessment, because it directly determines which attack is even mathematically possible:
1. Decode the header (`jwt.io`, Burp Suite, or manual Base64 decode).
2. Read the `alg` field.
3. Route your attack: `none` → strip signature; `HS256` → offline brute force; `RS256` → test algorithm confusion using the public key.

---

## JWE — Encrypted Tokens (Confidentiality Layer)

**Why it exists:** Standard signed JWTs (JWS) protect integrity but not secrecy — payloads are plainly readable. If a payload needs to carry genuinely sensitive data (SSNs, addresses, internal routing data), a signed-only JWT exposes it to anyone who intercepts it or inspects their own browser storage. JWE solves this by encrypting the payload into ciphertext.

**Key role reversal vs. signing — important to note precisely:**
| | Signing (JWS) — Integrity | Encrypting (JWE) — Secrecy |
|---|---|---|
| Who uses the Private Key | Auth Server (to **sign**) | App (to **decrypt**) |
| Who uses the Public Key | App (to **verify**) | Auth Server (to **lock/encrypt**) |

Asymmetric encryption works like a mailbox slot: anyone (the Auth Server) can push data in using the public key, but only the holder of the private key (the specific app) can open it and read the contents.

### Issues, Attack Vectors, Strategy for JWE

**1. Symmetric Encryption (shared key)**
**Issue:** One shared secret both encrypts and decrypts — same single-point-of-failure risk as symmetric signing.
**Attack vector:** If the shared key is compromised from any server holding it, an attacker can decrypt every token on the network.
**Strategy:** Same logic as HS256 — treat the shared key as a high-value target; a leak anywhere compromises everywhere.

**2. Asymmetric Encryption (public/private key pair)**
**Issue:** More resilient in theory — compromising a verifying/encrypting party only yields the public key, which cannot decrypt anything.
**Attack vector:** To actually decrypt, an attacker needs the private key held by the specific receiving application — a much higher bar than symmetric key compromise.

**3. Testing a JWE (practical pentester approach)**
- **You cannot read the payload directly** — it's ciphertext, so standard decode-and-tamper doesn't work.
- **Hunt for key leaks** — search front-end JavaScript bundles and public/leaked GitHub repositories for hardcoded symmetric decryption keys; developers sometimes accidentally ship them client-side.
- **Downgrade attacks** — attempt to strip encryption entirely, reverting the token to a plain signed JWT or even an unsigned (`alg: none`) token, to see if the server will still accept the downgraded, unencrypted version out of implementation confusion.

---

## Quick-Reference: JWT/JWE Weakness → Attack

| Weakness | Attack |
|---|---|
| `alg: none` accepted | Signature stripping, arbitrary claim forgery |
| Weak/guessable HMAC secret (HS256) | Offline brute force (Hashcat/John) → full token forgery |
| Backend accepts multiple algorithms | Algorithm confusion (RS256→HS256 using public key) |
| Sensitive data in unencrypted payload | Direct read via Base64 decode (no attack needed) |
| Hardcoded JWE decryption key in front-end code/repos | Key leak → full payload decryption |
| Server accepts downgraded (unencrypted/unsigned) tokens | Downgrade attack bypassing JWE/JWS entirely |
| No central key compromise containment (symmetric) | Single leaked key compromises every token network-wide |

## Defensive Takeaways
- Never implement or allow `alg: none` server-side — reject it explicitly, don't just fail to check for it.
- Enforce a single expected algorithm per key type server-side; never trust the `alg` field from the token itself to decide verification method.
- Use long, high-entropy, randomly generated secrets for HS256 — never dictionary words or predictable strings.
- Prefer RS256 (or similar asymmetric signing) for any multi-service/SSO architecture, since it contains the blast radius of a single compromised verifying service.
- Never hardcode symmetric JWE keys anywhere reachable by the client (JS bundles, public repos).
- Treat algorithm/key identification as the mandatory first step of any JWT-focused assessment — it determines the entire attack path.

---

[⬅ Domain](./README.md) | [🏠 Dashboard](../README.md)