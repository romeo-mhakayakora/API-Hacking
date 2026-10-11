# 02 — 🔐 Access Control

> Enforcing what an authenticated identity is allowed to do.
>
> [⬅ Back to API Hacking Dashboard](../README.md)

## Modules

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Broken Object Level Authorization (BOLA)](./bola.md) | API1:2023 | ⬜ | [📖](./bola.md) |
| [Broken Object Property Level Authorization (Mass Assignment)](./bopla.md) | API3:2023 | ⬜ | [📖](./bopla.md) |
| [Broken Function Level Authorization](./bfla.md) | API5:2023 | ⬜ | [📖](./bfla.md) |
| [Session Management](./session-management.md) | API2:2023 | 🔄 | [📖](./session-management.md) |

## 🎯 Domain Goal

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