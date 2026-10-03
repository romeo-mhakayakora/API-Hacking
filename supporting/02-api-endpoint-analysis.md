# API Hacking Notes — Phase 1: Reconnaissance & Mapping

> Ethical hacking only: these notes assume you're working against a lab (crAPI, VAmPI, DVGA, Juice Shop) or a target you have **written authorization** to test.

---

## Objective

When an API has no documentation, it's a black box. The goal is to force the
application to reveal its own architecture by capturing live traffic, then
convert that traffic into a structured, testable API spec (OpenAPI/Swagger)
you can systematically attack.

**Mindset:** Assume "security by obscurity" — no published docs means the
devs are hoping attackers won't find the hidden endpoints. Your job is to
capture every call between client and server until nothing is hidden.

---

## Strategy / Process Flow

### 1. Establish the vantage point (proxy + TLS certificates)
Put a proxy between the client (browser, mobile app, emulator) and the
target server so you can see — and decrypt — all HTTPS traffic.

- **Why certs matter:** HTTPS is encrypted end-to-end. A proxy sitting in
  the middle has to present *its own* TLS certificate to the client so it
  can decrypt, read, and re-encrypt traffic on the fly. If the client
  doesn't trust that cert, it'll flag the connection as tampered and block
  it (or throw SSL errors) — so this step isn't optional, it's what makes
  interception possible at all.
- **Install the proxy's root CA certificate** on the client device:
  - **Browser:** install mitmproxy/Burp's CA cert into the browser's (or
    OS's) trusted root store.
  - **Mobile (iOS/Android) / emulator:** install the CA cert via device
    settings and explicitly mark it as trusted for HTTPS inspection — on
    newer Android versions this may require pushing the cert into the
    system trust store (root) rather than just "user" trust, or apps using
    certificate pinning won't be interceptable at all.
- Route traffic to the proxy via an extension (FoxyProxy for browsers) or
  device-level proxy settings, instead of changing system-wide settings —
  keeps unrelated traffic out of your capture.
- If you hit certificate errors mid-capture (e.g. against a lab like
  `crapi.apisec.ai`), it usually means the CA cert wasn't installed/trusted
  correctly — redo the cert install before re-capturing.

### 2. Emulate the perfect (thorough) user
Methodically click through **every** feature of the app: register, log in,
upload files, change settings, post content, trigger errors, use every menu
option. Incompleteness here = blind spots later — if you skip a feature,
you'll never discover its endpoints.

### 3. Capture and export traffic
Record the session and save it to a file (e.g. a `flows` file in mitmweb).

### 4. Automate the translation
Feed the raw capture into a conversion tool to turn chaotic traffic logs
into a standardized API contract (OpenAPI/YAML) rather than working from
raw logs by hand.

### 5. Refine and weaponize
Clean up the generated spec — unignore endpoints the tool skipped, fix path
variables, add titles — then import into your testing platform (Postman)
with environment variables and auth configured. Result: a clickable map of
the full attack surface, ready for testing.

---

## Two Ways to Reverse-Engineer an API

### Method A — Manual (Postman Proxy only)
Keeps everything inside Postman; more manual cleanup, but single-tool.

1. In Postman: **Capture Requests** → enable proxy (e.g. port 5555).
2. Route browser traffic to that port via FoxyProxy.
3. Add target URL filter so only relevant requests are captured.
4. Explore the app thoroughly (see Step 2 above).
5. Stop the proxy, select captured requests, **Add to Collection**.
6. Rename/organize requests into folders by endpoint group.

**Pros:** no extra tools. **Cons:** passive capture only, more manual tidy-up.

### Method B — Automated (mitmproxy2swagger)
Produces a real OpenAPI spec; preferred by security pros since the chain
(mitmproxy/Burp) allows active manipulation, not just passive listening.

1. Start the proxy: `mitmweb` (listener on :8080, dashboard on :8081).
2. Route client traffic through :8080 (FoxyProxy or device-level proxy).
3. Explore the app thoroughly.
4. In the mitmweb dashboard, **File → Save** → produces a `flows` file.
5. Convert to spec:
   ```
   sudo mitmproxy2swagger -i flows -o spec.yml -p http://target.com -f flow
   ```
6. Open `spec.yml` in a text editor — the tool marks uncertain endpoints
   with `ignore:`. Remove `ignore:` from the ones you want included (keep
   indentation/dashes intact or the file breaks).
7. Re-run with examples to fix formatting and inject real captured payloads:
   ```
   sudo mitmproxy2swagger -i flows -o spec.yml -p http://target.com -f flow --examples
   ```
8. Validate the spec at https://editor.swagger.io (File → Import file).
9. Import `spec.yml` into Postman as a Collection.

### Method C — All-in-One (Postman only, full lifecycle)
Postman can run capture → generate spec → test entirely inside one
workspace, no external conversion tool needed.

1. **Capture:** Use Postman's built-in proxy (same as Method A) to
   intercept browser/mobile traffic directly into a Collection.
2. **Generate:** Postman can auto-generate an OpenAPI spec straight from
   that populated Collection — no `mitmproxy2swagger` step required.
3. **Refine & Test:** Edit the generated spec in Postman's **Spec Hub** and
   immediately run live tests against the endpoints, all in one app.

**Trade-off vs. the mitmproxy chain:** Postman's proxy is *passive* —
listen and build a collection. mitmproxy/Burp let you *actively* intercept,
pause, and rewrite requests mid-flight before they reach the server. That's
why security pros tend to favor the proxy-chain (Method B) even with more
moving parts — Method C is more of a developer/QA convenience path, less
suited to aggressive manipulation during recon.

| | Method C: Postman (All-in-One) | Method B: mitmproxy chain |
|---|---|---|
| Convenience | High — one workspace | Low — files move between tools |
| Traffic modification | Passive capture only | Active — pause/rewrite/drop requests |
| Target audience | Developers & QA testers | Security auditors & pentesters |
| Tooling | Postman proxy → Spec Hub | mitmproxy → mitmproxy2swagger → Swagger Editor |

---

## Post-Mapping Setup (before testing)

1. **Read the docs/spec carefully** — note the overview (auth method, rate
   limits), functionality (HTTP method + endpoint pairs), and request
   requirements (headers, path vars, body params). Common conventions:
   - `:id` or `{id}` → path variable
   - `[param]` → optional input
   - `a || b || c` → allowed values
2. **Set collection variables** — e.g. confirm `baseUrl` matches your target
   (localhost vs. hosted lab instance).
3. **Authenticate and set a token globally:**
   - Register/log in via the API (e.g. `POST /auth/login`).
   - Copy the returned Bearer token.
   - Paste it into the Collection's Authorization tab (not per-request) so
     every request in the collection inherits it.
4. You now have an authenticated, structured, testable map of the target.

---

## Toolset Reference

| Tool | Role | How it helps |
|---|---|---|
| **mitmproxy / mitmweb** | Interception engine | Decrypts & logs HTTPS traffic between client and server; saves full session to a `flows` file. Dashboard at `127.0.0.1:8081`. |
| **FoxyProxy** | Traffic router | Browser extension to route traffic to a specific proxy (Postman/mitmproxy/Burp) with one click, without touching system proxy settings. Keeps capture clean of unrelated traffic. |
| **mitmproxy2swagger** | Spec generator | CLI tool that converts raw captured traffic into an OpenAPI 3.0 YAML spec — infers paths, methods, JSON schema, and (with `--examples`) real payload examples. |
| **Text editor (Sublime/VS Code)** | Spec cleanup | Multi-cursor / batch find-replace to strip `ignore:` tags across many endpoints at once. |
| **Swagger Editor** (editor.swagger.io) | Spec validator/visualizer | Renders the YAML as an interactive, browsable doc — confirms the spec is well-formed before import. |
| **Postman** | Test execution hub | Imports the spec as a Collection; centralizes `baseUrl` and auth token management; lets you fire/chain authenticated requests and organize by endpoint. Also has a built-in proxy for the manual capture method. |
| **Burp Suite (Repeater)** | Manual request manipulation | Intercepts individual requests for editing/replay; critical in later testing phases for bypassing front-end filtering and inspecting raw server responses. |

---

## Quick Command Reference

```bash
# Start traffic interception
mitmweb

# Convert captured traffic to OpenAPI spec (first pass)
sudo mitmproxy2swagger -i /path/to/flows -o spec.yml -p http://target.com -f flow

# Second pass with real payload examples
sudo mitmproxy2swagger -i /path/to/flows -o spec.yml -p http://target.com -f flow --examples
```

---

*Next up: Excessive Data Exposure (OWASP API3) — covered separately.*