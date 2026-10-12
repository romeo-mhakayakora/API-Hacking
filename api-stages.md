# API Attack Stages — Lab Map

> [⬅ Back to API Hacking Dashboard](README.md)

**OWASP:** API Top 10 (2023) | **Status:** 🔄 In Progress

This page maps each API attack category to the TryHackMe labs that exercise it. Use it to plan your path: start with the stage that matches the vulnerability you're hunting, then work through the labs in order.

---

## 📋 Stage Map

| API Stage | OWASP API Top 10 | TryHackMe Lab | Notes |
|-----------|:----------------:|---------|-------|
| **Authentication Attacks** | API2:2023 | [Session Management](./01-authentication/session-creation.md) | Session fixation, predictable IDs, exposure, weak PRNG |
| **Authentication Attacks** | API2:2023 | [User Enumeration](./01-authentication/user-enumeration.md) | Verbose errors, account enumeration, policy disclosure |
| **Authentication Attacks** | API2:2023 | [HTTP Authentication](./01-authentication/http-authentication.md) | Basic Auth, OAuth/OIDC, JWT, WebAuthn/Passkeys |
| **Authentication Attacks** | API2:2023 | [Password Reset Logic](./01-authentication/password-reset-logic.md) | Token entropy, expiry, host header poisoning, enumeration |
| **Authentication Attacks** | API2:2023 | [Token Attacks](./01-authentication/token-attacks.md) | Sequencer, JWT alg confusion, secret cracking, replay |
| **Authentication Attacks** | API2:2023 | [Authentication Attacks](./01-authentication/authentication-attacks.md) | Brute-force, spraying, credential stuffing, MFA bypass |
| **Access Control** | API1:2023 | [BOLA / IDOR](./02-access-control/bola.md) | Object-level authorization bypass |
| **Access Control** | API3:2023 | [BOPLA](./02-access-control/bopla.md) | Mass assignment / parameter tampering |
| **Access Control** | API5:2023 | [BFLA](./02-access-control/bfla.md) | Function-level authorization bypass |
| **Access Control** | API2:2023 | [Session Management](./02-access-control/session-management.md) | Session lifecycle, fixation, exposure, IAAA model |
| **Access Control** | API2:2023 | [Session Tracking, Expiry & Termination](./02-access-control/session-tracking-expiry-termination.md) | Vertical/horizontal bypass, logging, expiry, termination |
| **Abuse & Logic** | API4:2023 | [Unrestricted Resource Consumption](./03-abuse-logic/unrestricted-resource-consumption.md) | Rate limiting, DoS, quota bypass |
| **Abuse & Logic** | API6:2023 | [Business Flows](./03-abuse-logic/unrestricted-business-flow.md) | Workflow abuse, business logic flaws |
| **Abuse & Logic** | API7:2023 | [SSRF](./03-abuse-logic/ssrf.md) | Server-side request forgery |
| **Posture & Consumption** | API8:2023 | [Security Misconfiguration](./04-posture-consumption/security-misconfiguration.md) | Misconfigured headers, verbose errors, debug mode |
| **Posture & Consumption** | API9:2023 | [Improper Inventory Management](./04-posture-consumption/improper-inventory-management.md) | Shadow APIs, unmaintained endpoints |
| **Posture & Consumption** | API10:2023 | [Unsafe Consumption](./04-posture-consumption/unsafe-api-consumption.md) | Insecure third-party integrations, deserialization |

---

## How to Use This Map

1. **Pick a vulnerability type** you're hunting (e.g., broken authentication).
2. **Find the matching stage** in the table above.
3. **Work through the linked labs in order** — each builds on the last.
4. **After each lab**, write a short note in the module's README.md: what the vulnerability was, how you exploited it, and how you'd fix it.

---

## Suggested Learning Path

```text
Authentication Attacks
  → Session Creation
  → User Enumeration
  → HTTP Authentication
  → Password Reset Logic
  → Token Attacks
  → Authentication Attacks (full)

Access Control
  → BOLA / IDOR
  → BOPLA
  → BFLA
  → Session Management
  → Session Tracking, Expiry & Termination

Abuse & Logic
  → Unrestricted Resource Consumption
  → Business Flows
  → SSRF

Posture & Consumption
  → Security Misconfiguration
  → Improper Inventory Management
  → Unsafe Consumption
```

---

## Lab Status Tracker

| Lab | Module | Status | Notes |
|-----|:------:|:------:|:------|
| Session Management | Authentication | ⬜ | |
| User Enumeration | Authentication | ⬜ | |
| HTTP Authentication | Authentication | ⬜ | |
| Password Reset Logic | Authentication | ⬜ | |
| Token Attacks | Authentication | ⬜ | |
| Authentication Attacks | Authentication | ⬜ | |
| BOLA / IDOR | Access Control | ⬜ | |
| BOPLA | Access Control | ⬜ | |
| BFLA | Access Control | ⬜ | |
| Session Management | Access Control | ⬜ | |
| Session Tracking, Expiry & Termination | Access Control | ⬜ | |
| Unrestricted Resource Consumption | Abuse & Logic | ⬜ | |
| Business Flows | Abuse & Logic | ⬜ | |
| SSRF | Abuse & Logic | ⬜ | |
| Security Misconfiguration | Posture & Consumption | ⬜ | |
| Improper Inventory Management | Posture & Consumption | ⬜ | |
| Unsafe Consumption | Posture & Consumption | ⬜ | |

---

[⬆ Back to API Hacking Dashboard](README.md)