# API Token Attacks — Methodology Notes

## Background

Tokens (session tokens, JWTs) are issued after successful authentication and used for all subsequent requests. If the token generation process is weak or predictable, or if the token's integrity isn't properly verified by the provider, an attacker can forge valid tokens without ever knowing a password — effectively bypassing authentication entirely.

## 1. Token Randomness Analysis (Burp Sequencer)

**Definition:** Statistical analysis of a large sample of tokens to determine whether they're generated with sufficient randomness/entropy, or whether they follow a predictable pattern that can be reverse-engineered.

### Methodology

1. Capture an authentication request that returns a token in the response, and send it to Burp Sequencer (right-click → Send to Sequencer).
2. Define the token location in the response: Configure (next to Custom Location) → highlight the token within quotes → OK.
3. Choose a capture method:
   - Live Capture — Sequencer actively sends the request repeatedly and collects live tokens from the target
   - Manual Load — feed it a pre-collected set of tokens (useful for analyzing leaked/sample tokens, e.g., testing against known "bad token" sets)
4. Run the analysis — either let Live Capture collect thousands of samples, or click Analyze Now for a quicker read.
5. Review the Character-Level Analysis — this is where predictability shows up. Look for:
   - Positions in the token that never change across samples (static characters)
   - Positions with limited variation (e.g., only 2 letters + 1 digit in the last 3 characters)
6. If predictable positions are found, calculate the total keyspace of the variable portion. A small keyspace (e.g., aa# format in the last 3 chars) can be brute-forced in a few thousand requests rather than needing to guess the full token length.
7. Use forged/brute-forced tokens against an authenticated endpoint (e.g., /identity/api/v2/user/dashboard) to validate access and harvest further usernames/emails for follow-on attacks.

### Tools

- Burp Suite Sequencer — entropy/randomness analysis engine, built-in character-level breakdown
- Reference dataset for practice: bad_tokens sample set (Hacking APIs GitHub repo) — good for seeing what a genuinely weak token generation process looks like, since well-implemented tokens (like crAPI's by default) won't show this weakness

### Reading Results

- Strong tokens: Sequencer reports high overall entropy, no fixed character positions, large effective keyspace — not brute-forceable in any practical timeframe
- Weak/predictable tokens: fixed segments (e.g., sequential generation where most characters don't change) + a small variable segment → the full token is crackable far faster than its length would suggest
- Don't judge a token as "safe" just because it's long — a 20+ character token with mostly static characters is just as broken as a short one

## 2. JWT Structure & Manual Decoding

**Background:** A JWT has three base64url-encoded segments separated by periods: header.payload.signature. Recognizable by starting with ey (base64 of {") and containing exactly two periods.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.signature...
   [header]                            [payload]                [signature]
```

### Methodology — Manual Decode

1. Split the token on the periods.
2. Base64-decode the header → reveals alg (signing algorithm) and typ:
   ```
   echo <header> | base64 -d
   ```
3. Base64-decode the payload → reveals claims (sub, iat, exp, and often extra provider-added fields like email, role, username — a common information disclosure point even without breaking the signature).
4. Attempt to decode the signature — this will fail/return garbage, since it's the raw HMAC output, not encoded data. The signature is HMAC(base64(header) + "." + base64(payload), secret) — without the secret, it can't be forged.
5. Use JWT.io (debugger) or jwt_tool instead of doing this by hand for faster iteration.

### Tools

- JWT.io — free web-based decoder, good for quick manual inspection
- jwt_tool (JWT_Tool) — CLI Swiss-army knife for JWT analysis and attacks:
  ```
  jwt_tool -t http://target/endpoint -rh "Authorization: Bearer <JWT>" -M pb
  ```
  - `-t` → target URL
  - `-M pb` → playbook scan (automated test suite for common JWT misconfigs)
  - `-M at` → run all tests
  - `-rh` / `-rc` → add request headers / cookies
  - `-pd` → add POST data

### Reading Results

- A disclosed payload containing sensitive fields (password hashes, roles, full email) = information disclosure finding even before any forgery attempt
- alg value is the first thing to check — this decides which attack path below applies

## 3. JWT Attack: Captured/Leaked Token Replay

**Definition:** Simplest JWT attack — no forgery needed. If you obtain another user's valid, unexpired token (recon, leak, logs), just use it directly in your own requests.

### Methodology

1. Obtain the token (leaked in recon, logs, old requests, etc.).
2. Add it as the Authorization: Bearer \<token\> header in your own requests.
3. Check exp claim first — if expired, the provider should reject it (test this too — expired-token enforcement is itself worth checking).

## 4. The "None" Algorithm Attack

**Definition:** If a JWT's header specifies "alg":"none", the provider expects no signature at all — meaning you can edit the payload freely and the provider may still accept it.

### Methodology

1. Decode the JWT, confirm alg is (or can be switched to) none.
2. Edit the payload to whatever you want — e.g., swap the username/email to a likely admin account:
   ```
   {"username":"root","iat":1516239022}
   ```
3. Base64-encode the modified payload (Burp Decoder, or jwt_tool).
4. Remove the signature entirely — keep the trailing period after the payload, but nothing after it.
5. Send the modified token to the provider and check whether it's accepted.

### Tools

- Burp Decoder — manual base64 encode/decode of the edited payload
- jwt_tool -X a — automatically generates a "none" algorithm variant of a captured token

### Reading Results

- If the provider accepts the token and grants access as the claimed user → critical finding, full authentication bypass
- If rejected, pivot to the Algorithm Switch attack below

## 5. The Algorithm Switch Attack (RS256 → HS256)

**Definition:** Exploits providers that don't strictly validate/pin the expected algorithm. If an API uses RS256 (asymmetric — public/private keypair) but doesn't restrict which algorithms it will accept, you may be able to get it to process a token signed with HS256 (symmetric — single shared key) using the provider's own public key as that shared secret.

### Methodology

1. First, try sending the JWT with the signature stripped entirely (same as the none attack test) — sometimes this alone works.
2. If not, decode the header and change alg to none, re-encode, test (as above).
3. If the provider uses RS256, obtain the provider's RS256 public key (via recon — often published, e.g., in a .well-known JWKS endpoint, docs, or config leaks).
4. Save the public key locally (e.g., public-key.pem).
5. Use jwt_tool to sign a new token using the public key as if it were an HS256 secret:
   ```
   jwt_tool TOKEN -X k -pk public-key.pem
   ```
6. Send the resulting token to the provider.

### Tools

- jwt_tool -X k -pk \<keyfile\> — automates the key-confusion forgery

### Reading Results

- If accepted → you can now forge tokens for any user (not just yourself), since you effectively have a valid signing key. Repeat for other discovered usernames/emails to escalate access (e.g., forge an admin token).

## 6. JWT Secret Crack Attack

**Definition:** For HMAC-signed JWTs (HS256/HS512), if the signing secret is weak, you can brute-force/dictionary-attack it offline — no requests sent to the provider, so no rate-limit/lockout risk.

### Methodology

1. Build a wordlist/keyspace. For full brute-force of short secrets, generate all character combinations with Crunch:
   ```
   crunch 5 5 -o crAPIpw.txt
   ```
   (adjust length/charset based on expected secret complexity)
2. Run the crack against the captured token:
   ```
   jwt_tool TOKEN -C -d /wordlist.txt
   ```
   - `-C` → crack mode
   - `-d` → dictionary/wordlist file
3. Read the output — jwt_tool reports either CORRECT key! (secret found) or exhausts the list with no match.
4. Once the secret is known, use the JWT.io debugger (or jwt_tool) to:
   - Paste in the secret
   - Edit any claim (commonly sub/email) to impersonate another user
   - Generate a newly, validly-signed token
5. Use the forged token against a protected endpoint (e.g., GET /identity/api/v2/user/dashboard) to confirm unauthorized access.

### Tools

- jwt_tool -C -d \<wordlist\> — fast, can test ~12 million passwords/minute
- Hashcat — better suited for large-scale/long-running brute-force (GPU-accelerated) when the keyspace is too large for jwt_tool to be practical
- Crunch — generates exhaustive character-combination wordlists for small/short secret spaces

### Reading Results

- CORRECT key! = full compromise of the signing process — you can now mint arbitrary valid tokens for any user/role
- No match = secret is strong enough to resist the wordlist/keyspace tried; escalate to Hashcat with a larger ruleset/keyspace, or pivot to a different attack vector entirely
