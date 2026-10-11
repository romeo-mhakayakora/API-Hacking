# 🔌 API Hacking

> My structured path to hacking APIs — from BOLA to business-logic abuse.
>
> Four domains → individual vulnerabilities → technical notes → practical labs → API penetration testing.

> [⬅ My Hacking Hub](https://github.com/romeo-mhakayakora/Hacking-Hub) · [🌐 CPTS Notes Site](https://romeo-mhakayakora.github.io/CPTS/)

---

## 🗺️ API Hacking Map

> Every node below is clickable — it takes you directly to its notes.

```mermaid
flowchart TB
    API["🔌 API HACKING<br/>OWASP API Top 10"]
    D1["🔐 AUTHENTICATION<br/>⬜"]
    D2["🔐 ACCESS CONTROL<br/>⬜"]
    D3["♻️ ABUSE & LOGIC<br/>⬜"]
    D4["🧱 POSTURE & CONSUMPTION<br/>⬜"]
    SUP["🧰 SUPPORTING<br/>⬜"]
    PENT["🏁 API PENETRATION TEST"]

    API --> D1 & D2 & D3 & D4
    D1 & D2 & D3 & D4 --> SUP --> PENT

    subgraph 01_AUTHENTICATION ["01 — 🔐 Authentication"]
        BA["⬜ Broken Authentication"]
        AA["🔄 Authentication Attacks"]
        TA["🔄 Token Attacks"]
        PRL["🔄 Password Reset Logic"]
        HTTP["🔄 HTTP Authentication"]
        UE["🔄 User Enumeration"]
    end
    subgraph 02_ACCESS_CONTROL ["02 — 🔐 Access Control"]
        BOLA["⬜ Broken Object Level Authorization"]
        BOPLA["⬜ Broken Object Property Level Authorization"]
        BFLA["⬜ Broken Function Level Authorization"]
    end
    subgraph 03_ABUSE_LOGIC ["03 — ♻️ Abuse & Logic"]
        URC["⬜ Unrestricted Resource Consumption"]
        UBF["⬜ Unrestricted Access to Sensitive Business Flows"]
        S["⬜ Server-Side Request Forgery"]
    end
    subgraph 04_POSTURE_CONSUMPTION ["04 — 🧱 Posture & Consumption"]
        SM["⬜ Security Misconfiguration"]
        IIM["⬜ Improper Inventory Management"]
        UAC["⬜ Unsafe Consumption of APIs"]
    end
    subgraph SUPPORTING ["🧰 — Supporting Skills"]
        MET["⬜ API Hacking Methodology"]
        EPA["⬜ API Endpoint Analysis"]
        TLS["⬜ Tools"]
    end

    D1 --- 01_AUTHENTICATION
    D2 --- 02_ACCESS_CONTROL
    D3 --- 03_ABUSE_LOGIC
    D4 --- 04_POSTURE_CONSUMPTION
    SUP --- SUPPORTING

    click BA href "./01-authentication/broken-authentication.md"
    click AA href "./01-authentication/authentication-attacks.md"
    click TA href "./01-authentication/token-attacks.md"
    click PRL href "./01-authentication/password-reset-logic.md"
    click HTTP href "./01-authentication/http-authentication.md"
    click UE href "./01-authentication/user-enumeration.md"
    click BOLA href "./02-access-control/bola.md"
    click BOPLA href "./02-access-control/bopla.md"
    click BFLA href "./02-access-control/bfla.md"
    click URC href "./03-abuse-logic/unrestricted-resource-consumption.md"
    click UBF href "./03-abuse-logic/unrestricted-business-flow.md"
    click S href "./03-abuse-logic/ssrf.md"
    click SM href "./04-posture-consumption/security-misconfiguration.md"
    click IIM href "./04-posture-consumption/improper-inventory-management.md"
    click UAC href "./04-posture-consumption/unsafe-api-consumption.md"
    click MET href "./supporting/01-api-methodology.md"
    click EPA href "./supporting/02-api-endpoint-analysis.md"
    click TLS href "./supporting/03-tools.md"
```

### Legend

| Status | Meaning |
|--------|---------|
| ✅ | Vulnerability mastered + notes written |
| 🔄 | Currently practicing |
| ⬜ | Not started |
| 🔁 | Needs review / practical reinforcement |

---

## 📊 Overall Progress

| Domain | OWASP Coverage | Status | Entry Point |
|--------|:--------------:|:------:|-------------|
| 🔐 Authentication | API2 | ⬜ | [Open →](./01-authentication/) |
| 🔐 Access Control | API1–API5 | ⬜ | [Open →](./02-access-control/) |
| ♻️ Abuse & Logic | API4–API7 | ⬜ | [Open →](./03-abuse-logic/) |
| 🧱 Posture & Consumption | API8–API10 | ⬜ | [Open →](./04-posture-consumption/) |
| 🧰 Supporting Skills | — | ⬜ | [Open →](./supporting/) |

---

---

## 01 — 🔐 Authentication

> Proving identity: credentials, tokens, and reset flows. → [Domain README](./01-authentication/README.md)

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Broken Authentication](./01-authentication/broken-authentication.md) | API2:2023 | ⬜ | [📖](./01-authentication/broken-authentication.md) |
| [Authentication Attacks](./01-authentication/authentication-attacks.md) | API2:2023 | 🔄 | [📖](./01-authentication/authentication-attacks.md) |
| [Token Attacks](./01-authentication/token-attacks.md) | API2:2023 | 🔄 | [📖](./01-authentication/token-attacks.md) |
| [Password Reset Logic](./01-authentication/password-reset-logic.md) | API2:2023 | 🔄 | [📖](./01-authentication/password-reset-logic.md) |
| [HTTP Authentication](./01-authentication/http-authentication.md) | API2:2023 | 🔄 | [📖](./01-authentication/http-authentication.md) |
| [User Enumeration](./01-authentication/user-enumeration.md) | API2:2023 | 🔄 | [📖](./01-authentication/user-enumeration.md) |

### 🎯 Domain Goal

```text
Credentials
      ↓
Token Issued
      ↓
Identity Proven
      ↓
Session Established
```

---

## 02 — 🔐 Access Control

> Enforcing what an authenticated identity is allowed to do. → [Domain README](./02-access-control/README.md)

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Broken Object Level Authorization (BOLA)](./02-access-control/bola.md) | API1:2023 | ⬜ | [📖](./02-access-control/bola.md) |
| [Broken Object Property Level Authorization (Mass Assignment)](./02-access-control/bopla.md) | API3:2023 | ⬜ | [📖](./02-access-control/bopla.md) |
| [Broken Function Level Authorization](./02-access-control/bfla.md) | API5:2023 | ⬜ | [📖](./02-access-control/bfla.md) |

### 🎯 Domain Goal

```text
Authenticated Request
      ↓
Object / Function
      ↓
Authorization Check
      ↓
Bypass
      ↓
Unauthorized Access
```

---

## 03 — ♻️ Abuse & Logic

> Abusing how the API behaves: resources, flows, server-side requests. → [Domain README](./03-abuse-logic/README.md)

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Unrestricted Resource Consumption](./03-abuse-logic/unrestricted-resource-consumption.md) | API4:2023 | ⬜ | [📖](./03-abuse-logic/unrestricted-resource-consumption.md) |
| [Unrestricted Access to Sensitive Business Flows](./03-abuse-logic/unrestricted-business-flow.md) | API6:2023 | ⬜ | [📖](./03-abuse-logic/unrestricted-business-flow.md) |
| [Server-Side Request Forgery (SSRF)](./03-abuse-logic/ssrf.md) | API7:2023 | ⬜ | [📖](./03-abuse-logic/ssrf.md) |

### 🎯 Domain Goal

```text
Legitimate Feature
      ↓
Missing Limits
      ↓
Automation / Abuse
      ↓
DoS / Fraud / SSRF
```

---

## 04 — 🧱 Posture & Consumption

> Configuration, inventory, and trusting third parties. → [Domain README](./04-posture-consumption/README.md)

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Security Misconfiguration](./04-posture-consumption/security-misconfiguration.md) | API8:2023 | ⬜ | [📖](./04-posture-consumption/security-misconfiguration.md) |
| [Improper Inventory Management (Shadow APIs)](./04-posture-consumption/improper-inventory-management.md) | API9:2023 | ⬜ | [📖](./04-posture-consumption/improper-inventory-management.md) |
| [Unsafe Consumption of APIs](./04-posture-consumption/unsafe-api-consumption.md) | API10:2023 | ⬜ | [📖](./04-posture-consumption/unsafe-api-consumption.md) |

### 🎯 Domain Goal

```text
Deployment / Integration
      ↓
Misconfig / Blind Spot
      ↓
Exposure
      ↓
Compromise
```

## 🧰 — Supporting Skills

> Methodology and tooling supporting all API attacks. → [Domain README](./supporting/README.md)

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [API Hacking Methodology (Recon to Exploitation)](./supporting/01-api-methodology.md) | — | ⬜ | [📖](./supporting/01-api-methodology.md) |
| [API Endpoint Analysis](./supporting/02-api-endpoint-analysis.md) | — | ⬜ | [📖](./supporting/02-api-endpoint-analysis.md) |
| [Tools (Burp, Postman, ffuf, JWT Tooling)](./supporting/03-tools.md) | — | ⬜ | [📖](./supporting/03-tools.md) |

---

## 🧭 Suggested Order

```text
API Discovery & Mapping
      ↓
Broken Authentication
      ↓
BOLA
      ↓
BFLA / BOPLA
      ↓
SSRF
      ↓
Rate Limits & Business Flow
      ↓
Misconfig & Shadow APIs
```

## ✅ Readiness

A vulnerability counts as mastered when I can explain it, exploit it against an unfamiliar target, troubleshoot failures, document evidence, and chain it with other techniques.

---

[⬆ Back to top](#-api-hacking)
