# JWT Issue: Signature Validation Mistakes

> [⬅ Domain](../README.md) | [🏠 Dashboard](../README.md)

**OWASP:** API2:2023 | **Status:** 🔄 In Progress

---

## A JWT's security depends entirely on the signature actually being checked. If verification is skipped, bypassed, weak, or confused between algorithms, an attacker can forge arbitrary claims — including privilege escalation to admin — without needing to break any cryptography at all.

---

## 1. Not Verifying the Signature At All

**Issue:** The server decodes the JWT and reads its claims but never actually checks whether the signature is valid. This is uncommon across a full API, but can easily be missed on a single endpoint, especially in server-to-server contexts.

**Development mistake:**
```python
payload = jwt.decode(token, options={'verify_signature': False})
```
This explicitly disables signature checking — the decode call succeeds and returns the claims regardless of whether the signature matches.

**Attack vector:** Authenticate normally, capture the JWT, strip the third segment (the signature) entirely — leaving the trailing dot so the token is still structurally valid — then freely modify any claim (e.g., `admin: 0` → `admin: 1`). Resubmit. If the endpoint doesn't verify signatures, the forged claims are accepted as-is.

**Strategy:** Test this on *every* endpoint that accepts a JWT, not just the login/primary verification endpoint — signature-checking omissions are often inconsistent across an API's surface, and a sensitive endpoint missing this check can have outsized impact even if the main flow is secure.

**The fix:** Always verify using the secret or public key:
```python
payload = jwt.decode(token, self.secret, algorithms="HS256")
```
For server-to-server communication specifically, either verify the signature at every hop, or use an additional authentication factor (e.g., mutual TLS/certificates) rather than relying on an already-verified-upstream assumption.

---

## 2. Downgrading to the "None" Algorithm

### What "none" actually is
In the JWT standard, `"alg": "none"` literally means no signature is required. A JWT header declaring this tells the verifying server: "Don't bother checking math — just trust the payload as-is."

### Why it exists (the original, legitimate intent)
It wasn't invented as a backdoor — it was designed to save processing overhead in internal server-to-server chains:
1. **The front door (API Gateway):** A user sends a properly signed JWT to the company's external-facing gateway. The gateway performs the full cryptographic verification and confirms the user is legitimate.
2. **The internal network:** The gateway then needs to forward that identity to an internal microservice (e.g., a Billing Server) sitting behind the corporate firewall.
3. **The shortcut:** Since the gateway already did the verification, it strips the signature, sets `alg: none`, and passes the token onward. The internal service trusts it purely because it arrived from the already-verified internal gateway — skipping redundant cryptographic work.

### How this becomes a vulnerability
The exposure happens when a developer uses a standard JWT library on a **public-facing** application but never explicitly forbids the `none` algorithm. Some libraries, by default, read the header's `alg` field *before* deciding how to verify — and if it says `none`, they simply return `True` (valid) without performing any check at all. The API then blindly trusts a completely unverifiable, attacker-controlled token.

**Development mistake (pattern that enables this):**
```python
header = jwt.get_unverified_header(token)
signature_algorithm = header['alg']
payload = jwt.decode(token, self.secret, algorithms=signature_algorithm)
```
The algorithm used for verification is taken directly from the attacker-controlled header — if the attacker sets it to `none`, verification is bypassed entirely. (Note: `pyjwt` specifically has protections against this — it raises an exception if a secret is supplied alongside a `none` algorithm — but not every library or implementation enforces this.)

### The attack (step-by-step, pentester perspective)
This is one of the first checks performed on any captured JWT, due to how simple and high-impact it is:
1. **Capture the token:** Log in normally, intercept the legitimate JWT: `Base64(Header).Base64(Payload).Base64(Signature)`.
2. **Modify the header:** Decode it and change `"alg": "HS256"` (or whatever's present) to `"alg": "none"`.
3. **Modify the payload:** Decode it and change the relevant claim, e.g., `"role": "user"` → `"role": "admin"` (or `"admin": 0` → `"admin": 1"`).
4. **Delete the signature:** Remove the entire third segment, but **keep the trailing period** so the token still parses as a structurally valid three-part JWT: `Base64(FakeHeader).Base64(FakePayload).`
5. **Send it:** If the server is vulnerable, it reads `none`, skips verification entirely, and grants access under the forged claims.

**Practical tooling note:** CyberChef's URL-Encoded Base64 option is useful for manually editing header/payload segments during this process.

**Strategy:** Always test for `none` downgrade regardless of what algorithm the legitimate token uses — this is a verification-logic flaw, not a cryptographic weakness, so it's independent of how strong the real signing key/secret is.

### The fix
Modern frameworks patch this by requiring the developer to explicitly declare which algorithm(s) are expected/allowed, rather than trusting the algorithm claimed in the token's own header:
```python
payload = jwt.decode(token, self.secret, algorithms=["HS256", "HS384", "HS512"])
username = payload['username']
flag = self.db_lookup(username, "flag")
```
If the token presents `none` (or any algorithm not in the explicitly allowed list), it's rejected outright — the server decides what's acceptable, not the attacker-controlled header.

---

## 3. Weak Symmetric Secrets

**Issue:** When a symmetric algorithm (HS256, etc.) is used, the entire security of the token rests on the strength/entropy of the shared secret. A weak secret can be recovered via offline cracking.

**Attack vector:**
1. Save the captured JWT to a file (`jwt.txt`).
2. Obtain a JWT-specific secrets wordlist (e.g., `wget https://raw.githubusercontent.com/wallarm/jwt-secrets/master/jwt.secrets.list`).
3. Run an offline cracking tool: `hashcat -m 16500 -a 0 jwt.txt jwt.secrets.list`.
4. If successful, the secret is recovered — now any claim can be modified and a valid signature recalculated, enabling full token forgery (e.g., minting a self-signed admin token).

**Development mistake:** Weak secrets typically result from developers rushing, or copy-pasting example/tutorial code that uses placeholder secrets which are never replaced.

**Strategy:** This is purely an offline attack once a single valid token is captured — no live interaction with the server is needed for the cracking phase, making it low-risk/low-detection reconnaissance with potentially total payoff.

**The fix:** Use a long, high-entropy, randomly generated secret. Since this value is only ever used by software (never typed by a human), there's no usability tradeoff for making it maximally strong.

---

## 4. Signature Algorithm Confusion (Symmetric ↔ Asymmetric)

**Issue:** This is a variant of the `none` downgrade, but specifically exploits confusion between symmetric and asymmetric signing when an API's decode logic allows *both* algorithm families without properly separating how each is verified.

**Development mistake:**
```python
payload = jwt.decode(token, self.secret, algorithms=["HS256", "HS384", "HS512", "RS256", "RS384", "RS512"])
```
The `secret` parameter here is ambiguous — it might be used as an HMAC secret for `HS*` algorithms or as a key material reference for `RS*` algorithms, and the library can be tricked about which role it's playing.

**How the attack works:** If an API normally signs tokens asymmetrically (RS256 — private key signs, public key verifies) but also still accepts HS256, an attacker can:
1. Obtain the public key — these are not considered sensitive and are often openly available (sometimes even embedded as a claim in the JWT itself).
2. Change the token's header algorithm from `RS256` to `HS256`.
3. Sign the forged token using **the public key as if it were an HMAC secret**.
4. Because vulnerable libraries default to using whatever's passed as the "secret" parameter regardless of whether it's actually a public key, the server validates the forged HMAC signature against the known public key — and accepts it.

**Practical example (forging script, used with `pyjwt`):**
```python
import jwt

# Pyjwt normally refuses to use an asymmetric key as an HMAC secret.
# This line overrides that check at runtime for this script only.
jwt.algorithms.HMACAlgorithm.prepare_key = lambda self, key: key.encode() if isinstance(key, str) else key

public_key = "ADD_KEY_HERE"

payload = {
    'username': 'user',
    'admin': 0
}

access_token = jwt.encode(payload, public_key, algorithm="HS256")
print(access_token)
```
Modifying `admin` to `1` and `username` to `admin` before signing produces a forged admin token. (This can also be done without scripting, directly via jwt.io.)

**Strategy:** Whenever a JWT is signed with RS256 (or another asymmetric algorithm), always check whether the API also accepts HS256/other symmetric algorithms on the same decode path — that acceptance is the actual vulnerability, not the presence of RS256 itself. Locate the public key first; it's typically easy to find and is the only material needed for this forgery.

**The fix:** Allowing both algorithm families is fine, but the verification logic must explicitly branch based on the token's declared algorithm family and use the *correct* corresponding key type for each — never a single ambiguous "secret" parameter shared across both:
```python
header = jwt.get_unverified_header(token)
algorithm = header['alg']
payload = ""

if "RS" in algorithm:
    payload = jwt.decode(token, self.public_key, algorithms=["RS256", "RS384", "RS512"])
elif "HS" in algorithm:
    payload = jwt.decode(token, self.secret, algorithms=["HS256", "HS384", "HS512"])

username = payload['username']
flag = self.db_lookup(username, "flag")
```

---

## Quick-Reference: Signature Validation Weakness → Attack

| Weakness | Attack |
|---|---|
| Signature verification disabled/skipped | Direct claim tampering, no forgery needed (#1) |
| `alg: none` not explicitly denied | Signature stripping, claim forgery (#2) |
| Weak/dictionary HMAC secret | Offline brute force → full forgery capability (#3) |
| Both symmetric & asymmetric algorithms accepted via one key parameter | Algorithm confusion using the public key as HMAC secret (#4) |

---

[⬅ Domain](../README.md) | [🏠 Dashboard](../README.md)