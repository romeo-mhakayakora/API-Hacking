# API Credential Attacks — Methodology Notes

## Background

RESTful APIs are stateless, so basic auth (username/password) is typically only used once, at a dedicated login endpoint. A successful login returns a bearer token used for all subsequent requests. This makes the login endpoint the single point of attack for credential-based techniques.

## 1. Password Brute-Force

**Definition:** Many password guesses against one (or a small, fixed set of) known username(s) until one succeeds.

### Methodology

1. Capture a real login request (Burp/browser) to learn the exact JSON structure the API expects:
   `{"email":"a@email.com","password":"FUZZ"}`
2. Check for base64 encoding — some APIs encode credentials before comparing server-side. Note this now; it changes your payload setup later.
3. Establish a baseline. Send one deliberately wrong login manually first. Record its status code and response length — this is what you compare every result against.
4. Build a wordlist.
   - Generic: rockyou.txt (`gzip -d /usr/share/wordlists/rockyou.txt.gz` to unzip on Kali)
   - Targeted: generated from recon data (e.g., leaked details from an excessive data exposure finding) via Mentalist or CUPP
5. Run the attack, injecting the wordlist into the password field, username held constant.
6. Filter out known-failure responses to cut noise.
7. Review remaining results for anomalies (see "Reading Results" below).
8. Manually verify any candidate hit before treating it as valid.

### Tools

**WFuzz**

```
wfuzz -d '{"email":"a@email.com","password":"FUZZ"}' \
  -H 'Content-Type: application/json' \
  -z file,/usr/share/wordlists/rockyou.txt \
  -u http://target/identity/api/auth/login \
  --hc 405
```

- `-d` → POST body
- `-H` → headers (Content-Type: application/json often required or you get a 415)
- `-z file,<path>` → wordlist source, injected at FUZZ
- `--hc` / `--hl` / `--hw` / `--hh` → hide results by code / lines / words / chars

**Burp Suite Intruder** — Sniper attack mode (single payload position)

⚠️ Only brute-force your own lab instances — hammering shared hosted labs can break them for others.

## 2. Password Spraying

**Definition:** The inverse of brute-force — a small list of likely passwords sprayed across a large list of usernames. This avoids per-account lockout thresholds that brute-force would trip.

### Methodology

1. Determine the lockout threshold during recon (e.g., account locks after 10 failed attempts).
2. Build a short password list — fewer entries than the lockout threshold (e.g., 9 if the limit is 10):
   - Pattern-based: QWER!@#$, Password1!, Winter2025!, Spring2025?
   - Org-specific: incorporate company name/year/conventions — Twitter@2025, JPD1976!, Musk@2025
   - Keep each policy-compliant: 8+ chars, upper/lower case, number, symbol
3. Maximize the username list — this is the real lever for success. Pull from:
   - Excessive data exposure vulnerabilities
   - Any other recon source (forums, public directories, etc.)
4. Extract and dedupe usernames/emails:
   ```
   grep -oe "[a-zA-Z0-9._]\+@[a-zA-Z]\+.[a-zA-Z]\+" response.json | sort -u
   ```
5. Configure a Cluster Bomb attack in Burp Intruder — two independent payload sets (usernames × passwords).
6. If auth is base64-encoded, add a payload processing rule: Payloads tab → Add → Encoded → Base64-encode, so each password encodes before sending.
7. Analyze results for anomalies (see below).
8. Manually verify any hit.

### Tools

- Burp Suite Intruder (Cluster Bomb — needs two independent lists)
- grep + regex — harvesting usernames/emails from captured responses
- Mentalist / CUPP — generating the targeted password list

## 3. Reading Results — How to Tell Success from Failure

This applies to both attack types equally.

| Signal | What to look for |
|---|---|
| HTTP status code | Baseline failure might be 400/401/403/405. A deviation (e.g., 200/3xx) is a candidate hit. |
| Response length (chars/words/lines) | Most reliable signal — some APIs lazily return 200 for everything, making status code useless alone. Compare char/word/line counts; a hit's response (containing a token/user object) is almost always longer and structurally different than a generic error body. |
| Response body content | Failures repeat the same generic error message. A hit contains a JWT/bearer token, "success":true, or user data. |
| Response time | Occasionally a valid password takes measurably longer (DB lookup, hash comparison) — not reliable alone, but worth noting if other signals are ambiguous. |

**Workflow:**

1. Baseline a known-bad login manually first.
2. Run the attack.
3. Sort/filter by response length first (status code can be misleading).
4. Open and inspect any outlier's body to confirm.
5. Manually replay the credential pair to confirm it's a genuine hit, not a fluke.

## Quick Comparison

| | Brute-Force | Password Spray |
|---|---|---|
| Goal | Crack one account | Crack any account across many |
| Usernames | 1 fixed | Many (maximize) |
| Passwords | Many (large list) | Few (short, targeted list) |
| Burp attack type | Sniper | Cluster Bomb |
| Beats | No/weak lockout | Lockout policies |
| Primary tool | WFuzz / Intruder | Burp Intruder |

---

## 📚 Further Learning

- [PortSwigger Web Security Academy — Authentication labs](https://portswigger.net/web-security/all-labs#authentication)
- [Hack The Box Academy — Module 80](https://academy.hackthebox.com/app/module/80)
- [TryHackMe — Web Application Pentesting path (Authentication section)](https://tryhackme.com/path/outline/webapppentesting)
- [APIsec University — API Penetration Testing](https://university.apisec.ai/products/api-penetration-testing/categories/2150251352)
