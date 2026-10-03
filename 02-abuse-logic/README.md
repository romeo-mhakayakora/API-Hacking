# 02 — ♻️ Abuse & Logic

> Abusing how the API behaves: resources, flows, server-side requests.
>
> [⬅ Back to API Hacking Dashboard](../README.md)

## Modules

| Module | OWASP | Status | Notes |
|--------|:-----:|:------:|:-----:|
| [Unrestricted Resource Consumption](./unrestricted-resource-consumption.md) | API4:2023 | ⬜ | [📖](./unrestricted-resource-consumption.md) |
| [Unrestricted Access to Sensitive Business Flows](./unrestricted-business-flow.md) | API6:2023 | ⬜ | [📖](./unrestricted-business-flow.md) |
| [Server-Side Request Forgery (SSRF)](./ssrf.md) | API7:2023 | ⬜ | [📖](./ssrf.md) |

## 🎯 Domain Goal

```text
Legitimate Feature
      ↓
Missing Limits
      ↓
Automation / Abuse
      ↓
DoS / Fraud / SSRF
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
