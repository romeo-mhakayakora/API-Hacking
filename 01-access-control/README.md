# 01 — 🔐 Access Control

> Who can access what: objects, functions, properties.
>
> [⬅ Back to API Hacking Dashboard](../README.md)

## Modules

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Broken Object Level Authorization (BOLA)](./bola.md) | API1:2023 | ⬜ | [📖](./bola.md) |
| [Broken Authentication](./broken-authentication.md) | API2:2023 | ⬜ | [📖](./broken-authentication.md) |
| [Broken Object Property Level Authorization (Mass Assignment)](./bopla.md) | API3:2023 | ⬜ | [📖](./bopla.md) |
| [Broken Function Level Authorization](./bfla.md) | API5:2023 | ⬜ | [📖](./bfla.md) |

## 🎯 Domain Goal

```text
Endpoint
      ↓
Identifier / Role
      ↓
Authorization Check
      ↓
Bypass
      ↓
Unauthorized Access
```

## Readiness Checklist

- [ ] Explain the underlying concepts
- [ ] Identify attack opportunities
- [ ] Execute the relevant techniques
- [ ] Troubleshoot when the obvious approach fails
- [ ] Document commands and evidence
- [ ] Apply the technique in an unfamiliar environment
- [ ] Combine it with other skills

---

[⬅ Back to API Hacking Dashboard](../README.md)
